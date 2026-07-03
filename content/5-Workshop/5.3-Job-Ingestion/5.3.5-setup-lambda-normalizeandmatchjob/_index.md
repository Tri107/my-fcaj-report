---
title : "Build Normalize Lambda"
date : 2026-07-02 
weight : 5
chapter : false
pre : " <b> 5.3.5 </b> "
---

#### 1. Initialize Lambda

Create the Lambda Function using Python.

The Lambda receives JSON data from the Process Job stage.

Configure template.yaml:

```yaml
NormalizeAndMatchFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src
      Handler: functions/normalizeAndMatch/handler.lambda_handler
      Runtime: python3.13
      Description: Normalize job and perform deduplication hashing and fuzzy matching
      Timeout: 30
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref JobsTable
      Environment:
        Variables:
          JOBS_TABLE: !Ref JobsTable
```

---

#### 2. Job Handler Function

Perform the following steps:

* Check for required fields.
* Normalize date format.
* Standardize company name.
* Normalize location.
* Generate identifier key (ID) if necessary.
* Remove invalid data.

> **Code Placeholders**

* `src/functions/normalizeAndMatch/handler.py`:
```python
import json
import hashlib
import uuid
from datetime import datetime, timezone

from repositories.jobRepository import JobRepository
from services.matchAndNormalizeService import (
    normalize_title,
    normalize_posted_at,
    check_exact_duplicate,
    check_fuzzy_duplicate
)

# Initialize repository globally to reuse DynamoDB client connection
try:
    repository = JobRepository()
except Exception as e:
    print(f"Warning: Repository initialization failed (likely local environment without environment variables): {e}")
    repository = None

def lambda_handler(event, context):
    print("Received raw event:", json.dumps(event))
    
    global repository
    if not repository:
        # Fallback reload just in case
        repository = JobRepository()

    # 1. Extract required information
    original_title = event.get("title") or event.get("job_title", "")
    company_name = event.get("company_name", "")
    location = event.get("location", "")
    
    if not original_title or not company_name:
        print("Warning: Missing required fields (title, company_name) in event payload. Rejecting.")
        return {
            "status": "rejected",
            "reason": "missing_required_fields",
            "received_fields": list(event.keys())
        }
    
    detected_extensions = event.get("detected_extensions", {})
    original_posted_at = detected_extensions.get("posted_at")
    original_schedule_type = detected_extensions.get("schedule_type")
    
    # Fallback to extensions list if detected_extensions is empty
    extensions = event.get("extensions", [])
    if not original_posted_at and len(extensions) > 0:
        original_posted_at = extensions[0]
    if not original_schedule_type and len(extensions) > 1:
        original_schedule_type = extensions[1]
        
    source_link = event.get("source_link", "")
    description = event.get("description", "")
    thumbnail = event.get("thumbnail") or ""
    
    # 2. Normalization
    normalized_title = normalize_title(original_title)
    normalized_posted_at = normalize_posted_at(original_posted_at)
    
    # 3. Deduplication Pipeline
    # Step 1: Hashing
    combined_string = f"{normalized_title}+{company_name}+{location}"
    hash_object = hashlib.sha256(combined_string.encode("utf-8"))
    sha256_hash = hash_object.hexdigest()
    
    # Check exact match
    print(f"Checking hash uniqueness: {sha256_hash}")
    if check_exact_duplicate(sha256_hash, repository):
        print("Duplicate detected (exact hash match). Rejecting.")
        return {
            "status": "duplicate",
            "reason": "exact_hash_match",
            "hash": sha256_hash
        }
        
    # Step 2: Fuzzy Matching
    print(f"Scanning table for candidates with company: {company_name}")
    fuzzy_result = check_fuzzy_duplicate(normalized_title, company_name, location, repository)
    if fuzzy_result.get("is_duplicate"):
        similarity = fuzzy_result.get("similarity")
        matched_job_id = fuzzy_result.get("matched_job_id")
        print(f"Duplicate detected (fuzzy match similarity {similarity:.2f} > 0.8). Rejecting.")
        return {
            "status": "duplicate",
            "reason": "fuzzy_match",
            "similarity": similarity,
            "matched_job_id": matched_job_id
        }
            
    # 4. Insert into DynamoDB
    job_id = str(uuid.uuid4())
    new_job = {
        "jobId": job_id,
        "hash": sha256_hash,
        "title": normalized_title,
        "originalTitle": original_title,
        "companyName": company_name,
        "location": location,
        "postedAt": normalized_posted_at,
        "originalPostedAt": original_posted_at,
        "scheduleType": original_schedule_type or "",
        "sourceLink": source_link,
        "description": description,
        "thumbnail": thumbnail,
        "createdAt": datetime.now(timezone.utc).isoformat().replace("+00:00", "Z")
    }
    
    print(f"Inserting new job: {job_id}")
    repository.insert(new_job)
    
    return {
        "status": "inserted",
        "jobId": job_id,
        "hash": sha256_hash
    }

```

