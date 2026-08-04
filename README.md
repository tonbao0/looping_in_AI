#  Quy trình phân tích giá cổ phiếu tự động (n8n)

## Giới thiệu

Đây là một workflow (quy trình tự động) được xây dựng trên nền tảng **n8n**, sử dụng cơ chế **vòng lặp (looping)** kết hợp với **AI Agent** để liên tục phân tích và đánh giá dữ liệu giá cổ phiếu, sau đó gửi kết quả qua Telegram và lưu lại vào Google Sheet.

Ý tưởng cốt lõi: một Agent sẽ **phân tích** dữ liệu, một Agent khác đóng vai trò **giám khảo (evaluator)** để kiểm tra chất lượng/độ tin cậy của kết quả phân tích. Nếu kết quả chưa đạt yêu cầu, quy trình sẽ **lặp lại** để phân tích lại cho đến khi đạt chuẩn, rồi mới gửi thông báo và ghi log.

##  Sơ đồ quy trình

<!-- ẢNH SƠ ĐỒ QUY TRÌNH N8N -->
<img width="1685" height="346" alt="image" src="https://github.com/user-attachments/assets/c72c4e9f-76b4-4538-9014-d69008d17d19" />


##  Các bước trong quy trình

| # | Node | Chức năng |
|---|------|-----------|
| 1 | **Telegram Trigger** | Khởi động quy trình khi có tin nhắn/lệnh gửi tới bot Telegram (ví dụ: mã cổ phiếu cần phân tích) |
| 2 | **Analyze Agent** | AI Agent thực hiện phân tích giá cổ phiếu, sử dụng: <br>• **OpenAI Chat Model** làm bộ não xử lý ngôn ngữ<br>• **Google Search (SerpApi)** làm công cụ tra cứu thông tin/tin tức thị trường mới nhất<br>• **Memory** để lưu ngữ cảnh hội thoại |
| 3 | **Evaluator Agent** | AI Agent thứ hai đóng vai trò đánh giá kết quả phân tích ở bước trên (dùng **OpenAI Chat Model1**), kiểm tra độ chính xác, tính đầy đủ và mức độ tin cậy của phân tích |
| 4 | **If1** | Điều kiện rẽ nhánh dựa trên kết quả đánh giá:<br>• **True** → kết quả đạt yêu cầu → gửi thông báo ngay qua Telegram<br>• **False** → kết quả chưa đạt → chuyển sang xử lý bổ sung |
| 5 | **Code (JavaScript)** | Xử lý logic tùy chỉnh bằng code (ví dụ: định dạng lại dữ liệu, tính số lần lặp, chuẩn bị dữ liệu đầu vào cho vòng lặp tiếp theo) |
| 6 | **If** | Điều kiện kiểm tra thứ hai (ví dụ: đã đạt số lần lặp tối đa hoặc điều kiện dừng hay chưa):<br>• **True** → gửi thông báo qua Telegram (Send a text message1)<br>• **False** → **quay lại Analyze Agent để lặp lại quá trình phân tích** |
| 7 | **Send a text message / Send a text message1** | Gửi kết quả phân tích cổ phiếu cuối cùng tới người dùng qua Telegram |
| 8 | **Append row in sheet** | Ghi lại kết quả phân tích (mã cổ phiếu, thời gian, nội dung phân tích, kết luận...) vào Google Sheet để lưu trữ và theo dõi lịch sử |

##  Nền tảng thiết kế: 4 node cơ bản

Quy trình trên được xây dựng dựa trên một khung tư duy gồm **4 node/khối cơ bản**, lặp lại theo chu kỳ "NEXT RUN" — đây chính là nguyên lý nền tảng cho cơ chế vòng lặp (looping) đã mô tả ở trên:

> Chèn ảnh sơ đồ 4 node cơ bản vào đây (dán thẻ `<img>` hoặc `![alt](link)` trực tiếp, KHÔNG để trong khối code ```):
<!-- ẢNH SƠ ĐỒ 4 NODE CƠ BẢN -->
<img width="1692" height="421" alt="image" src="https://github.com/user-attachments/assets/960d55e4-39ef-4d02-b082-cdccfdf645b8" />


| # | Node | Vai trò | Thành phần tương ứng |
|---|------|---------|----------------------|
| 01 | **One automation** | Một tác vụ tự động hóa: quy định thời điểm chạy, tần suất (cadence) và điều kiện dừng | `/loop` · `/goal` |
| 02 | **One skill** | Ngữ cảnh/kỹ năng của dự án mà agent sẽ dùng, thay vì phải tự suy luận lại từ đầu mỗi lần chạy | `SKILL.md` |
| 03 | **One state file** | Một file trạng thái ghi lại việc gì đã hoàn thành và việc gì cần làm tiếp theo | `Markdown / Linear` |
| 04 | **One gate** | Một cổng kiểm soát: bài test, type check, hoặc build sẽ chặn lại nếu kết quả không đạt | `CAN SAY NO` |

Sau khi qua **04 – One gate**, kết quả sẽ được đưa trở lại **01 – One automation** cho lần chạy tiếp theo (**NEXT RUN**), tạo thành một vòng lặp khép kín. Đây chính là mô hình mà workflow n8n ở trên áp dụng: **Analyze Agent** đóng vai trò *automation + skill*, **Evaluator Agent** đóng vai trò *gate* (quyết định "CAN SAY NO" khi kết quả chưa đạt), và việc quay lại phân tích khi gặp nhánh **False** chính là **NEXT RUN**.

##  Cơ chế vòng lặp (Looping)

Vòng lặp được tạo thành từ cặp node **Evaluator Agent → If1 → Code → If**, trong đó:

- Nếu **Evaluator Agent** đánh giá kết quả phân tích **chưa đạt**, nhánh **False** của node **If** sẽ đưa dữ liệu **quay trở lại Analyze Agent** để phân tích lại.
- Vòng lặp này tiếp tục cho đến khi kết quả phân tích đạt yêu cầu (nhánh **True**), lúc đó quy trình mới đi tiếp đến bước gửi tin nhắn và ghi vào Google Sheet.

Cơ chế này giúp đảm bảo chất lượng đầu ra: AI không chỉ phân tích một lần rồi gửi ngay, mà có một "vòng kiểm duyệt" tự động để tăng độ chính xác trước khi thông báo cho người dùng.

##  Các thành phần / công cụ sử dụng

- **Telegram**: nhận lệnh kích hoạt và gửi kết quả
- **OpenAI Chat Model**: mô hình ngôn ngữ cho Agent phân tích
- **OpenAI Chat Model1**: mô hình ngôn ngữ cho Agent đánh giá
- **Google Search (SerpApi)**: công cụ tra cứu dữ liệu/tin tức thị trường thời gian thực
- **Code node (JavaScript)**: xử lý logic, biến đổi dữ liệu
- **Google Sheets**: lưu trữ lịch sử kết quả phân tích

##  Cách sử dụng

1. Import file workflow (`.json`) vào n8n.
2. Cấu hình các credential cần thiết: Telegram Bot Token, OpenAI API Key, SerpApi Key, Google Sheets OAuth.
3. Kích hoạt (activate) workflow.
4. Gửi mã cổ phiếu cần phân tích tới bot Telegram để bắt đầu quy trình.

##  Ghi chú

- Có thể giới hạn số lần lặp tối đa trong node **Code (JavaScript)** để tránh vòng lặp vô hạn khi Evaluator Agent liên tục đánh giá "chưa đạt".
- Nội dung prompt của **Analyze Agent** và **Evaluator Agent** nên được tinh chỉnh phù hợp với tiêu chí phân tích cổ phiếu cụ thể (kỹ thuật, cơ bản, tin tức...).
