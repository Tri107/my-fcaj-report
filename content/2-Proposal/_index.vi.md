---
title: "Bản đề xuất"
date: 2026-07-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---
# Jobs Matching Platform
## Giải pháp AWS Serverless tổng hợp việc làm và phân tích hồ sơ tự động

### 1. Tóm tắt điều hành
Jobs Matching Platform được thiết kế bởi nhóm NOVA nhằm tự động hóa toàn diện quy trình thu thập dữ liệu việc làm và đánh giá mức độ phù hợp của hồ sơ ứng viên (CV). Giải pháp này phục vụ trực tiếp nhu cầu tối ưu hóa quá trình tìm kiếm việc làm, đặc biệt hướng tới đối tượng sinh viên và nhân sự khối ngành công nghệ thông tin. Thông qua việc ứng dụng trí tuệ nhân tạo vào đối chiếu năng lực, nền tảng hỗ trợ người dùng rút ngắn thời gian tìm kiếm và định vị chính xác vị trí công việc phù hợp. Hệ thống tận dụng triệt để hệ sinh thái AWS Serverless nhằm đảm bảo tính sẵn sàng cao, khả năng mở rộng linh hoạt và tối ưu hóa chi phí vận hành.

### 2. Tuyên bố vấn đề
*Vấn đề hiện tại*
Quá trình tìm kiếm thông tin tuyển dụng trên nhiều nền tảng phân tán hiện nay đòi hỏi lượng lớn thời gian và thao tác thủ công. Người tìm việc thường gặp trở ngại trong việc tự đánh giá khách quan và chính xác mức độ phù hợp của năng lực cá nhân đối với yêu cầu cụ thể của từng mô tả công việc, đặc biệt là với các bộ kỹ năng chuyên sâu như lập trình hay quản trị hạ tầng đám mây.

*Giải pháp*
Hệ thống sử dụng dịch vụ AWS Lambda để tự động thu thập thông tin công việc qua SerpApi, điều phối và kiểm soát luồng dữ liệu thông qua Amazon SQS, sau đó lưu trữ có cấu trúc tại Amazon DynamoDB. Phân hệ tương tác người dùng được bảo vệ bởi tường lửa AWS WAF và quản lý xác thực qua Amazon Cognito. Luồng phân tích CV tự động trích xuất văn bản từ tài liệu được tải lên Amazon S3, tiếp nối bằng việc tích hợp dịch vụ AI để đối chiếu, phân tích ngữ nghĩa và chấm điểm mức độ phù hợp. Toàn bộ quy trình phát triển và quản lý tác vụ phần mềm được vận hành chặt chẽ thông qua Jira, kết hợp kiểm soát phiên bản mã nguồn trên GitHub.

*Lợi ích và hoàn vốn đầu tư (ROI)*
Giải pháp tiến tới chuẩn hóa toàn trình quy trình tìm kiếm và chắt lọc công việc. Việc vận dụng các kiến thức thực tiễn từ các bài thực hành và tự nghiên cứu về kiến trúc AWS Cloud giúp thiết kế hệ thống đạt hiệu suất cao với chi phí vận hành tối thiểu. Nền tảng hạ tầng serverless đảm bảo chi phí chỉ phát sinh dựa trên lưu lượng tính toán thực tế, mang lại hiệu quả kinh tế tối ưu và khả năng mở rộng hệ thống minh bạch trong tương lai.

### 3. Kiến trúc giải pháp
Nền tảng được xây dựng dựa trên nguyên tắc kiến trúc AWS Serverless, phân chia thành các luồng xử lý độc lập nhưng có tính liên kết chặt chẽ: thu thập dữ liệu, tương tác người dùng, phân tích hồ sơ và vận hành bảo mật.


