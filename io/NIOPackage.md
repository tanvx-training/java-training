## The NIO Package

Chúng ta sẽ hoàn thành phần giới thiệu về các cơ sở của các cơ sở Java I/O bằng cách quay lại gói java.nio. Tên viết tắt NIO là "New I/O" và, như chúng ta đã thấy trước đó trong chương này trong cuộc thảo luận về java.nio.file, một khía cạnh của NIO đơn giản là cập nhật và cải tiến các tính năng của gói cũ java.io. Thực tế, nhiều chức năng chung của NIO có sự trùng lắp với các API hiện có. Tuy nhiên, NIO ban đầu được giới thiệu để giải quyết các vấn đề cụ thể về khả năng mở rộng cho các hệ thống lớn, đặc biệt là trong các ứng dụng mạng. Phần tiếp theo sẽ trình bày các yếu tố cơ bản của NIO, tập trung vào việc làm việc với bộ đệm và kênh.

### Asynchronous I/O

Hầu hết nhu cầu cho gói NIO được thúc đẩy bởi mong muốn thêm I/O không chặn và có thể chọn vào Java. Trước khi có NIO, hầu hết các hoạt động đọc và ghi trong Java đều bị ràng buộc vào các luồng và bị buộc phải chặn trong khoảng thời gian không đoán trước. Mặc dù một số API như Sockets (mà chúng ta sẽ thấy trong Chương 13) cung cấp các phương tiện cụ thể để giới hạn thời gian một cuộc gọi I/O có thể mất, điều này chỉ là một biện pháp tạm thời để bù đắp cho việc thiếu một cơ chế tổng quát hơn. Trong nhiều ngôn ngữ, ngay cả những ngôn ngữ không sử dụng luồng, I/O vẫn có thể được thực hiện một cách hiệu quả bằng cách đặt các luồng I/O vào chế độ không chặn và kiểm tra chúng để xem liệu chúng sẵn sàng để gửi hoặc nhận dữ liệu hay không. Trong chế độ không chặn, một cuộc đọc hoặc ghi chỉ thực hiện một lượng công việc có thể thực hiện được ngay lập tức - điền vào hoặc làm rỗng một bộ đệm và sau đó trả về. Kết hợp với khả năng kiểm tra sự sẵn sàng, điều này cho phép một ứng dụng với một luồng duy nhất phục vụ liên tục nhiều kênh một cách hiệu quả. Luồng chính "chọn" một luồng dữ liệu đã sẵn sàng và làm việc với nó cho đến khi nó bị chặn và sau đó chuyển sang một luồng khác. Trên một hệ thống với một bộ xử lý, điều này cơ bản tương đương với việc sử dụng nhiều luồng. Rất thú vị khi phát hiện ra rằng kiểu xử lý này có các ưu điểm về khả năng mở rộng ngay cả khi sử dụng một pool luồng (thay vì chỉ một). Chúng ta sẽ thảo luận về điều này chi tiết trong Chương 13 khi thảo luận về mạng và xây dựng máy chủ có thể xử lý nhiều khách hàng đồng thời.

Ngoài I/O không chặn và có thể chọn, gói NIO cũng cho phép đóng và ngắt các hoạt động I/O một cách không đồng bộ. Như đã thảo luận trong Chương 9, trước khi có NIO, không có cách đáng tin cậy nào để dừng hoặc đánh thức một luồng bị chặn trong một hoạt động I/O. Với NIO, các luồng bị chặn trong các hoạt động I/O luôn đánh thức khi bị ngắt hoặc khi kênh được đóng bởi bất kỳ ai. Ngoài ra, nếu bạn ngắt một luồng trong khi nó bị chặn trong một hoạt động NIO, kênh của nó sẽ tự động đóng. (Việc đóng kênh vì luồng bị ngắt có vẻ mạnh mẽ, nhưng thường là đúng điều đó.)

### Performance

Kênh I/O được thiết kế xung quanh khái niệm của bộ đệm, đó là một dạng tinh vi của mảng, được điều chỉnh để làm việc với truyền thông. Gói NIO hỗ trợ khái niệm của bộ đệm trực tiếp - các bộ đệm duy trì bộ nhớ của chúng bên ngoài máy ảo Java trong hệ điều hành máy chủ. Vì tất cả các hoạt động I/O thực tế cuối cùng đều phải làm việc với hệ điều hành máy chủ bằng cách duy trì không gian bộ đệm ở đó, một số hoạt động có thể trở nên hiệu quả hơn nhiều. Dữ liệu di chuyển giữa hai điểm cuối bên ngoài có thể được chuyển mà không cần sao chép trước vào Java và sau đó trở lại.

### Mapped and Locked Files

NIO cung cấp hai tính năng liên quan đến tệp không tồn tại trong java.io: các tệp được ánh xạ vào bộ nhớ và khóa tệp. Chúng ta sẽ thảo luận về các tệp được ánh xạ vào bộ nhớ sau, nhưng đủ để nói rằng chúng cho phép bạn làm việc với dữ liệu tệp như nếu chúng đều nằm trong bộ nhớ một cách kỳ diệu. Khóa tệp hỗ trợ khái niệm về khóa chia sẻ và khóa độc quyền trên các khu vực của tệp - hữu ích cho việc truy cập song song bởi nhiều ứng dụng.

### Channels

Trong khi java.io xử lý với các luồng, java.nio làm việc với các kênh (channels). Một kênh là một điểm kết nối cho giao tiếp. Mặc dù trong thực tế các kênh tương tự như các luồng, khái niệm cơ bản của một kênh là trừu tượng và nguyên thủy hơn. Trong khi các luồng trong java.io được định nghĩa dựa trên đầu vào hoặc đầu ra với các phương thức để đọc và ghi byte, giao diện kênh cơ bản chỉ nói về việc giao tiếp xảy ra như thế nào. Nó chỉ có khái niệm về việc mở hoặc đóng, được hỗ trợ thông qua các phương thức isOpen() và close(). Các triển khai của các kênh cho tệp, ổ cắm mạng hoặc các thiết bị tùy ý sau đó thêm các phương thức của riêng họ để thực hiện các hoạt động, chẳng hạn như đọc, ghi hoặc chuyển dữ liệu. Các kênh sau được cung cấp bởi NIO:

- FileChannel
- Pipe.SinkChannel, Pipe.SourceChannel
- SocketChannel, ServerSocketChannel, DatagramChannel

Chúng ta sẽ bàn về FileChannel trong chương này. Các kênh Ống (Pipe) đơn giản chỉ là các kênh tương đương với các cơ sở dữ liệu Ống (Pipe) trong java.io. Chúng ta sẽ nói về các kênh Socket và Datagram trong Chương 13. Ngoài ra, trong Java 7, hiện đã có các phiên bản không đồng bộ của cả hai kênh tệp và kênh socket: AsynchronousFileChannel, AsynchronousSocketChannel, AsynchronousServerSocketChannel và AsynchronousDatagramChannel. Những phiên bản không đồng bộ này cơ bản làm đệm tất cả các hoạt động của chúng thông qua một pool luồng và báo cáo kết quả lại thông qua một API không đồng bộ. Chúng ta sẽ nói về kênh tệp không đồng bộ sau trong chương này.

Tất cả các kênh cơ bản này đều thực hiện giao diện ByteChannel, được thiết kế cho các kênh có các phương thức đọc và ghi như các luồng I/O. ByteChannels đọc và ghi Byte Buffers, tuy nhiên, khác với các mảng byte đơn giản.

Ngoài các triển khai kênh này, bạn có thể kết nối các kênh với luồng I/O và người đọc và người viết java.io để tương thích. Tuy nhiên, nếu bạn kết hợp các tính năng này, bạn có thể không nhận được tất cả các lợi ích và hiệu suất được cung cấp bởi gói NIO.

### Buffers

Hầu hết các tiện ích của các gói java.io và java.net hoạt động trên các mảng byte. Các công cụ tương ứng của gói NIO được xây dựng xung quanh ByteBuffers (với bộ đệm dựa trên ký tự CharBuffer cho văn bản). Mảng byte đơn giản, vì vậy tại sao cần có bộ đệm?

Chúng phục vụ một số mục đích:

- Chúng hệ thống hóa các mẫu sử dụng cho dữ liệu được đệm, cung cấp cho những điều như bộ đệm chỉ đọc và theo dõi các vị trí đọc/giữa và giới hạn trong một không gian bộ đệm lớn. Chúng cũng cung cấp một cơ sở đánh dấu/làm mới tương tự như của java.io.BufferedInputStream.

- Chúng cung cấp các API bổ sung để làm việc với dữ liệu nguyên thủy đại diện. Bạn có thể tạo các bộ đệm "nhìn" vào dữ liệu byte của bạn như là một chuỗi các nguyên thủy lớn hơn, chẳng hạn như short, int hoặc float. Loại dữ liệu buffer tổng quát nhất, Byte Buffer, bao gồm các phương thức cho phép bạn đọc và ghi tất cả các loại nguyên thủy giống như DataOutputStream làm cho luồng.

- Chúng trừu tượng hóa lưu trữ dữ liệu cơ bản, cho phép tối ưu hóa đặc biệt bởi Java. Cụ thể, các bộ đệm có thể được cấp phát dưới dạng các bộ đệm trực tiếp sử dụng các bộ đệm cấp thấp của hệ điều hành máy chủ thay vì các mảng trong bộ nhớ Java. Các cơ sở NIO Channel làm việc với bộ đệm có thể nhận dạng bộ đệm trực tiếp tự động và cố gắng tối ưu hóa I/O để sử dụng chúng. Ví dụ, một lệnh đọc từ một kênh tệp vào một mảng byte Java thông thường đòi hỏi Java sao chép dữ liệu để đọc từ hệ điều hành máy chủ vào bộ nhớ của Java. Với một bộ đệm trực tiếp, dữ liệu có thể được giữ lại trong hệ điều hành máy chủ, bên ngoài không gian bộ nhớ bình thường của Java cho đến khi và trừ khi cần thiết.

#### Buffer operations

Một buffer là một lớp con của một đối tượng java.nio.Buffer. Lớp cơ sở Buffer giống như một mảng với trạng thái. Nó không xác định loại phần tử nó giữ (điều đó là để các lớp con quyết định), nhưng nó xác định các chức năng phổ biến cho tất cả các bộ đệm dữ liệu.

Một Buffer có một kích thước cố định gọi là dung lượng của nó. Mặc dù tất cả các Buffers tiêu chuẩn cung cấp "truy cập ngẫu nhiên" đến nội dung của chúng, một Buffer thông thường mong đợi được đọc và ghi tuần tự, vì vậy Buffers duy trì khái niệm về vị trí mà phần tử tiếp theo được đọc hoặc ghi. Ngoài vị trí, một Buffer có thể duy trì hai thông tin trạng thái khác: một giới hạn, là một vị trí là một giới hạn "mềm" đến mức độ đọc hoặc ghi, và một dấu hiệu, có thể được sử dụng để nhớ một vị trí trước đó để gọi lại trong tương lai.

Các triển khai của Buffer thêm các phương thức get và put cụ thể, được gõ chức năng đọc và ghi nội dung của bộ đệm. Ví dụ, ByteBuffer là một buffer của byte và nó có các phương thức get() và put() để đọc và ghi byte và mảng byte (cùng với nhiều phương thức hữu ích khác chúng ta sẽ thảo luận sau). Việc lấy và đặt vào Buffer thay đổi vị trí đánh dấu, vì vậy Buffer theo dõi nội dung của nó giống như một luồng. Cố gắng đọc hoặc ghi vượt quá giới hạn dấu hiệu sẽ tạo ra BufferUnderflowException hoặc BufferOverflowException, tương ứng.