* `src/services/matchAndNormalizeService.py`:
```python
import re
import difflib
from datetime import datetime, timezone, timedelta
from repositories.jobRepository import JobRepository

def normalize_title(title: str) -> str:
    if not title:
        return ""
    # Convert to lowercase
    cleaned = title.lower()
    # Strip bracketed/parenthesized content
    cleaned = re.sub(r"\[.*?\]|\(.*?\)", "", cleaned)
    # Clean separators and noise characters
    cleaned = re.sub(r"[-/|:,_]", " ", cleaned)
    # Clean level adjectives / noise words
    noise_words = [
        r"\bsenior\b", r"\bjunior\b", r"\bfresher\b", r"\binternship\b", r"\bintern\b",
        r"\blead\b", r"\bprincipal\b", r"\bstaff\b", r"\bmid\b", r"\bentry\b", r"\blevel\b",
        r"\bhcm\b", r"\bhn\b", r"\bhanoi\b", r"\bdanang\b", r"\bvietnam\b", r"\bvn\b", r"\bcity\b",
        r"\btuyển dụng\b", r"\bviệc làm\b", r"\btuyển\b", r"\bgấp\b"
    ]
    for noise in noise_words:
        cleaned = re.sub(noise, "", cleaned)
    
    # Standardize developer/engineer synonyms
    cleaned = re.sub(r"\bback-end\b", "backend", cleaned)
    cleaned = re.sub(r"\bfront-end\b", "frontend", cleaned)
    cleaned = re.sub(r"\bsoftware developer\b|\bsoftware engineer\b|\bdeveloper\b|\bengineer\b|\bdev\b", "developer", cleaned)
    
    # Strip extra whitespaces
    cleaned = re.sub(r"\s+", " ", cleaned).strip()
    return cleaned

def normalize_posted_at(posted_at_str: str) -> str:
    now = datetime.now(timezone.utc)
    if not posted_at_str:
        return now.strftime("%Y-%m-%d")
    
    cleaned = posted_at_str.lower().strip()
    
    # Check for today, yesterday, hours/minutes/seconds ago
    if any(word in cleaned for word in ["hôm nay", "vừa xong", "mới", "hour", "giờ", "minute", "phút", "giây", "second"]):
        return now.strftime("%Y-%m-%d")
    if "hôm qua" in cleaned or "yesterday" in cleaned:
        return (now - timedelta(days=1)).strftime("%Y-%m-%d")
        
    # Extract number
    match = re.search(r"(\d+)", cleaned)
    if not match:
        return now.strftime("%Y-%m-%d")
    
    num = int(match.group(1))
    
    # Check units
    if "ngày" in cleaned or "day" in cleaned:
        days = num
    elif "tuần" in cleaned or "week" in cleaned:
        days = num * 7
    elif "tháng" in cleaned or "month" in cleaned:
        days = num * 30
    else:
        days = num
        
    est_date = now - timedelta(days=days)
    return est_date.strftime("%Y-%m-%d")

def check_exact_duplicate(sha256_hash: str, repository: JobRepository) -> bool:
    items = repository.find_by_hash(sha256_hash)
    return len(items) > 0

def check_fuzzy_duplicate(normalized_title: str, company_name: str, location: str, repository: JobRepository) -> dict:
    candidates = repository.find_by_company(company_name)
    
    new_match_str = f"{normalized_title} {company_name} {location}".lower()
    
    for candidate in candidates:
        candidate_title = candidate.get("title", "")
        candidate_company = candidate.get("companyName", "")
        candidate_location = candidate.get("location", "")
        
        cand_match_str = f"{candidate_title} {candidate_company} {candidate_location}".lower()
        
        similarity = difflib.SequenceMatcher(None, new_match_str, cand_match_str).ratio()
        
        if similarity > 0.8:
            return {
                "is_duplicate": True,
                "similarity": similarity,
                "matched_job_id": candidate.get("jobId")
            }
            
    return {
        "is_duplicate": False
    }

```

---

#### 3. Save to DynamoDB

After the data is normalized, write the record to the Jobs table.

> **Code Placeholder**

* `src/repositories/jobRepository.py`:
```python
import os
import boto3
from boto3.dynamodb.conditions import Key, Attr

class JobRepository:
    def __init__(self, table_name=None):
        self.table_name = table_name or os.environ.get("JOBS_TABLE")
        if not self.table_name:
            raise ValueError("JOBS_TABLE environment variable not configured")
        
        self.dynamodb = boto3.resource("dynamodb")
        self.table = self.dynamodb.Table(self.table_name)

    def find_by_hash(self, sha256_hash: str) -> list:
        response = self.table.query(
            IndexName="HashIndex",
            KeyConditionExpression=Key("hash").eq(sha256_hash)
        )
        return response.get("Items", [])

    def find_by_company(self, company_name: str) -> list:
        response = self.table.scan(
            FilterExpression=Attr("companyName").eq(company_name)
        )
        return response.get("Items", [])

    def insert(self, job_item: dict) -> None:
        self.table.put_item(Item=job_item)

```

---

#### 4. Deploy

Run the following commands in sequence:

```bash
sam build
sam validate
sam deploy
```

Deploy successfully:

![Deploy normalize and match job](/images/5-Workshop/5.3-Job-ingestion/5.3.5-setup-lambda-normalizeandmatchjob/deploy-normalizeandmatchjob.png)

---

#### 5. Testing

Re-execute the verification steps for the fetch jobs function; the process jobs and normalize-and-match functions are triggered automatically when data is pushed into the SQS queue.

![DynamoDB Items](/images/5-Workshop/5.3-Job-ingestion/5.3.5-setup-lambda-normalizeandmatchjob/dynamodb-items.png)