*Dịch vụ AWS sử dụng*
- *AWS EventBridge Scheduler*: Lên lịch tự động hóa quá trình kích hoạt truy xuất dữ liệu việc làm từ SerpApi.
- *AWS Lambda*: Đảm nhiệm xử lý logic nghiệp vụ cho các tác vụ lấy dữ liệu, tìm kiếm, trích xuất văn bản CV và giao tiếp với API của mô hình AI.
- *Amazon SQS*: Quản lý hàng đợi tin nhắn, đóng vai trò cơ chế đệm nhằm chống quá tải cho quá trình chuẩn hóa khối lượng lớn dữ liệu việc làm.
- *Amazon Cognito*: Quản lý định danh, xác thực và phân quyền truy cập người dùng.
- *AWS Amplify*: Cung cấp môi trường lưu trữ và phân phối ứng dụng giao diện web (Hosting).
- *AWS WAF & Amazon API Gateway*: Thiết lập tuyến phòng thủ bảo vệ ứng dụng khỏi các lỗ hổng bảo mật web và phân phối lưu lượng RESTful API một cách an toàn.
- *Amazon DynamoDB*: Lưu trữ danh sách công việc, danh sách yêu thích cá nhân và kết quả đánh giá CV.
- *Amazon S3*: Cung cấp kho lưu trữ an toàn, độ bền cao và khả năng mở rộng không giới hạn dành cho các tệp hồ sơ ứng viên.
- *AWS IAM, Amazon CloudWatch, AWS Budgets*: Thực thi quản trị phân quyền cốt lõi, giám sát sức khỏe hệ thống và kiểm soát nghiêm ngặt ngân sách vận hành.
- *AWS Systems Manager (SSM) Parameter Store*: Lưu trữ và quản lý an toàn các API keys và cấu hình nhạy cảm.

*Thiết kế thành phần*
- *Jobs Ingestion*: Thu thập dữ liệu theo lịch trình từ nền tảng bên thứ ba, xử lý bất đồng bộ thông qua SQS, tiến hành chuẩn hóa dữ liệu bằng Lambda và lưu trữ vào DynamoDB với cơ chế vòng đời dữ liệu (TTL) được thiết lập.
- *User Interaction*: Giao diện front-end giao tiếp qua API Gateway, cho phép thực thi các truy vấn trên DynamoDB để lọc, tìm kiếm việc làm và lưu trữ vào bảng danh mục yêu thích độc lập.
- *CV Matching Flow*: Kích hoạt tự động dựa trên sự kiện người dùng tải CV lên S3. Hệ thống tiến hành bóc tách nội dung văn bản, chuyển tiếp dữ liệu tới dịch vụ AI để đánh giá, đối chiếu đa chiều với mô tả công việc và ghi nhận điểm số trả về.

### 4. Triển khai kỹ thuật
Dự án được cấu trúc thành 3 phần chính: Thu thập và xử lý dữ liệu; Xây dựng hệ thống tổng hợp việc làm; Đánh giá CV theo công việc. Mỗi phần được triển khai qua 4 giai đoạn cụ thể:
1. *Nghiên cứu và thiết kế*: Phân tích chuyên sâu cấu trúc dữ liệu trả về từ SerpApi và thiết kế quy trình xác thực người dùng đa lớp. Nghiên cứu tài liệu về kiến trúc AWS Serverless để lựa chọn dịch vụ phù hợp.
2. *Tính toán chi phí và cấu hình*: Khởi tạo giới hạn ngân sách tự động trên nền tảng AWS và chuẩn bị môi trường kiểm thử cục bộ thông qua các container Docker.
3. *Phát triển luồng nghiệp vụ và đánh giá khả thi*: Xây dựng các hàm tính toán Lambda, thiết lập định tuyến API Gateway, cấu hình pipeline xử lý hàng đợi SQS và tối ưu hóa cấu trúc dữ liệu NoSQL của DynamoDB. Tinh chỉnh liên tục để đảm bảo cân bằng giữa hiệu suất và chi phí.
4. *Kiểm thử và triển khai*: Thực hiện kiểm thử chịu tải, kiểm định độ chính xác của logic hệ thống, xử lý ngoại lệ và đưa vào môi trường vận hành thực tế.

