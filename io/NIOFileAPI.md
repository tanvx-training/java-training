## The NIO File API

Bây giờ chúng ta sẽ chuyển sự chú ý từ "cổ điển", API Tệp Java gốc sang "mới", API Tệp NIO được giới thiệu với Java 7. Như chúng tôi đã đề cập trước đó, API Tệp NIO có thể được coi là một sự thay thế hoặc bổ sung cho API cổ điển. Bao gồm trong gói NIO, API mới này được gọi là một phần của nỗ lực để di chuyển Java về một phong cách I/O hiệu suất cao và linh hoạt hơn, hỗ trợ kênh có thể chọn và bị gián đoạn bất đồng bộ.

Tuy nhiên, trong ngữ cảnh làm việc với tệp, sức mạnh của API mới này là nó cung cấp một trừu tượng hóa đầy đủ của hệ thống tệp trong Java. Ngoài việc hỗ trợ tốt hơn cho các loại hệ thống tệp hiện có trong thế giới thực - bao gồm lần đầu tiên khả năng sao chép và di chuyển tệp, quản lý liên kết và lấy các thuộc tính tệp chi tiết như chủ sở hữu và quyền hạn - API Tệp mới cho phép triển khai hoàn toàn các loại hệ thống tệp mới trực tiếp trong Java. Ví dụ tốt nhất về điều này là nhà cung cấp hệ thống tệp ZIP mới, cho phép "gắn" một tệp lưu trữ ZIP như một hệ thống tệp và làm việc với các tệp bên trong nó trực tiếp bằng cách sử dụng các API tiêu chuẩn, giống như bất kỳ hệ thống tệp nào khác.

Ngoài ra, gói Tệp NIO cung cấp một số tiện ích mà đã giúp các nhà phát triển Java tiết kiệm được rất nhiều mã lặp lại qua các năm, bao gồm giám sát thay đổi cây thư mục, duyệt hệ thống tệp (một mẫu thiết kế du khách), "globbing" tên tệp và các phương thức tiện ích để đọc toàn bộ các tệp trực tiếp vào bộ nhớ.

Chúng tôi sẽ bao quát API Tệp cơ bản trong phần này và quay lại API NIO một lần nữa ở cuối chương khi chúng tôi bao quát chi tiết đầy đủ về bộ đệm và kênh NIO. Đặc biệt, chúng tôi sẽ nói về ByteChannels và FileChannel, mà bạn có thể coi là các luồng hướng bộ đệm thay thế để đọc và ghi tệp và các loại dữ liệu khác.

### FileSystem and Path

Các thành phần chính trong gói java.nio.file bao gồm: FileSystem, đại diện cho một cơ chế lưu trữ cơ bản và hoạt động như một nhà máy cho các đối tượng Path; Path, đại diện cho một tệp hoặc thư mục trong hệ thống tệp; và tiện ích Files, chứa một bộ các phương thức tĩnh phong phú để thao tác các đối tượng Path để thực hiện tất cả các hoạt động cơ bản trên tệp tương tự như API cổ điển.

Lớp FileSystems (số nhiều) là điểm khởi đầu của chúng ta. Nó là một nhà máy cho đối tượng FileSystem:
```java
// Hệ thống tệp máy tính mặc định
FileSystem fs = FileSystems.getDefault();
// Một hệ thống tệp tùy chỉnh
URI zipURI = URI.create("jar:file:/Users/pat/tmp/MyArchive.zip");
FileSystem zipfs = FileSystems.newFileSystem( zipURI, env ) );
```
Như đã thể hiện trong đoạn mã này, thường chúng ta sẽ đơn giản yêu cầu hệ thống tệp mặc định để thao tác với các tệp trong môi trường máy tính máy chủ, giống như với API cổ điển. Nhưng lớp FileSystems cũng có thể xây dựng một FileSystem bằng cách lấy một URI (một loại định danh đặc biệt) trỏ đến một loại hệ thống tệp tùy chỉnh. Chúng tôi sẽ hiển thị một ví dụ về làm việc với nhà cung cấp hệ thống tệp ZIP sau trong chương này khi chúng tôi thảo luận về nén dữ liệu.