Các giá trị của dấu hiệu, vị trí, giới hạn và dung lượng luôn tuân theo công thức sau:
```
mark <= position <= limit <= capacity
```
Vị trí để đọc và ghi vào Buffer luôn nằm giữa dấu hiệu, làm nhiệm vụ như một giới hạn dưới, và giới hạn, làm nhiệm vụ như một giới hạn trên. Dung lượng đại diện cho phạm vi vật lý của không gian bộ đệm.

Bạn có thể thiết lập các dấu hiệu vị trí và giới hạn một cách rõ ràng bằng các phương thức position() và limit(). Cung cấp một số phương thức thuận tiện cho các mẫu sử dụng phổ biến. Phương thức reset() đặt vị trí trở lại dấu hiệu. Nếu không có dấu hiệu nào đã được đặt, một InvalidMarkException sẽ được ném ra. Phương thức clear() đặt vị trí về 0 và làm cho giới hạn bằng dung lượng, chuẩn bị bộ đệm cho dữ liệu mới (dấu hiệu bị loại bỏ).

Lưu ý rằng phương thức clear() không thực sự làm bất cứ điều gì với dữ liệu trong bộ đệm; nó chỉ đơn giản là thay đổi các dấu hiệu vị trí. Phương thức flip() được sử dụng cho mẫu phổ biến của việc ghi dữ liệu vào bộ đệm và sau đó đọc nó ra. flip đặt vị trí hiện tại là giới hạn và sau đó đặt lại vị trí hiện tại về 0 (bất kỳ dấu hiệu nào cũng được bỏ qua), điều này giúp tiết kiệm việc theo dõi có bao nhiêu dữ liệu đã được đọc. Một phương thức khác, rewind(), đơn giản là đặt lại vị trí về 0, để lại giới hạn không đổi. Bạn có thể sử dụng nó để ghi lại cùng kích thước dữ liệu. Dưới đây là một đoạn mã sử dụng các phương thức này để đọc dữ liệu từ một kênh và ghi vào hai kênh:
```java
ByteBuffer buff = ...
while (inChannel.read(buff) > 0) { // position = ?
    buff.flip(); // limit = position; position = 0;
    outChannel.write(buff);
    buff.rewind(); // position = 0
    outChannel2.write(buff);
    buff.clear(); // position = 0; limit = capacity
}
```
Điều này có thể làm cho bạn rối bời lần đầu tiên bạn nhìn vào nó vì ở đây, việc đọc từ Channel thực sự là một việc ghi vào Buffer và ngược lại. Vì ví dụ này viết tất cả dữ liệu có sẵn lên đến giới hạn, flip() hoặc rewind() có cùng hiệu quả trong trường hợp này.

#### Buffer types 

Như đã nêu trước đó, các loại bộ đệm khác nhau thêm các phương thức get và put để đọc và ghi các loại dữ liệu cụ thể. Mỗi loại nguyên thủy trong Java đều có một loại bộ đệm đi kèm: Byte Buffer, Char Buffer, Short Buffer, Int Buffer, Long Buffer, Float Buffer và Double Buffer. Mỗi loại này cung cấp các phương thức get và put để đọc và ghi loại của nó và các mảng của loại đó. Trong số này, ByteBuffer là linh hoạt nhất. Bởi vì nó có "hạt cát tinh tế" nhất của tất cả các bộ đệm, nó đã được trang bị một bộ đầy đủ các phương thức get và put để đọc và ghi tất cả các loại dữ liệu khác cũng như byte. Dưới đây là một số phương thức của ByteBuffer:

byte get()
char getChar()
short getShort()
int getInt()
long getLong()
float getFloat()
double getDouble()
void put(byte b)
void put(ByteBuffer src)
void put(byte[] src, int offset, int length)
void put(byte[] src)
void putChar(char value)
void putShort(short value)
void putInt(int value)
void putLong(long value)
void putFloat(float value)
void putDouble(double value)

Như chúng tôi đã nói, tất cả các bộ đệm tiêu chuẩn cũng hỗ trợ truy cập ngẫu nhiên. Đối với mỗi phương thức của ByteBuffer được đề cập trước đó, một dạng bổ sung khác nhận một chỉ mục; ví dụ:

getLong( int index )
putLong( int index, long value )

Nhưng đó không phải là tất cả. ByteBuffer cũng có thể cung cấp "các góc nhìn" của chính nó dưới dạng bất kỳ một trong các loại có cấu trúc thô. Ví dụ, bạn có thể lấy một góc nhìn Short Buffer của ByteBuffer với phương thức asShortBuffer(). Góc nhìn Short Buffer được hỗ trợ bởi ByteBuffer, điều này có nghĩa là chúng hoạt động trên cùng một dữ liệu và các thay đổi đối với một trong hai ảnh hưởng đến cái kia. Phạm vi của bộ đệm góc nhìn bắt đầu từ vị trí hiện tại của ByteBuffer, và dung lượng của nó là một hàm số của số byte còn lại, chia cho kích thước của loại mới. (Ví dụ, short chiếm hai byte mỗi byte, float bốn byte, và longs và doubles chiếm tám byte.) Bộ đệm góc nhìn là tiện ích để đọc và ghi các khối lớn của một loại liên tục trong một ByteBuffer.

CharBuffers cũng rất thú vị, chủ yếu là do tích hợp của chúng với Strings. Cả CharBuffers và Strings đều thực thi giao diện java.lang.CharSequence. Đây là giao diện cung cấp các phương thức charAt() và length() chuẩn. Do đó, các API mới (như gói java.util.regex) cho phép bạn sử dụng một CharBuffer hoặc một String một cách tương đương. Trong trường hợp này, CharBuffer hoạt động như một String có thể sửa đổi với vị trí bắt đầu và kết thúc được định cấu hình bởi người dùng.

#### Byte order