*Yêu cầu kỹ thuật*
- *Thu thập và xử lý dữ liệu*: Dữ liệu thô tiếp nhận từ SerpApi bắt buộc phải trải qua quá trình làm sạch và chuẩn hóa định dạng trước khi tiến hành thao tác ghi vào DynamoDB.
- *Thiết kế cơ sở dữ liệu*: Yêu cầu thiết kế mô hình dữ liệu DynamoDB với khóa phân vùng (partition key) và khóa sắp xếp (sort key) tối ưu nhằm phục vụ các truy xuất tốc độ cao và giảm thiểu chi phí đọc/ghi.
- *Tích hợp AI*: Dịch vụ ngôn ngữ tích hợp cần đảm bảo độ trễ thấp và khả năng hiểu ngữ cảnh chuyên ngành chính xác để phân tích các hồ sơ kỹ thuật phức tạp một cách khách quan nhất.

### 5. Lộ trình & Mốc triển khai
Lộ trình thực thi được hoạch định trong khuôn khổ 12 tuần của chương trình First Cloud AI Journey tại AWS Vietnam:
- *Tuần 1 - Tuần 6*: Tiếp thu kiến thức nền tảng và hoàn thành các bài thực hành (lab) theo chương trình để củng cố chuyên môn về hệ sinh thái dịch vụ AWS Cloud.
- *Tuần 6 - Tuần 10*: Hình thành ý tưởng, hoàn thiện bản thiết kế kiến trúc tổng thể cho dự án nhóm và tiến hành lập trình, xây dựng hệ thống cốt lõi.
- *Tuần 11 - Tuần 12*: Thực thi quy trình kiểm thử toàn diện, đánh giá mức độ hoàn thiện của hệ thống, tinh chỉnh cuối cùng và triển khai đưa vào sử dụng thực tế.

### 6. Ước tính ngân sách
Hệ thống vận hành hoàn toàn theo mô hình trả phí theo mức sử dụng thực tế (pay-as-you-go).

*Chi phí hạ tầng dự kiến*
- *SerpApi*: Tận dụng hạn mức miễn phí 250 lượt truy vấn/tháng. Với tần suất sử dụng hiện tại là 8 lượt/ngày, tổng mức sử dụng tối đa đạt 248 lượt/tháng, tương đương chi phí 0 USD.
- *AWS Lambda*: Tối ưu chi phí bằng cách vận dụng hạn mức miễn phí (1 triệu yêu cầu và 400.000 GB-giây tính toán mỗi tháng). Chi phí phát sinh duy trì ở mức tiệm cận không đối với các xử lý thông thường. Khi vượt hạn mức, chi phí áp dụng là 0.20 USD cho mỗi 1 triệu yêu cầu.
- *AWS Amplify*: Nằm trong giới hạn miễn phí (Free Tier) với 1.000 phút build/tháng, 5 GB lưu trữ và 15 GB băng thông truyền tải mỗi tháng. Với quy mô dự án hiện tại, chi phí dự kiến là 0 USD. Nếu vượt hạn mức, chi phí là 0.01 USD/phút build, 0.023 USD/GB lưu trữ và 0.15 USD/GB băng thông.
- *Amazon SQS & DynamoDB*: Nằm trong giới hạn miễn phí với 1 triệu yêu cầu SQS/tháng, 25 GB lưu trữ và 25 WCU/RCU cho DynamoDB. Với khối lượng dữ liệu việc làm, chi phí dự kiến là 0 USD. Nếu vượt mức, DynamoDB tính phí 1.25 USD/1 triệu WCU và 0.25 USD/1 triệu RCU.
- *Amazon S3*: Nằm trong giới hạn miễn phí 5 GB lưu trữ, 20.000 yêu cầu GET và 2.000 yêu cầu PUT. Ước tính với khoảng 1.000 CV (~1 GB), chi phí là 0 USD. Nếu vượt mức, chi phí là 0.023 USD/GB.
- *Amazon API Gateway*: Nằm trong giới hạn miễn phí 1 triệu lượt gọi API/tháng. Chi phí dự kiến là 0 USD. Vượt hạn mức tính phí 1.00 USD/1 triệu lượt gọi.
- *AWS WAF*: Không có gói miễn phí vĩnh viễn. Phí duy trì 5.00 USD/tháng cho 1 Web ACL, 1.00 USD/tháng cho mỗi quy tắc (Rule) và 0.60 USD/1 triệu yêu cầu. Chi phí dự kiến tối thiểu khoảng 6.00 - 7.00 USD/tháng (nếu bật WAF).
- *Gemini Flash Lite 3.1*:
  - *Hạn mức miễn phí*: 15 yêu cầu/phút; 250.000 tokens/phút; 500 yêu cầu/ngày.
  - *Hạn mức trả phí*: Văn bản đầu vào ở mức 0.25 USD / 1 triệu tokens; Đầu ra (bao gồm token suy luận) ở mức 1.50 USD / 1 triệu tokens.