FileSystem triển khai Closeable và khi một FileSystem được đóng, tất cả các kênh tệp mở và các đối tượng luồng khác liên kết với nó cũng sẽ bị đóng. Cố gắng đọc hoặc ghi vào các kênh đó sẽ ném một ngoại lệ tại thời điểm đó. Lưu ý rằng hệ thống tệp mặc định (liên kết với máy tính máy chủ) không thể được đóng.

Sau khi chúng ta có một FileSystem, chúng ta có thể sử dụng nó như một nhà máy cho các đối tượng Path đại diện cho các tệp hoặc thư mục. Một Path có thể được xây dựng bằng cách sử dụng một biểu diễn chuỗi giống như File cổ điển và sau đó được sử dụng với các phương thức của tiện ích Files để tạo, đọc, ghi hoặc xóa mục.

```java
Path fooPath = fs.getPath( "/tmp/foo.txt" );
OutputStream out = Files.newOutputStream( fooPath );
```

Ví dụ này mở một OutputStream để ghi vào tệp foo.txt. Mặc định, nếu tệp không tồn tại, nó sẽ được tạo ra và nếu nó tồn tại, nó sẽ được cắt ngắn (đặt về độ dài bằng không) trước khi dữ liệu mới được ghi - nhưng bạn có thể thay đổi các kết quả này bằng cách sử dụng các tùy chọn. Chúng tôi sẽ nói thêm về các phương thức Files trong phần tiếp theo.

Đối tượng Path triển khai giao diện java.lang.Iterable, có thể được sử dụng để lặp qua các thành phần đường dẫn chữ (ví dụ, các thành phần được phân tách bằng dấu gạch chéo "tmp" và "foo.txt" trong đoạn mã trước). Tuy nhiên, nếu bạn muốn duyệt qua đường dẫn để tìm các tệp hoặc thư mục khác, bạn có thể quan tâm hơn đến DirectoryStream và FileVisitor mà chúng tôi sẽ thảo luận sau. Path cũng triển khai giao diện java.nio.file.Watchable, cho phép nó được giám sát các thay đổi. Chúng tôi cũng sẽ thảo luận về việc giám sát cây thư mục tệp cho các thay đổi trong phần sắp tới.

Path có các phương thức tiện ích để giải quyết các đường dẫn tương đối đối với một tệp hoặc thư mục.
```java
Path patPath = fs.getPath( "/User/pat/" );
Path patTmp = patPath.resolve("tmp" ); // "/User/pat/tmp"
// Tương tự như trên, sử dụng Path
Path tmpPath = fs.getPath( "tmp" );
Path patTmp = patPath.resolve( tmpPath ); // "/User/pat/tmp"
// Giải quyết một đ

ường dẫn tuyệt đối cho bất kỳ đường dẫn nào chỉ cho ra đường dẫn đã cho
Path absPath = patPath.resolve( "/tmp" ); // "/tmp"
// Giải quyết anh em của Pat (cùng cha mẹ)
Path danPath = patPath.resolveSibling( "dan" ); // "/Users/dan"
```
Trong đoạn mã này, chúng tôi đã hiển thị các phương thức resolve() và resolveSibling() được sử dụng để tìm các tệp hoặc thư mục tương đối với một đối tượng Path đã cho. Phương thức resolve() thường được sử dụng để thêm một đường dẫn tương đối vào một Path đã tồn tại đại diện cho một thư mục. Nếu đối số được cung cấp cho phương thức resolve() là một đường dẫn tuyệt đối, nó sẽ chỉ trả về đường dẫn tuyệt đối (nó hoạt động giống như lệnh "cd" Unix hoặc DOS). Phương thức resolveSibling() hoạt động theo cách tương tự, nhưng nó tương đối với thư mục cha của Path mục tiêu; phương thức này hữu ích để mô tả mục tiêu của một hoạt động di chuyển().

#### Path to classic file and back

Để kết nối giữa các API cũ và mới, các phương thức tương ứng toPath() và toFile() đã được cung cấp trong java.io.File và java.nio.file.Path, lần lượt, để chuyển đổi sang dạng khác. Tất nhiên, chỉ có các loại Paths có thể được tạo ra từ File là các đường dẫn đại diện cho các tệp và thư mục trong hệ thống tệp mặc định của máy chủ.