Bởi vì chúng ta đang nói về việc đọc và ghi các loại dữ liệu lớn hơn một byte, câu hỏi đặt ra là: các byte của các giá trị đa byte (ví dụ, shorts và ints) được ghi theo thứ tự nào?
Có hai trường phái trong thế giới này: "big endian" và "little endian". Big endian có nghĩa là byte có giá trị quan trọng nhất được ghi trước; little endian thì ngược lại. Nếu bạn đang viết dữ liệu nhị phân để sử dụng bởi một ứng dụng native nào đó, điều này rất quan trọng. Các máy tính tương thích với Intel sử dụng little endian, và nhiều máy trạm chạy Unix sử dụng big endian. Lớp ByteOrder đóng gói lựa chọn này. Bạn có thể chỉ định thứ tự byte để sử dụng với phương thức order() của ByteBuffer, sử dụng các định danh ByteOrder.BIG_ENDIAN và ByteOrder.LITTLE_ENDIAN như sau:
```java
byteArray.order(ByteOrder.BIG_ENDIAN);
```
Bạn có thể lấy thứ tự mặc định cho nền tảng của bạn bằng cách sử dụng phương thức static ByteOrder.nativeOrder(). (Tôi biết bạn tò mò.)

#### Allocating buffers

Bạn có thể tạo một bộ đệm bằng cách cấp phát một cách rõ ràng bằng cách sử dụng allocate() hoặc bằng cách bọc một loại mảng Java đơn giản hiện có. Mỗi loại bộ đệm đều có một phương thức allocate() tĩnh nhận một dung lượng (kích thước) và cũng có một phương thức wrap() nhận một mảng hiện có:
```java
CharBuffer cbuf = CharBuffer.allocate(64 * 1024);
```
Một bộ đệm trực tiếp được cấp phát theo cùng một cách, với phương thức allocateDirect():
```java
ByteBuffer bbuf = ByteBuffer.allocateDirect(64 * 1024);
```
```java
ByteBuffer bbuf2 = ByteBuffer.wrap(someExistingArray);
```
Như chúng tôi đã mô tả trước đó, các bộ đệm trực tiếp có thể sử dụng các cấu trúc bộ nhớ của hệ điều hành được tối ưu hóa cho việc sử dụng với một số loại thao tác I/O. Sự đánh đổi là việc cấp phát một bộ đệm trực tiếp là một thao tác chậm và nặng hơn một chút so với một bộ đệm thông thường, vì vậy bạn nên cố gắng sử dụng chúng cho các bộ đệm dài hạn.

### Character Encoders and Decoders

Trình mã hóa và giải mã ký tự chuyển đổi các ký tự thành các byte thuần túy và ngược lại, ánh xạ từ tiêu chuẩn Unicode sang các lược đồ mã hóa cụ thể. Trình mã hóa và giải mã đã tồn tại lâu trong Java để sử dụng bởi các luồng Reader và Writer và trong các phương thức của lớp String làm việc với các mảng byte. Tuy nhiên, ban đầu không có API để làm việc với mã hóa một cách rõ ràng; bạn đơn giản chỉ tham chiếu đến trình mã hóa và giải mã mỗi khi cần bằng tên như một String. Gói java.nio.charset đã hình thành ý tưởng về một bộ ký tự Unicode với lớp Charset.

Lớp Charset là một nhà máy cho các trường hợp của Charset, biết cách mã hóa bộ đệm ký tự thành bộ đệm byte và giải mã bộ đệm byte thành bộ đệm ký tự. Bạn có thể tìm kiếm một bộ ký tự bằng tên với phương thức static Charset.forName() và sử dụng nó trong các chuyển đổi:
```java
Charset charset = Charset.forName("US-ASCII");
CharBuffer charBuff = charset.decode(byteBuff); // chuyển sang ascii
ByteBuffer byteBuff = charset.encode(charBuff); // và ngược lại
```
Bạn cũng có thể kiểm tra xem một mã hóa có sẵn không bằng phương thức static Charset.isSupported().

Các bộ ký tự sau đây được đảm bảo sẽ được cung cấp:
- US-ASCII
- ISO-8859-1
- UTF-8
- UTF-16BE
- UTF-16LE
- UTF-16

Bạn có thể liệt kê tất cả các trình mã hóa có sẵn trên nền tảng của bạn bằng phương thức static availableCharsets():
```java
Map<String, Charset> map = Charset.availableCharsets();
Iterator<String> it = map.keySet().iterator();
while (it.hasNext())
    System.out.println(it.next());
```
Kết quả của availableCharsets() là một bản đồ vì các bộ ký tự có thể có "bí danh" và xuất hiện dưới nhiều tên.

Ngoài các lớp hướng bộ đệm của gói java.nio, các lớp cầu nối InputStream Reader và OutputStreamWriter của gói java.io cũng đã được cập nhật để làm việc với Charset. Bạn có thể chỉ định mã hóa dưới dạng một đối tượng Charset hoặc bằng tên.

#### CharsetEncoder and CharsetDecoder

Bạn có thể có được sự kiểm soát hơn về quá trình mã hóa và giải mã bằng cách tạo một trường hợp của CharsetEncoder hoặc CharsetDecoder (một bộ mã) với các phương thức newEncoder() và newDecoder() của Charset. Trong đoạn mã trước đó, chúng ta giả định rằng tất cả dữ liệu đều có sẵn trong một bộ đệm duy nhất. Tuy nhiên, thường xuyên hơn, chúng ta có thể phải xử lý dữ liệu khi nó đến theo từng phần. API mã hóa/giải mã cho phép điều này bằng cách cung cấp các phương thức encode() và decode() tổng quát hơn, nhận một cờ chỉ định liệu có thêm dữ liệu được mong đợi không. Bộ mã cần biết điều này vì nó có thể đã bị treo giữa quá trình chuyển đổi ký tự đa byte khi dữ liệu cạn kiệt. Nếu nó biết rằng có thêm dữ liệu đang đến, nó sẽ không gây ra lỗi trong quá trình chuyển đổi chưa hoàn chỉnh này. Trong đoạn mã sau đây, chúng ta sử dụng một bộ giải mã để đọc từ ByteBuffer bbuff và tích lũy dữ liệu ký tự vào CharBuffer cbuff:

```java
CharsetDecoder decoder = Charset.forName("US-ASCII").newDecoder();
boolean done = false;
while (!done) {
    bbuff.clear();
    done = (in.read(bbuff) == -1);
    bbuff.flip();
    decoder.decode(bbuff, cbuff, done);
}
cbuff.flip();
// sử dụng cbuff. . .
```

Ở đây, chúng ta tìm kiếm điều kiện kết thúc đầu vào trên kênh đầu vào để đặt cờ done. Lưu ý rằng chúng ta tận dụng phương thức flip() trên ByteBuffer để đặt giới hạn là lượng dữ liệu đã đọc và đặt lại vị trí, chuẩn bị cho quá trình giải mã trong một bước. Các phương thức encode() và decode() cũng trả về một đối tượng kết quả, CoderResult, có thể xác định tiến trình của việc mã hóa (chúng ta không sử dụng nó trong đoạn mã trước đó). Các phương thức isError(), isUnderflow() và isOverflow() trên CoderResult chỉ định lý do tại sao việc mã hóa dừng lại: cho một lỗi, một thiếu byte trên bộ đệm đầu vào, hoặc một bộ đệm đầu ra đầy, tương ứng.

### FileChannel

Bây giờ chúng ta đã bao quát được cơ bản về các kênh và bộ đệm, là lúc để xem xét một loại kênh thực sự. FileChannel là tương đương của NIO với java.io.RandomAccessFile, nhưng nó cung cấp một số tính năng mới chính ngoài một số tối ưu hóa hiệu suất. Đặc biệt, sử dụng một FileChannel thay cho một luồng tệp plain java.io nếu bạn muốn sử dụng khóa tệp, truy cập tệp được ánh xạ vào bộ nhớ, hoặc truyền dữ liệu được tối ưu hóa cao giữa các tệp hoặc giữa kênh tệp và mạng.

Một FileChannel có thể được tạo cho một Path bằng cách sử dụng phương thức static FileChannel.open().
```java
FileSystem fs = FileSystems.getDefault();
Path p = fs.getPath("/tmp/foo.txt");
// Mở mặc định để đọc
try (FileChannel channel = FileChannel.open(p)) {
    ...
}
```
```java
// Mở với tùy chọn để ghi
import static java.nio.file.StandardOpenOption.*;
try (FileChannel channel = FileChannel.open(p, WRITE, APPEND, ...)) {
    ...
}
```
Mặc định, open() tạo một kênh chỉ đọc cho tệp. Chúng ta có thể mở một kênh để ghi hoặc nối và kiểm soát các tính năng tiên tiến khác như tạo nguyên tử và đồng bộ dữ liệu bằng cách truyền các tùy chọn bổ sung như đã được hiển thị trong phần thứ hai của ví dụ trước đó. Bảng 12-4 tóm tắt các tùy chọn này.

| Tùy chọn            | Mô tả                                                                                                                                                                |
|---------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| READ, WRITE         | Mở tệp chỉ cho đọc hoặc chỉ cho ghi (mặc định là chỉ cho đọc). Sử dụng cả hai để mở cho đọc-viết.                                                                   |
| APPEND              | Mở tệp cho việc ghi; tất cả các ghi được định vị ở cuối tệp.                                                                                                          |
| CREATE              | Sử dụng với WRITE để mở tệp và tạo nó nếu cần.                                                                                                                      |
| CREATE_NEW          | Sử dụng với WRITE để tạo một tệp một cách nguyên tử; thất bại nếu tệp đã tồn tại.                                                                                     |
| DELETE_ON_CLOSE     | Cố gắng xóa tệp khi nó được đóng hoặc, nếu mở, khi VM thoát.                                                                                                          |
| SYNC, DSYNC         | Ở mức có thể, đảm bảo rằng các thao tác ghi chặn cho đến khi tất cả dữ liệu được ghi vào bộ nhớ. SYNC làm điều này cho tất cả các thay đổi tệp bao gồm cả dữ liệu và siêu dữ liệu (thuộc tính) trong khi DSYNC chỉ thêm yêu cầu này cho nội dung dữ liệu của tệp. |
| SPARSE              | Sử dụng khi tạo một tệp mới, yêu cầu tệp được hiểu là rải rác. Trên các hệ thống tệp hỗ trợ điều này, một tệp rải rác xử lý các tệp rất lớn, phần lớn là trống mà không cấp phát nhiều bộ nhớ thực cho các phần trống.   |
| TRUNCATE_EXISTING  | Sử dụng WRITE trên một tệp hiện có, đặt độ dài của tệp thành không khi mở nó.                                                                                          |

Một FileChannel cũng có thể được tạo ra từ một FileInputStream, FileOutputStream, hoặc RandomAccessFile cổ điển:

```java
FileChannel readOnlyFc = new FileInputStream("file.txt").getChannel();
FileChannel readWriteFc = new RandomAccessFile("file.txt", "rw").getChannel();
```

Các FileChannel được tạo ra từ các luồng nhập và xuất tệp này là chỉ đọc hoặc chỉ ghi, tương ứng. Để có một FileChannel đọc/viết, bạn phải tạo một RandomAccessFile với quyền đọc/viết, như trong ví dụ trước đó.

Sử dụng một FileChannel tương tự như một RandomAccessFile, nhưng nó hoạt động với ByteBuffer thay vì mảng byte:

```java
ByteBuffer bbuf = ByteBuffer.allocate(...);
bbuf.clear();
readOnlyFc.position(index);
readOnlyFc.read(bbuf);
bbuf.flip();
readWriteFc.write(bbuf);
```

Bạn có thể kiểm soát lượng dữ liệu được đọc và ghi bằng cách đặt vị trí và giới hạn của bộ đệm hoặc sử dụng một dạng khác của phương thức đọc/viết nhận vị trí bắt đầu của bộ đệm và độ dài. Bạn cũng có thể đọc và viết vào một vị trí ngẫu nhiên bằng cách cung cấp các chỉ số với các phương thức đọc và viết:

```java
readWriteFc.read(bbuf, index);
readWriteFc.write(bbuf, index2);
```

Trong mỗi trường hợp, số byte thực sự được đọc hoặc viết phụ thuộc vào một số yếu tố. Thao tác cố gắng đọc hoặc viết đến giới hạn của bộ đệm, và hầu hết thời gian đó là điều xảy ra với việc truy cập tệp địa phương. Thao tác được đảm bảo chặn chỉ cho đến khi ít nhất một byte đã được xử lý. Dù có điều gì xảy ra, số byte đã được xử lý được trả về, và vị trí của bộ đệm được cập nhật tương ứng, chuẩn bị cho bạn để lặp lại thao tác cho đến khi nó hoàn thành nếu cần. Điều này là một trong những tiện ích của việc làm việc với bộ đệm; chúng có thể quản lý số lượng cho bạn. Tương tự như các luồng tiêu chuẩn, phương thức read() của kênh trả về -1 khi đạt đến cuối đầu vào.

Kích thước của tệp luôn có sẵn với phương thức size(). Nó có thể thay đổi nếu bạn ghi qua cuối tệp. Ngược lại, bạn có thể cắt tệp thành một độ dài cụ thể với phương thức truncate().

#### Concurrent access

FileChannels được an toàn cho việc sử dụng bởi nhiều luồng và đảm bảo rằng dữ liệu "nhìn thấy" bởi chúng là nhất quán qua các kênh trong cùng một VM. Trừ khi bạn chỉ định các tùy chọn SYNC hoặc DSYNC, không có bất kỳ đảm bảo nào được thực hiện về tốc độ ghi được truyền đến cơ chế lưu trữ. Nếu bạn chỉ đơn giản cần đảm bảo rằng dữ liệu an toàn trước khi tiếp tục, bạn có thể sử dụng phương thức force() để đẩy các thay đổi lên đĩa. Phương thức force() nhận một đối số Boolean chỉ định liệu các siêu dữ liệu của tệp, bao gồm timestamp và quyền, phải được ghi (sync hoặc dsync) hay không. Một số hệ thống theo dõi cả đọc và ghi trên các tệp, vì vậy bạn có thể tiết kiệm nhiều cập nhật nếu bạn đặt cờ thành false, cho biết rằng bạn không quan tâm đến việc đồng bộ hóa dữ liệu đó ngay lập tức.

Tương tự như với tất cả các Channels, một FileChannel có thể được đóng bởi bất kỳ luồng nào. Một khi đã đóng, tất cả các phương thức đọc/viết và liên quan đến vị trí của nó sẽ ném một ClosedChannelException.

#### File locking

FileChannels hỗ trợ khóa độc quyền và chia sẻ trên các khu vực của các tệp thông qua phương thức lock():

```java
FileLock fileLock = fileChannel.lock();
int start = 0, len = fileChannel2.size();
FileLock readLock = fileChannel2.lock(start, len, true);
```

Các khóa có thể là khóa chia sẻ hoặc độc quyền. Một khóa độc quyền ngăn các bên khác khỏi việc có được một loại khóa nào đó trên tệp hoặc khu vực tệp được chỉ định. Một khóa chia sẻ cho phép các bên khác có được các khóa chia sẻ chồng lên nhau nhưng không phải là các khóa độc quyền. Chúng rất hữu ích khi bạn muốn khóa ghi và đọc, tương ứng. Khi bạn đang viết, bạn không muốn người khác có thể viết cho đến khi bạn hoàn thành, nhưng khi đọc, bạn chỉ cần chặn người khác khỏi việc viết, không phải là đọc đồng thời.

Phương thức lock() không có đối số trong ví dụ trước đó cố gắng có được một khóa độc quyền cho toàn bộ tệp. Hình thức thứ hai chấp nhận một tham số bắt đầu và độ dài cũng như một cờ chỉ ra liệu khóa nên được chia sẻ (hoặc độc quyền). Đối tượng FileLock được trả về bởi phương thức lock() có thể được sử dụng để giải phóng khóa:

```java
fileLock.release();
```

Lưu ý rằng các khóa tệp chỉ được đảm bảo bởi một API hợp tác; chúng không nhất thiết ngăn ai đó khác đọc hoặc viết vào nội dung tệp đã khóa. Nói chung, cách duy nhất để đảm bảo rằng các khóa được tuân thủ là cả hai bên đều cố gắng có được khóa và sử dụng nó. Ngoài ra, các khóa chia sẻ không được thực hiện trên một số hệ thống, trong trường hợp đó tất cả các khóa được yêu cầu là độc quyền. Bạn có thể kiểm tra xem một khóa có được chia sẻ hay không bằng phương thức isShared().

Các khóa của FileChannel được giữ cho đến khi kênh được đóng hoặc bị gián đoạn, vì vậy việc thực hiện các khóa trong một tuyên bố try-with-resources sẽ giúp đảm bảo rằng các khóa được giải phóng một cách mạnh mẽ hơn:

```java
try (FileChannel channel = FileChannel.open(p, WRITE)) {
    channel.lock();
    ...
}
```

#### Memory-mapped files

Một trong những tính năng thú vị được cung cấp thông qua FileChannel là khả năng ánh xạ một tệp vào bộ nhớ. Khi một tệp được ánh xạ vào bộ nhớ, giống như phép màu, nó trở nên truy cập được thông qua một ByteBuffer duy nhất—như thể toàn bộ tệp đã được đọc vào bộ nhớ cùng một lúc. Việc triển khai này cực kỳ hiệu quả, thường là một trong những cách nhanh nhất để truy cập dữ liệu. Đối với việc làm việc với các tệp lớn, ánh xạ bộ nhớ có thể tiết kiệm rất nhiều tài nguyên và thời gian.

