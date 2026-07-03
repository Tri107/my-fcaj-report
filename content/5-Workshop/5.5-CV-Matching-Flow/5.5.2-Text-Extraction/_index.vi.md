---
title : "Trích xuất văn bản"
date : 2026-07-03
weight : 2
chapter : false
pre : " <b> 5.5.2. </b> "
---

## Mục tiêu

Text extraction chuyển file CV đã upload thành plain text để bước AI evaluation có thể so sánh với job description. API chỉ nhận `cvKey`; API không nhận trực tiếp file PDF.

`ExtractCvFunction` xử lý endpoint `POST /cv/extract`. Lambda đọc `userId` từ Cognito JWT claim `sub`, kiểm tra `cvKey` có thuộc user đang đăng nhập không, đọc CV từ S3, rồi gọi Amazon Textract.

Region: `ap-southeast-1`  
CV bucket: `jobs-matching-cvs-dev-973370772729`

## Flow

```text
POST /cv/extract
  -> ExtractCvFunction
  -> Kiểm tra user từ Cognito
  -> Kiểm tra cvKey thuộc user hiện tại
  -> Đọc file CV từ S3
  -> Gọi Amazon Textract
  -> Trả về text đã trích xuất
```

## Request body

```json
{
  "cvKey": "997ab59c-20b1-707d-4083-cb651fcdfb6f/cv_1782699166153.pdf"
}
```

`userId` không nằm trong body. Giá trị này được lấy từ `requestContext.authorizer.jwt.claims.sub`.

## Expected response

```json  
{
  "cvKey": "997ab59c-20b1-707d-4083-cb651fcdfb6f/cv_1782699166153.pdf",
  "pageCount": 1,
  "wordCount": 428,
  "text": "DUONG BINH CUONG\nInformation Technology Student\nEmail: example@email.com\nSkills: JavaScript, React, Node.js, SQL, Postman..."
}
```

## Giải thích field

- `cvKey`: key của file CV trong S3.
- `pageCount`: số trang được phát hiện từ CV.
- `wordCount`: số từ ước tính được trích xuất từ CV.
- `text`: nội dung plain text được trích xuất từ PDF. Field này có thể dài, nên trong workshop chỉ nên hiển thị bản rút gọn.

## Lỗi phổ biến

- Gửi trực tiếp file PDF vào `/cv/extract` thay vì gửi `cvKey`.
- Dùng `cvKey` không thuộc user đang đăng nhập.
- Test thiếu Cognito JWT claims.
- Hiển thị toàn bộ CV text trong screenshot hoặc log.
- Nhầm bước Textract extraction với bước AI evaluation.

> Lưu ý chi phí: Amazon Textract có thể phát sinh chi phí theo số trang được xử lý. Bước này cũng có thể phát sinh chi phí từ Lambda, S3 và CloudWatch Logs.