```java
Path tmpPath = fs.getPath( "/tmp" );
File file = tmpPath.toFile(); // Chuyển đổi Path sang File

File tmpFile = new File( "/tmp" );
Path path = tmpFile.toPath(); // Chuyển đổi File sang Path
```

Như trong ví dụ trên, bạn có thể chuyển đổi giữa đối tượng Path và File một cách linh hoạt sử dụng các phương thức toPath() và toFile() tương ứng.

### NIO File Operations

Sau khi chúng ta có một Path, chúng ta có thể thực hiện các thao tác trên nó bằng các phương thức tĩnh của tiện ích Files để tạo đường dẫn dưới dạng tệp hoặc thư mục, đọc và ghi vào nó, và thăm dò và thiết lập các thuộc tính của nó. Chúng ta sẽ liệt kê hầu hết chúng và sau đó thảo luận về một số phương thức quan trọng hơn khi chúng ta tiếp tục.
Bảng dưới đây tóm tắt các phương thức này của lớp java.nio.file.Files. Như bạn có thể mong đợi, vì lớp Files xử lý tất cả các loại hoạt động trên tệp, nó chứa một số lượng lớn các phương thức. Để làm cho bảng dễ đọc hơn, chúng tôi đã bỏ qua các biến thể của cùng một phương thức (những phương thức nhận các loại đối số khác nhau) và nhóm các loại phương thức tương ứng và liên quan lại với nhau.

Dưới đây là bảng tóm tắt các phương thức của lớp `java.nio.file.Files`:

| Phương thức          | Kiểu trả về      | Mô tả                                                                                         |
|----------------------|------------------|-----------------------------------------------------------------------------------------------|
| copy()               | long hoặc Path  | Sao chép một luồng dữ liệu vào một đường dẫn tệp, đường dẫn tệp thành luồng dữ liệu, hoặc từ đường dẫn này sang đường dẫn khác. Trả về số byte đã sao chép hoặc đường dẫn đích. Có thể thay thế tệp đích nếu nó đã tồn tại (mặc định là không thể thay thế nếu tệp đích tồn tại). Sao chép một thư mục sẽ tạo ra một thư mục trống ở đích (nội dung không được sao chép). Sao chép một liên kết tượng trưng sẽ sao chép dữ liệu tệp được liên kết (tạo ra một bản sao tệp thông thường). |
| createDirectory(), createDirectories() | Path           | Tạo một thư mục duy nhất hoặc tất cả các thư mục trong một đường dẫn cụ thể. Phương thức createDirectory() ném một ngoại lệ nếu thư mục đã tồn tại, trong khi createDirectories() sẽ bỏ qua các thư mục đã tồn tại và chỉ tạo mới khi cần thiết. |
| createFile()         | Path             | Tạo một tệp trống. Thao tác này là nguyên tử và chỉ thành công nếu tệp không tồn tại. (Thuộc tính này có thể được sử dụng để tạo các tệp cờ bảo vệ tài nguyên, v.v.)  |
| createTempDirectory(), createTempFile() | Path         | Tạo một thư mục hoặc tệp tạm thời, đảm bảo, có tên duy nhất với tiền tố được chỉ định. Tùy chọn đặt nó trong thư mục tạm thời mặc định của hệ thống. |
| delete(), deleteIfExists() | void           | Xóa một tệp hoặc một thư mục trống. deleteIfExists() sẽ không ném ra ngoại lệ nếu tệp không tồn tại.   |
| exists(), notExists()  | boolean          | Xác định xem tệp tồn tại (notExists() đơn giản trả về ngược lại). Tùy chọn chỉ định liệu các liên kết nên được theo dõi hay không (mặc định là có). |
| exists(), isDirectory(), isExecutable(), isHidden(), isReadable(), isRegularFile(), isWritable() | boolean | Kiểm tra các tính năng cơ bản của tệp: xem xét xem đường dẫn có tồn tại, có phải là thư mục không và các thuộc tính cơ bản khác. |
| createLink(), createSymbolicLink(), isSymbolicLink(), readSymbolicLink(), createLink() | boolean hoặc Path | Tạo một liên kết cứng hoặc liên kết tượng trưng, kiểm tra xem một tệp có phải là một liên kết tượng trưng hay không, hoặc đọc tệp mục tiêu được chỉ vào bởi liên kết tượng trưng. Liên kết tượng trưng là các tệp tham chiếu đến các tệp khác. Các liên kết thông thường ("cứng") là các bản sao gần cấp thấp của một tệp nơi hai tên tệp chỉ vào cùng một dữ liệu cơ bản. Nếu bạn không biết nên sử dụng cái nào, hãy sử dụng liên kết tượng trưng. |
| getAttribute(), setAttribute(), getFileAttributeView(), readAttributes() | Object, Map, hoặc FileAttributeView | Lấy hoặc đặt các thuộc tính tệp cụ thể của hệ thống tệp như thời gian truy cập và cập nhật, quyền chi tiết và thông tin chủ sở hữu bằng các tên cụ thể của việc triển khai. |
| getFileStore()        | FileStore        | Lấy một đối tượng FileStore đại diện cho thiết bị, ổ đĩa hoặc loại phân vùng của hệ thống tệp mà đường dẫn đó đặt trên đó. |
| getLastModifiedTime(), setLastModifiedTime() | FileTime hoặc Path | Lấy hoặc đặt thời gian sửa đổi cuối cùng của một tệp hoặc thư mục.   |
| getOwner(), setOwner() | UserPrincipal   | Lấy hoặc đặt một đối tượng UserPrincipal đại diện cho chủ sở hữu của tệp. Sử dụng toString() hoặc getName() để lấy biểu diễn chuỗi của tên người dùng. |
| getPosixFilePermissions(), setPosixFilePermissions() | Set hoặc Path | Lấy hoặc đặt toàn bộ quyền đọc và ghi POSIX theo kiểu người-nhóm-khác đầy đủ cho đường dẫn dưới dạng một Set của các giá trị PosixFilePermission enum. |
| isSameFile()          | boolean          | Kiểm tra xem hai đường dẫn có tham chiếu đến cùng một tệp hay không (điều này có thể là đúng ngay cả khi các đường dẫn không giống nhau). |
| move()               | Path             | Di chuyển một tệp hoặc thư mục bằng cách đổi tên hoặc sao chép nó, tùy chọn chỉ định xem có thay thế bất kỳ mục tiêu hiện có nào hay không. Việc đổi tên sẽ được sử dụng trừ khi cần sao chép để di chuyển một tệp qua các cửa hàng tệp hoặc hệ thống tệp khác nhau. Chỉ có thể di chuyển các thư mục bằng cách sử dụng phương thức này nếu việc đổi tên đơn giản hoặc nếu thư mục đó trống. Nếu việc di chuyển thư mục yêu cầu sao chép các tệp qua các cửa hàng tệp hoặc hệ thống tệp khác nhau, phương thức sẽ ném ra một IOException. (Trong trường hợp này, bạn phải tự sao chép các tệp của mình. Xem walkFileTree().) |
| newBufferedReader(), newBufferedWriter() | BufferedReader hoặc BufferedWriter | Mở một tệp để đọc thông qua một BufferedReader hoặc tạo và mở một tệp để ghi thông qua BufferedWriter. Trong cả hai trường hợp, một bảng mã ký tự được chỉ định. |
| newByteChannel()     | SeekableByteChannel | Tạo một tệp mới hoặc mở một tệp hiện có dưới dạng một kênh byte có thể tìm kiếm. (Xem thảo luận đầy đủ về NIO sau trong chương này.) Cân nhắc sử dụng FileChannel.open() như một lựa chọn thay thế. |
| newDirectoryStream() | DirectoryStream  | Trả về một DirectoryStream để lặp qua cấu trúc thư mục chỉ định. Tùy chọn, cung cấp một mẫu glob hoặc đối tượng bộ lọc để phù hợp với các tệp. |
| newInputStream(), newOutputStream() | InputStream hoặc OutputStream | Mở một tệp để đọc thông qua InputStream hoặc tạo và mở một tệp để ghi thông qua OutputStream. Tùy chọn, chỉ định việc cắt tệp cho dòng đầu ra; mặc định là tạo một cắt ngắn khi ghi. |
| probeContentType()   | String           | Trả về loại MIME của tệp nếu có thể xác định bằng các dịch vụ Detector Kiểu Tệp đã cài đặt hoặc null nếu không biết. |
| readAllBytes(), readAllLines() | byte[] hoặc List<String> | Đọc tất cả dữ liệu từ tệp dưới dạng mảng byte[] hoặc tất cả các ký tự dưới dạng danh sách chuỗi sử dụng một bảng mã ký tự đã chỉ định. |
| size()               | long             | Lấy kích thước trong byte của tệp tại đường dẫn cụ thể.   |
| walkFileTree()       | Path             | Áp dụng một FileVisitor vào cấu trúc thư mục được chỉ định, tùy chọn chỉ định xem có theo dõi các liên kết và một độ sâu tối đa của việc đi qua. |
| write()              | Path             | Ghi một mảng byte hoặc một bộ sưu tập chuỗi (với một bảng mã ký tự đã chỉ định) vào tệp tại đường dẫn cụ thể và đóng tệp, tùy chọn chỉ định hành vi gắn thêm và cắt. Mặc định là cắt và viết dữ liệu.   |