Có thể có vẻ nghịch lý; chúng ta đang có một cách truy cập dữ liệu dễ hiểu một cách khái niệm và nó cũng nhanh hơn và hiệu quả hơn? Có gì là lời hứa? Thực ra không có gì là lời hứa. Lý do cho điều này là tất cả các hệ điều hành hiện đại dựa trên ý tưởng của bộ nhớ ảo. Nói một cách ngắn gọn, điều đó có nghĩa là hệ điều hành khiến không gian đĩa hoạt động giống như bộ nhớ bằng cách liên tục trang (đổi chỗ các khối 4KB gọi là "trang") giữa bộ nhớ và đĩa, trong khi ứng dụng không nhận biết. Hệ điều hành làm điều này rất tốt; chúng lưu trữ dữ liệu mà ứng dụng đang sử dụng một cách hiệu quả và thả các phần không được sử dụng. Ánh xạ bộ nhớ của một tệp thực sự chỉ là việc tận dụng những gì hệ điều hành đang làm bên trong.

Một ví dụ tốt về nơi một tệp được ánh xạ vào bộ nhớ sẽ hữu ích là trong một cơ sở dữ liệu. Hãy tưởng tượng một tệp 10 GB chứa các bản ghi được chỉ mục tại các vị trí khác nhau. Bằng cách ánh xạ tệp, chúng ta có thể làm việc với một ByteBuffer tiêu chuẩn, đọc và ghi dữ liệu tại các vị trí tùy ý và cho phép hệ điều hành cơ bản đọc và ghi dữ liệu dưới cấu trúc trang chi tiết khi cần thiết. Chúng ta có thể mô phỏng hành vi này bằng RandomAccessFile hoặc FileChannel, nhưng chúng ta sẽ phải đọc và ghi dữ liệu vào bộ đệm một cách rõ ràng trước, và việc triển khai hầu như chắc chắn sẽ không hiệu quả như vậy.

Một ánh xạ được tạo ra bằng phương thức map() của FileChannel. Ví dụ:

```java
FileChannel fc = FileChannel.open(fs.getPath("index.db"), CREATE, READ, WRITE);
MappedByteBuffer mappedBuff = fc.map(FileChannel.MapMode.READ_WRITE, 0, fc.size());
```

Phương thức map() trả về một MappedByteBuffer, đơn giản là ByteBuffer tiêu chuẩn với một vài phương thức bổ sung liên quan đến ánh xạ. Quan trọng nhất là phương thức force(), đảm bảo rằng bất kỳ dữ liệu được ghi vào bộ đệm đều được đẩy ra lưu trữ vĩnh viễn trên đĩa. Các hằng số READ_ONLY và READ_WRITE của lớp inner static FileChannel.MapMode chỉ định loại truy cập. Truy cập đọc/viết chỉ có sẵn khi ánh xạ một kênh tệp đọc/viết. Dữ liệu được đọc thông qua bộ đệm luôn nhất quán trong cùng một Java VM. Nó cũng có thể nhất quán trên các ứng dụng trên cùng một máy host, nhưng điều này không được đảm bảo. Một lần nữa,

MappedByteBuffer hoạt động giống như một ByteBuffer. Tiếp tục với ví dụ trước đó, chúng ta có thể giải mã bộ đệm với một bộ giải mã ký tự và tìm kiếm một mẫu như sau:
```java
CharBuffer cbuff = Charset.forName("US-ASCII").decode(mappedBuff);
Matcher matcher = Pattern.compile("abc*").matcher(cbuff);
while (matcher.find())
    System.out.println(matcher.start() + ": " + matcher.group(0));
```
Ở đây, chúng ta đã triển khai một cái gì đó giống như lệnh Unix grep dựa trên API Regular Expression làm việc với CharBuffer của chúng ta như một CharSequence. Chúng ta đã lừa một chút trong ví dụ này vì CharBuffer được cấp phát bởi phương thức decode() có kích thước bằng với tệp đã được ánh xạ và phải được giữ trong bộ nhớ. Để làm điều này một cách hiệu quả, chúng ta có thể sử dụng CharsetDecoder đã thảo luận trước đó trong chương này để lặp lại qua không gian ánh xạ lớn mà không kéo hết mọi thứ vào bộ nhớ.

#### Direct transfer

Các tính năng cuối cùng của FileChannel mà chúng ta sẽ xem xét là tối ưu hóa hiệu suất. FileChannel hỗ trợ hai phương thức chuyển dữ liệu được tối ưu hóa cao: transferFrom() và transferTo(), di chuyển dữ liệu giữa kênh tệp và một kênh khác. Các phương thức này có thể tận dụng các bộ đệm trực tiếp nội bộ để di chuyển dữ liệu giữa các kênh một cách nhanh chóng, thường mà không cần sao chép các byte vào không gian bộ nhớ của Java.

Ví dụ sau đây nên là cách nhanh nhất để thực hiện một bản sao tệp trong Java ngoài việc sử dụng phương thức Filescopy() tích hợp sẵn:

```java
import java.nio.channels.*;
import java.nio.file.*;
import static java.nio.file.StandardOpenOption.*;

public class CopyFile {
    public static void main(String[] args) throws Exception {
        FileSystem fs = FileSystems.getDefault();
        Path fromFile = fs.getPath(args[0]);
        Path toFile = fs.getPath(args[1]);
        try (FileChannel in = FileChannel.open(fromFile);
             FileChannel out = FileChannel.open(toFile, CREATE, WRITE)) {
            in.transferTo(0, in.size(), out);
        }
    }
}
```

Trong ví dụ này, chúng ta mở hai FileChannel cho tệp nguồn và tệp đích, sau đó chúng ta sử dụng phương thức transferTo() để chuyển dữ liệu từ kênh nguồn đến kênh đích. Phương thức này di chuyển dữ liệu từ vị trí bắt đầu 0 với độ dài là kích thước của tệp nguồn (in.size()).

