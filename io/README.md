## Streams

Hầu hết các I/O cơ bản trong Java dựa trên các luồng (streams). Một luồng biểu diễn một dòng dữ liệu với (ít nhất là theo khái niệm) một bộ ghi ở một đầu và một bộ đọc ở đầu kia. Khi bạn làm việc với gói java.io để thực hiện nhập và xuất terminal, đọc hoặc ghi các tệp tin, hoặc truyền thông qua các socket trong Java, bạn đang sử dụng các loại luồng khác nhau. Sau này trong chương này, chúng ta sẽ xem xét gói NIO, mà giới thiệu một khái niệm tương tự được gọi là kênh (channel). Một điểm khác biệt giữa hai loại này là luồng được hướng theo byte hoặc ký tự trong khi các kênh được hướng theo "bộ đệm" chứa các loại dữ liệu đó nhưng chúng thực hiện công việc tương tự. Hãy bắt đầu bằng việc tóm tắt các loại luồng có sẵn:
- InputStream, OutputStream: Các lớp trừu tượng xác định chức năng cơ bản cho việc đọc hoặc ghi một chuỗi byte không cấu trúc. Tất cả các luồng byte khác trong Java được xây dựng dựa trên InputStream và OutputStream.
- Reader, Writer: Các lớp trừu tượng xác định chức năng cơ bản cho việc đọc hoặc ghi một chuỗi dữ liệu ký tự, với hỗ trợ Unicode. Tất cả các luồng ký tự khác trong Java được xây dựng dựa trên Reader và Writer.
- InputStreamReader, OutputStreamWriter: Các lớp chuyển đổi giữa các luồng byte và ký tự bằng cách chuyển đổi theo một kế hoạch mã hóa ký tự cụ thể. (Hãy nhớ: trong Unicode, một ký tự không phải là một byte!)
- DataInputStream, DataOutputStream: Các bộ lọc luồng chuyên biệt thêm khả năng đọc và ghi các loại dữ liệu nhiều byte, như các nguyên thủy số và đối tượng String trong một định dạng thông thống.
- ObjectInputStream, ObjectOutputStream: Các bộ lọc luồng chuyên biệt có khả năng ghi toàn bộ các nhóm đối tượng Java được serialized và tái tạo chúng.
- BufferedInputStream, BufferedOutputStream, BufferedReader, BufferedWriter: Các bộ lọc luồng chuyên biệt thêm bộ đệm để tăng hiệu suất. Đối với I/O thực tế, bộ đệm gần như luôn được sử dụng.
- PrintStream, PrintWriter: Các luồng chuyên biệt giúp đơn giản hóa việc in văn bản.
- PipedInputStream, PipedOutputStream, PipedReader, PipedWriter: Các luồng "Loopback" có thể được sử dụng theo cặp để di chuyển dữ liệu trong một ứng dụng. Dữ liệu được ghi vào một PipedOutputStream hoặc PipedWriter được đọc từ PipedInputStream hoặc PipedReader tương ứng của nó.
- FileInputStream, FileOutputStream, FileReader, FileWriter: Các cài đặt của InputStream, OutputStream, Reader và Writer đọc từ và ghi vào các tệp trên hệ thống tệp cục bộ.

![Roadmap](img.png)