Các phương thức trước đó cho phép chúng ta lấy các luồng nhập hoặc xuất hoặc các bộ đọc và bộ ghi được đệm cho một tệp đã cho. Chúng ta cũng có thể tạo đường dẫn dưới dạng các tệp và thư mục và lặp qua các cấu trúc thư mục. Chúng tôi sẽ thảo luận về các hoạt động thư mục trong phần tiếp theo.

Như một lời nhắc, các phương thức resolve() và resolveSibling() của Path rất hữu ích để xây dựng các mục tiêu cho các hoạt động sao chép() và move().
```java
// Di chuyển tệp /tmp/foo.txt đến /tmp/bar.txt
Path foo = fs.getPath("/tmp/foo.txt" );
Files.move( foo, foo.resolveSibling("bar.txt") );
```
Để nhanh chóng đọc và ghi nội dung của các tệp mà không cần luồng truyền, chúng ta có thể sử dụng các phương thức đọc tất cả và ghi đồng thời mảng byte hoặc chuỗi vào và ra khỏi các tệp trong một hoạt động duy nhất. Điều này rất tiện lợi cho các tệp dễ dàng phù hợp vào bộ nhớ.

```java
// Đọc và ghi bộ sưu tập của String (ví dụ: các dòng văn bản)
Charset asciiCharset = Charset.forName("US-ASCII");
List<String> csvData = Files.readAllLines( csvPath, asciiCharset );
Files.write( newCSVPath, csvData, asciiCharset );

// Đọc và ghi bytes
byte [] data = Files.readAllBytes( dataPath );
Files.write( newDataPath, data );
```


### Directory Operations

Ngoài các phương thức cơ bản để tạo và điều chỉnh thư mục của lớp Files, còn có các phương thức để liệt kê các tệp trong một thư mục đã cho và duyệt qua tất cả các tệp và thư mục trong một cây thư mục. Để liệt kê các tệp trong một thư mục duy nhất, chúng ta có thể sử dụng một trong các phương thức newDirectoryStream(), nó trả về một DirectoryStream có thể lặp qua.
```java
// In ấn các tệp và thư mục trong /tmp
try ( DirectoryStream<Path> paths = Files.newDirectoryStream(
 fs.getPath( "/tmp" ) ) ) {
 for ( Path path : paths ) { System.out.println( path ); }
}
```
Đoạn mã liệt kê các mục trong "/tmp," lặp qua luồng thư mục để in ra kết quả. Lưu ý rằng chúng ta mở DirectoryStream bên trong một cụm try-with-resources để nó tự động đóng cho chúng ta. Một DirectoryStream được triển khai như một loại có thể lặp qua một chiều tương tự như một luồng, và nó phải được đóng để giải phóng tài nguyên liên kết. Thứ tự mà các mục được trả về không được xác định bởi API và bạn có thể cần lưu trữ và sắp xếp chúng nếu cần thiết.

