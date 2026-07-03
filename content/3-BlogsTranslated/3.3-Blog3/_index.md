---
title: "Blog 3"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 3.3. </b> "
---

AWS has recently introduced **Amazon S3 Annotations**, a new **metadata** capability for **Amazon S3** that allows attaching business context directly to each **object**.
Unlike traditional **metadata**, **annotations** can store large amounts of data, be modified independently, and be queried at the scale of billions of **objects** without needing to build a separate **metadata** system.
Let's analyze what makes this feature interesting and how it differs from traditional approaches!

## What are S3 Annotations and what are the specifications?

Simply put, **S3 Annotations** allow you to "stuff" a massive amount of context directly into an **Object** without modifying the original file.
Look at these configuration numbers, for data professionals, it's truly impressive:
*   **1,000** independent annotations per **Object**.
*   Each annotation can be up to **1 MB** in size.
*   Total maximum annotation size for a file is up to **1 GB**.
*   Supports flexible formats: `JSON`, `XML`, `YAML`, `plain text`.

To visualize, **1 MB** of text data in `JSON` format is incredibly large. You can store a complete subtitle file, an AI transcript translation, or the entire processing log history right alongside the original file.

## Comparison: S3 Annotations vs. Traditional Metadata

To see why S3 Annotations is a step forward, let's compare it with 2 "classic" previous S3 solutions: "User-Defined Metadata" and "Object Tagging".

### 1. The Capacity Battle: Breaking the KB limit
*   **Traditional (Metadata/Tags):** Extremely strict limits. **User-defined metadata** is limited to a total of only about **2 KB** for all key-values. **Object Tags** are slightly better but also have a maximum limit of **10 tags** and a very small size. You can mostly only store states (like `status: processed`, `environment: production`).
*   **S3 Annotations:** Raises the limit to **1 GB/object**. You no longer have to carefully "pack" every character. Context data can now be a complex structured document.

### 2. System Architecture: Goodbye intermediate database
*   **Old way:** Because **S3's metadata** is too small, the tech industry often has to use a hybrid architecture. The original file is stored on **S3**, while all complex **metadata** (such as copyright information, AI analysis data) is stored in an external database (`DynamoDB`, `PostgreSQL`). This leads to a very painful problem: Data synchronization. If a file on **S3** is deleted or replicated to another region, how does the external database know to update?
*   **S3 Annotations:** Context goes with the entity. Annotations reside right in **S3**. When you copy a file to another Region, back it up, or delete it, these **Annotations** automatically follow or disappear with the file. You save yourself the trouble of managing and maintaining a database system.

### 3. Mutability without fear of cost
*   **With old User-Defined Metadata:** Metadata is tied to the **Object Header**. Want to change a line of **metadata**? **S3** forces you to perform a `CopyObject` command to overwrite the entire file. For files of several GB or TB, this overwriting just to fix a line of text is extremely costly in terms of money and bandwidth.
*   **S3 Annotations:** Designed as an independent module. You call the `PutObjectAnnotation` API to edit a specific annotation, and **S3** will update it immediately without touching or overwriting the original file.

### 4. Advanced Queryability
*   **Old way:** **Object Tags** can be used for authorization (**ABAC**) or setting up file lifecycle rules (**Lifecycle**), but cannot be used to run complex query commands.
*   **S3 Annotations:** When you enable the **S3 Metadata** feature, **Annotations** files in `JSON` format will automatically be flattened into structured data tables. You can use **Amazon Athena** to write **SQL**, searching directly for "Which video files contain keyword X in the transcript annotation".

## Future Perspective: Why is AWS doing this?

In my opinion, the introduction of **S3 Annotations** is not merely an upgrade to **metadata** storage capacity. This move is heading straight for the era of **AI Agents** and **Autonomous Workflows**.

In the **GenAI** era, AI models need to read and understand data very quickly. Instead of forcing AI to scan a multi-GB video file to understand its content, the system can read the **Annotation** section (containing a content summary, tags, timeline previously analyzed by AI) in just a few milliseconds. Data evolves itself, generating more context over time through AI feedback loops without bloating or altering the original file.

## Conclusion

**Amazon S3 Annotations** is truly a "small but mighty" feature. It solves the problem of storing large in-place context, reduces the dependency on auxiliary databases, and significantly optimizes operational costs.

If your upcoming projects involve processing large amounts of multimedia files, complex medical data, or building a **Data Lake** for AI, I think this is a highly worthwhile feature to test and integrate into your system architecture!

---

*Original article:* [Amazon S3 Annotations: Attach rich, queryable context directly to your objects](https://aws.amazon.com/blogs/aws/amazon-s3-annotations-attach-rich-queryable-context-directly-to-your-objects/)
