---
title: "Blog 3"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 3.3. </b> "
---

AWS vừa giới thiệu **Amazon S3 Annotations**, một khả năng **metadata** (siêu dữ liệu) mới của **Amazon S3** cho phép đính kèm thông tin nghiệp vụ (**business context**) trực tiếp vào mỗi **object**.
Khác với **metadata** truyền thống, **annotations** có thể lưu lượng dữ liệu lớn, chỉnh sửa độc lập và truy vấn trên quy mô hàng tỷ **object** mà không cần xây dựng một hệ thống **metadata** riêng.
Hãy cùng mình mổ xẻ xem tính năng này có gì thú vị và nó khác biệt thế nào so với những cách làm truyền thống nhé!

## S3 Annotations là gì và thông số như thế nào?

Nói một cách đơn giản, **S3 Annotations** cho phép bạn "nhét" hẳn một lượng ngữ cảnh khổng lồ trực tiếp vào bên trong một **Object** mà không làm thay đổi file gốc.
Hãy nhìn vào những con số cấu hình này, đối với dân làm data thì nó thực sự ấn tượng:
*   **1.000** chú thích độc lập trên mỗi **Object**.
*   Mỗi chú thích dung lượng lên tới **1 MB**.
*   Tổng dung lượng chú thích tối đa cho một file lên đến **1 GB**.
*   Hỗ trợ định dạng tự do: `JSON`, `XML`, `YAML`, `plain text`.

Để dễ hình dung, **1 MB** dữ liệu chữ (text) dạng `JSON` là cực kỳ lớn. Bạn có thể lưu cả một file sub hoàn chỉnh, một bản dịch transcript của AI, hoặc toàn bộ lịch sử log của một quy trình xử lý ngay bên cạnh file gốc.

## So sánh: S3 Annotations vs. Metadata Truyền Thống

Để thấy được tại sao S3 Annotations lại là một bước tiến, chúng ta hãy cùng so sánh nó với 2 giải pháp "kinh điển" trước đây của S3: "User-Defined Metadata" và "Object Tagging".

### 1. Trận chiến dung lượng: Phá bỏ giới hạn KB
*   **Truyền thống (Metadata/Tags):** Giới hạn cực kỳ nghiêm ngặt. **User-defined metadata** bị giới hạn tổng cộng chỉ khoảng **2 KB** cho toàn bộ key-value. **Object Tags** thì khá khẩm hơn một chút nhưng cũng giới hạn tối đa **10 tags** và kích thước rất nhỏ. Bạn gần như chỉ lưu được trạng thái (như `status: processed`, `environment: production`).
*   **S3 Annotations:** Nâng giới hạn lên **1 GB/object**. Bạn không còn phải "gói ghém" từng ký tự nữa. Dữ liệu ngữ cảnh giờ đây có thể là một tài liệu cấu trúc phức tạp.

### 2. Kiến trúc hệ thống: Tạm biệt database trung gian
*   **Cách làm cũ:** Vì **metadata** của **S3** quá nhỏ, giới công nghệ thường phải dùng một kiến trúc lai (**Hybrid**). File gốc lưu trên **S3**, còn toàn bộ **metadata** phức tạp (như thông tin bản quyền, dữ liệu phân tích AI) sẽ được lưu ở một database bên ngoài (`DynamoDB`, `PostgreSQL`). Việc này dẫn đến bài toán cực kỳ đau đầu: Đồng bộ dữ liệu. Nếu file trên **S3** bị xóa hoặc replicate sang vùng khác, làm sao để database bên ngoài biết mà cập nhật theo?
*   **S3 Annotations:** Ngữ cảnh đi liền với thực thể. Chú thích nằm ngay trong **S3**. Khi bạn sao chép file sang Region khác, sao lưu (backup) hay xóa file, phần **Annotations** này tự động đi theo hoặc biến mất cùng file. Bạn bớt được một hệ thống database phải quản lý và bảo trì.

### 3. Khả năng cập nhật (Mutability) mà không sợ tốn tiền
*   **Với User-Defined Metadata cũ:** Metadata gắn liền với **Object Header**. Bạn muốn đổi một dòng **metadata**? **S3** buộc bạn phải thực hiện lệnh `CopyObject` để ghi đè (**override**) lại toàn bộ file. Với file vài GB hay vài TB, việc ghi đè này chỉ để sửa 1 dòng chữ là cực kỳ tốn kém tiền bạc và băng thông.
*   **S3 Annotations:** Thiết kế theo dạng module độc lập. Bạn gọi API `PutObjectAnnotation` để sửa một chú thích cụ thể, **S3** sẽ cập nhật ngay lập tức mà không hề chạm vào hay ghi đè lên file gốc.

### 4. Khả năng truy vấn (Queryability) chuyên sâu
*   **Cách làm cũ:** **Object Tags** có thể dùng để phân quyền (**ABAC**) hoặc thiết lập vòng đời file (**Lifecycle**), nhưng không thể dùng để chạy các câu lệnh truy vấn phức tạp.
*   **S3 Annotations:** Khi bạn bật tính năng **S3 Metadata**, các file **Annotations** dạng `JSON` sẽ tự động được phân tách (**flatten**) thành các bảng dữ liệu có cấu trúc. Bạn có thể dùng **Amazon Athena** để gõ **SQL**, tìm kiếm xem "Những file video nào có chứa từ khóa X trong phần chú thích transcript" một cách trực tiếp.

## Góc nhìn tương lai: Tại sao AWS lại làm điều này?

Theo mình, sự ra đời của **S3 Annotations** không đơn thuần là nâng cấp bộ nhớ lưu trữ **metadata**. Động thái này hướng thẳng tới kỷ nguyên của **AI Agents** và **Autonomous Workflows** (Quy trình tự động hóa).

Trong kỷ nguyên **GenAI**, các mô hình AI cần đọc hiểu dữ liệu rất nhanh. Thay vì bắt AI phải quét qua một file video nặng vài GB để hiểu nội dung, hệ thống có thể đọc phần **Annotation** (chứa tóm tắt nội dung, nhãn dán, timeline do AI phân tích trước đó) chỉ trong vài mili-giây. Dữ liệu tự tiến hóa, tự sinh thêm ngữ cảnh theo thời gian thông qua các vòng lặp của AI mà không làm phình to hay xáo trộn file gốc.

## Lời kết

**Amazon S3 Annotations** thực sự là một tính năng "nhỏ nhưng có võ". Nó giải quyết được bài toán lưu trữ ngữ cảnh lớn ngay tại chỗ (**in-place context**), giảm tải sự phụ thuộc vào các database phụ trợ và tối ưu chi phí vận hành đáng kể.

Nếu dự án sắp tới của các bạn liên quan đến việc xử lý lượng lớn file đa phương tiện (Media), dữ liệu y tế phức tạp, hoặc làm **Data Lake** cho AI, mình nghĩ đây là một tính năng rất đáng để thử nghiệm và đưa vào kiến trúc hệ thống!

---

*Bài viết gốc:* [Amazon S3 Annotations: Attach rich, queryable context directly to your objects](https://aws.amazon.com/blogs/aws/amazon-s3-annotations-attach-rich-queryable-context-directly-to-your-objects/)