Một dạng khác của newDirectoryStream() nhận một mẫu glob để giới hạn các tệp được khớp trong danh sách:
```java
// Chỉ các tệp trong /tmp khớp với "*.txt" (globbing)
try ( DirectoryStream<Path> paths = Files.newDirectoryStream(
 fs.getPath( "/tmp" ), "*.txt" ) ) {
 ...
```
Bộ lọc globbing tên tệp bằng cách sử dụng các mẫu quen thuộc “*” và một số mẫu khác để chỉ định các tên khớp. Bảng dưới đây cung cấp một số ví dụ bổ sung về các mẫu globbing tệp. Nếu các mẫu globbing không đủ, chúng ta có thể cung cấp bộ lọc dòng của riêng mình bằng cách triển khai giao diện DirectoryStream.Filter. Đoạn mã sau đây là phiên bản thủ tục (code) của mẫu glob “*.txt”; các tên tệp khớp kết thúc bằng “.txt”. Chúng tôi đã triển khai bộ lọc dưới dạng một lớp nội tại ẩn danh ở đây vì nó ngắn:
```java
// Tương tự như trên sử dụng bộ lọc của chúng tôi (ẩn danh)
try ( DirectoryStream<Path> paths = Files.newDirectoryStream(
 fs.getPath( "/tmp" ),
 new DirectoryStream.Filter<Path>() {
 @Override
 public boolean accept( Path entry ) throws IOException {
 return entry.toString().endsWith( ".txt" );
 }
} ) ) {
 ...
```
Cuối cùng, nếu chúng ta cần lặp qua toàn bộ cây thư mục thay vì chỉ một thư mục duy nhất, chúng ta có thể sử dụng một FileVisitor. Phương thức Files.walkFileTree() nhận một đường dẫn bắt đầu và thực hiện một quá trình duyệt theo chiều sâu của cấu trúc thư mục, cho phép FileVisitor được cung cấp một cơ hội để “thăm” từng phần tử đường dẫn trong cây. Đoạn mã ngắn dưới đây in tất cả tên tệp và thư mục trong đường dẫn /Users/pat:
```java
// Thăm tất cả các tệp trong một cây thư mục
Files.walkFileTree( fs.getPath( "/Users/pat"), new SimpleFileVisitor<Path>() {@Override
 public FileVisitResult visitFile( Path file, BasicFileAttributes attrs )
 {
 System.out.println( "path = " + file );
 return FileVisitResult.CONTINUE;
 }
} );
```
Đối với mỗi mục trong cây tệp, phương thức visitFile() của trình duyệt của chúng tôi được gọi với phần tử Path và các thuộc tính làm đối số. Trình duyệt có

thể thực hiện bất kỳ hành động nào liên quan đến tệp và sau đó chỉ định liệu việc duyệt nên tiếp tục hay không bằng cách trả về một trong các loại kết quả được định nghĩa: FileVisitResult.CONTINUE hoặc TERMINATE. Ở đây, chúng tôi đã kế thừa lớp SimpleFileVisitor, đây là một lớp tiện ích mà thực hiện các phương thức của giao diện FileVisitor cho chúng ta với các phần thân trống (no-op), cho phép chúng ta ghi đè chỉ những phần quan trọng. Các phương thức khác bao gồm visitFileFailed(), được gọi nếu một tệp hoặc thư mục không thể được thăm (ví dụ: do quyền hạn), và cặp preVisitDirectory() và postVisitDirectory(), có thể được sử dụng để thực hiện hành động trước và sau khi một thư mục mới được thăm. Phương thức preVisitDirectory() có tính hữu ích bổ sung trong việc nó được phép trả về giá trị SKIP_SUBTREE để tiếp tục việc duyệt mà không đi sâu vào đường dẫn mục tiêu và giá trị SKIP_SIBLINGS, chỉ ra rằng duyệt nên tiếp tục, bỏ qua các mục còn lại cùng cấp với đường dẫn mục tiêu.