*Tổng chi phí và khả năng duy trì*
- **Tổng chi phí dự kiến trong 1 tháng**: Khoảng **0.00 USD** (nếu không bật WAF) hoặc **~8.00 USD** (nếu duy trì WAF bảo mật).
- **Khả năng duy trì**: Với kiến trúc Serverless, dự án hoàn toàn tận dụng được các gói Free Tier của AWS. Điều này mang lại khả năng duy trì hệ thống trong dài hạn với mức phí gần như bằng 0. Khi có lượng người dùng lớn, chi phí mới bắt đầu phát sinh tuyến tính theo lượng truy cập, giúp tối ưu hóa rủi ro tài chính cho sinh viên.

### 7. Đánh giá rủi ro
*Ma trận rủi ro*
- *Phụ thuộc dữ liệu*: Mức độ ảnh hưởng cao, xác suất trung bình (Hệ thống phụ thuộc vào API thu thập dữ liệu việc làm từ bên thứ 3 - SerpApi).
- *Chất lượng đánh giá CV*: Mức độ ảnh hưởng cao, xác suất trung bình (Kết quả hoàn toàn phụ thuộc vào năng lực của mô hình AI được sử dụng).

*Chiến lược giảm thiểu*
- *API & Lưu lượng*: Ứng dụng kiến trúc hướng sự kiện với SQS để điều tiết lưu lượng xử lý. Triển khai thuật toán chờ lũy thừa (exponential backoff) nhằm xử lý hiệu quả các lỗi giới hạn tốc độ (rate limit) khi truy xuất dữ liệu bên ngoài.
- *Chi phí*: Thiết lập AWS Budgets để tự động kích hoạt thông báo cảnh báo đến quản trị viên khi chi phí tiêu thụ thực tế chạm ngưỡng 80% ngân sách cho phép.
- *Đánh giá AI*: Phát triển tính năng minh bạch hóa, cho phép ứng viên tự kiểm tra và xác nhận danh sách các kỹ năng được trích xuất từ CV trước khi hệ thống tiến hành đối chiếu kết quả cuối cùng.

*Kế hoạch dự phòng*
- Kích hoạt quy trình thu thập dữ liệu nội bộ thủ công trong trường hợp dịch vụ API bên thứ ba hoặc nền tảng AWS gặp sự cố ngưng trệ.
- Sử dụng công cụ cơ sở hạ tầng dưới dạng mã (AWS CloudFormation) để nhanh chóng khôi phục, tái tạo cấu hình mạng và dịch vụ liên quan nhằm kiểm soát sự cố.

### 8. Kết quả kỳ vọng
*Cải tiến kỹ thuật*: Hiện thực hóa tự động hóa hoàn toàn chuỗi quy trình lấy dữ liệu và sàng lọc năng lực. Phương pháp đối chiếu bằng AI mang lại độ phân giải dữ liệu cao hơn, tối ưu hóa kết quả tìm kiếm mang tính cá nhân hóa vượt trội so với phương thức lọc từ khóa rập khuôn truyền thống.
*Giá trị dài hạn*: Xây dựng nền tảng kho dữ liệu việc làm định lượng liên tục được làm giàu, tạo tiền đề phục vụ cho các phân tích chuyên sâu về xu hướng thị trường lao động. Hệ thống cho phép mở rộng quy mô một cách độc lập để tích hợp thêm các tính năng phân tích dự đoán và học máy trong các chu kỳ phát triển tiếp theo.