Sử dụng transferTo() giữa các FileChannel là cách tốt nhất để sao chép tệp trong Java vì nó tận dụng được tính năng tối ưu hóa hiệu suất của NIO và giảm thiểu việc sao chép dữ liệu qua bộ nhớ của Java, tạo ra hiệu suất cao hơn so với các phương pháp truyền thống.

#### AsynchronousFileChannel

Khi chúng ta quay lại với NIO trong chương tiếp theo, chúng ta sẽ thấy rằng các kênh mạng là loại của SelectableChannel, điều này có nghĩa là chúng có thể được quản lý bằng một selector để kiểm tra khi các kênh sẵn sàng để đọc hoặc ghi và quản lý chúng một cách hiệu quả mà không chặn luồng. File channels không phải là các selectable channel và hầu hết các hoạt động tệp thông thường đơn giản là chặn đến khi chúng được hoàn thành. Điều này không có nghĩa là các hoạt động tệp luôn chặn đến khi tất cả các byte mà chúng ta muốn được đọc từ hoặc được ghi vào đĩa. Nói chung, các hoạt động đọc có thể trả về ít byte hơn được yêu cầu và các hoạt động ghi có thể ghi ít byte hơn và cũng có thể lưu trữ dữ liệu trong bộ nhớ trừ khi chúng ta sử dụng các tùy chọn mở SYNC hoặc DSYNC. Nhưng trong một thế giới nơi truy cập đĩa có thể chậm hơn rất nhiều lần so với các hoạt động trong bộ nhớ, thậm chí các đọc và ghi một phần cũng có thể chậm đến mức chúng ta không muốn chờ đợi cho chúng.

Giải pháp rõ ràng là sử dụng đa luồng và phối hợp các hoạt động đọc và ghi trong một luồng riêng biệt từ logic chính của chúng ta. Java 7 đã làm cho điều này dễ dàng hơn bằng cách giới thiệu AsynchronousFileChannel, đây là một file channel mà tất cả các hoạt động của nó được giao cho một thread pool và có thể báo cáo kết quả bằng cách sử dụng một Future object hoặc gọi lại không đồng bộ. Tất cả các hoạt động đọc và ghi trên các kênh file không đồng bộ phải chỉ định vị trí byte cho hoạt động (vì không có "vị trí" hiện tại xác định trong tệp tại bất kỳ thời điểm nào). Ví dụ đơn giản nhất là ghi một cập nhật tệp trong nền mà không cần thu thập kết quả:

```java
import java.nio.channels.*;
import java.nio.file.*;
import static java.nio.file.StandardOpenOption.*;

public class CopyFile {
    public static void main(String[] args) throws Exception {
        FileSystem fs = FileSystems.getDefault();
        Path fromFile = fs.getPath(args[0]);
        Path toFile = fs.getPath(args[1]);
        try (AsynchronousFileChannel channel = AsynchronousFileChannel.open(fromFile, WRITE)) {
            channel.write(logBuffer, channel.size());
        }
    }
}
```

Ở đây, chúng ta đã tạo một AsynchronousFileChannel tương tự như cách chúng ta mở một file channel thông thường. Việc ghi của chúng ta xảy ra ở nền và phương thức write() trả về ngay lập tức. Theo mặc định, kênh sẽ sử dụng một thread pool mặc định của hệ thống để thực hiện việc ghi của chúng ta ở nền. Một lựa chọn khác là chúng ta có thể cung cấp một Executor service của riêng chúng ta cho thread pool như một đối số cho cuộc gọi open(). Nếu tại một số điểm nào đó chúng ta cần phối hợp và đảm bảo rằng tất cả dữ liệu đã được ghi, chúng ta có thể sử dụng phương thức force() của kênh để chặn đến khi tất cả các hoạt động ghi được hoàn thành.

Một trường hợp thú vị hơn là một hoạt động đọc nơi chúng ta cần các byte được trả về từ hoạt động. Trong trường hợp này, chúng ta có thể cung cấp một CompletionHandler callback object sẽ đẩy kết quả cho chúng ta khi chúng đã sẵn sàng.

```java
AsynchronousFileChannel channel = AsynchronousFileChannel.open(path);
ByteBuffer bbuff = ByteBuffer.allocate(1024);
Object attachment = ...;
channel.read(bbuff, offset, attachment, new CompletionHandler<Integer, Object>() {
    @Override
    public void completed(Integer result, Object attachment) {
        System.out.println("read bytes = " + result);
    }

    @Override
    public void failed(Throwable exc, Object attachment) {
        // Handle failure
    }
});
```

Đối số attachment bổ sung trong cuộc gọi read có thể là bất kỳ đối tượng nào chúng ta muốn và nó được trả về cho chúng ta trong callback như một cách để chúng ta duy trì bất kỳ ngữ cảnh nào cần thiết để phục vụ kết quả. Ở đây, chúng ta in ra số lượng byte đã sẵn sàng, như thông thường có thể ít hơn số byte mà chúng ta yêu cầu, nhưng ít nhất không yêu cầu chúng ta phải chờ đợi cho đến khi chúng. Khả năng khác được minh họa ở đây là hoạt động đọc có thể thất bại, trong trường h

ợp đó phương thức failed() của chúng ta được gọi với ngoại lệ tương ứng.

### Scalable I/O with NIO

Chúng ta đã đặt nền móng cho việc sử dụng gói NIO trong chương này, nhưng bỏ qua một số phần quan trọng. Trong chương tiếp theo, chúng ta sẽ thấy nhiều hơn về động lực thực sự cho java.nio khi nói về I/O không chặn và có thể chọn. Ngoài các tối ưu hóa hiệu suất có thể thực hiện thông qua các buffer trực tiếp, những khả năng này làm cho các thiết kế cho các máy chủ mạng sử dụng ít luồng hơn và có thể mở rộng tốt cho các hệ thống lớn. Trong chương đó, chúng ta sẽ xem xét các loại Channel quan trọng khác: SocketChannel, ServerSocketChannel và DatagramChannel.