Như bạn có thể thấy, các phương thức liệt kê và duyệt qua tệp của gói NIO File cơ bản là phức tạp hơn nhiều so với của API java.io cổ điển và là một bổ sung đáng chào đón.

### Watching Paths

Một trong những tính năng tốt nhất của NIO File API là WatchService, có thể giám sát một Path để theo dõi các thay đổi đối với bất kỳ tệp hoặc thư mục nào trong cấu trúc cây thư mục. Chúng ta có thể chọn nhận các sự kiện khi các tệp hoặc thư mục được thêm, sửa đổi hoặc xóa. Đoạn mã sau đây giám sát các thay đổi trong thư mục /Users/pat:

```java
Path watchPath = fs.getPath("/Users/pat");
WatchService watchService = fs.newWatchService();
watchPath.register( watchService, ENTRY_CREATE, ENTRY_MODIFY, ENTRY_DELETE );

while( true )
{
    WatchKey changeKey = watchService.take();
    List<WatchEvent<?>> watchEvents = changeKey.pollEvents();
    for ( WatchEvent<?> watchEvent : watchEvents )
    {
        // Các sự kiện của chúng tôi đều là loại Path:
        WatchEvent<Path> pathEvent = (WatchEvent<Path>)watchEvent;
        Path path = pathEvent.context();
        WatchEvent.Kind<Path> eventKind = pathEvent.kind();
        System.out.println( eventKind + " for path: " + path );
    }
    changeKey.reset(); // Quan trọng!
}
```

Chúng ta tạo một WatchService từ một FileSystem bằng cách sử dụng cuộc gọi newWatchService(). Sau đó, chúng ta có thể đăng ký một đối tượng Watchable với dịch vụ (hiện tại, Path là loại duy nhất của Watchable) và kiểm tra các sự kiện. Như được hiển thị, thực tế API là ngược lại và chúng ta gọi phương thức đăng ký của đối tượng Watchable, truyền cho nó dịch vụ theo dõi và một danh sách đối số biến đổi của các giá trị được liệt kê đại diện cho các loại sự kiện quan tâm: ENTRY_CREATE, ENTRY_MODIFY hoặc ENTRY_DELETE. Một loại bổ sung, OVERFLOW, có thể được đăng ký để nhận các sự kiện cho biết khi triển khai máy chủ đã quá chậm chạp để xử lý tất cả các thay đổi và một số thay đổi có thể đã bị mất.

Sau khi đã cài đặt, chúng ta có thể kiểm tra các thay đổi bằng cách sử dụng phương thức take() của dịch vụ theo dõi, phương thức này trả về một đối tượng WatchKey. Phương thức take() chặn cho đến khi một sự kiện xảy ra; một biến thể khác, poll(), không chặn. Khi chúng ta có một WatchKey chứa các sự kiện, chúng ta có thể lấy chúng bằng phương thức pollEvents(). API lại một lần nữa hơi cồng kềnh ở đây vì WatchEvent là một loại chung được tham số hóa theo loại Watchable đối tượng. Trong trường hợp của chúng tôi, chỉ có thể là các sự kiện loại Path và do đó chúng ta đúng kiểu cần thiết. Loại sự kiện (tạo, sửa đổi, xóa) được chỉ định bởi phương thức kind() của WatchEvent và đường dẫn được thay đổi được chỉ định bởi phương thức context(). Cuối cùng, quan trọng là chúng ta gọi reset() trên đối tượng WatchKey để xóa các sự kiện và có thể nhận các cập nhật tiếp theo.

Hiệu suất của WatchService phụ thuộc nhiều vào việc triển khai. Trên nhiều hệ thống, việc giám sát hệ thống tệp được tích hợp vào hệ điều hành và chúng ta có thể nhận được các sự kiện thay đổi gần như ngay lập tức. Nhưng trong nhiều trường hợp, Java có thể chuyển sang triển khai chung, dựa trên luồng nền của dịch vụ theo dõi, mà rất chậm trong việc phát hiện các thay đổi. Ví dụ, vào thời điểm viết bài này, Java 7 trên Mac OS X không tận dụng việc giám sát tệp cấp độ hệ điều hành và thay vào đó sử dụng dịch vụ theo dõi polling chậm chạp.