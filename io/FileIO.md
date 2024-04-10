## File I/O

Trong chương này, chúng ta sẽ nói về API Java file I/O. Để chính xác hơn, chúng ta sẽ nói về hai API file: trước hết, có java.io File I/O core là một phần của Java từ đầu. Sau đó là "new" java.nio.file API được giới thiệu trong Java 7. Nói chung, các gói NIO, mà chúng ta sẽ tìm hiểu chi tiết sau và mà không chỉ đụng vào các tệp mà còn vào tất cả các loại I/O mạng và kênh, đã được giới thiệu để thêm các tính năng tiên tiến làm cho Java có khả năng mở rộng và hiệu suất cao hơn. Tuy nhiên, trong trường hợp của file NIO, gói mới cũng chỉ là một phần của việc "làm lại" API ban đầu. Trong các bộ phim, bạn có thể coi hai API như là "cổ điển" và "làm lại" của loạt phim. API mới hoàn toàn sao chép chức năng của API gốc, nhưng vì API core quá cơ bản (và trong một số trường hợp đơn giản hơn), có lẽ nhiều người sẽ ưa thích tiếp tục sử dụng nó. Chúng ta sẽ bắt đầu với API cổ điển tập trung vào java.io.File và sau này chúng ta sẽ tìm hiểu về API mới, tập trung vào java.nio.Path tương tự.

Làm việc với tệp trong Java là dễ dàng, nhưng đặt ra một số vấn đề về khái niệm. Hệ thống tệp thực tế có thể khác nhau rộng rãi về kiến trúc và triển khai: hãy nghĩ đến sự khác biệt giữa các hệ thống Mac, PC và Unix khi nói đến tên tệp. Java cố gắng che giấu một số khác biệt này và cung cấp thông tin để giúp ứng dụng điều chỉnh mình cho môi trường địa phương, nhưng nó để lại rất nhiều chi tiết về triển khai truy cập tệp phụ thuộc vào. Chúng ta sẽ nói về các kỹ thuật xử lý vấn đề này khi điều tra.

Trước khi chúng ta rời khỏi File I/O, chúng ta cũng sẽ chỉ cho bạn một số công cụ cho trường hợp đặc biệt của các tệp ứng dụng "resource" được đóng gói với ứng dụng của bạn và được tải thông qua classpath Java.

### The java.io.File Class
Lớp java.io.File đóng gói quyền truy cập thông tin về một tệp hoặc thư mục. Nó có thể được sử dụng để lấy thông tin thuộc tính về một tệp, liệt kê các mục trong một thư mục và thực hiện các hoạt động cơ bản trên hệ thống tệp, chẳng hạn như xóa một tệp hoặc tạo một thư mục. Trong khi đối tượng File xử lý những hoạt động "meta" này, nó không cung cấp API cho việc đọc và ghi dữ liệu tệp; có các luồng tệp cho mục đích đó.

#### File constructors

Bạn có thể tạo một thể hiện của lớp File từ một chuỗi đường dẫn (pathname) như sau:
```java
File fooFile = new File("/tmp/foo.txt");
File barDir = new File("/tmp/bar");
```
Bạn cũng có thể tạo một tệp với một đường dẫn tương đối:
```java
File f = new File("foo");
```
Trong trường hợp này, Java hoạt động dựa trên "thư mục làm việc hiện tại" (current working directory) của trình thông dịch Java. Bạn có thể xác định thư mục làm việc hiện tại bằng cách đọc thuộc tính user.dir trong danh sách System Properties:
```java
System.getProperty("user.dir"); // ví dụ: "/Users/pat"
```
Một phiên bản nạp chồng của constructor của File cho phép bạn chỉ định đường dẫn thư mục và tên tệp như là các đối tượng String riêng biệt:
```java
File fooFile = new File("/tmp", "foo.txt");
```
Với một biến thể khác, bạn có thể chỉ định thư mục bằng một đối tượng File và tên tệp bằng một chuỗi:
```java
File tmpDir = new File("/tmp"); // File cho thư mục /tmp
File fooFile = new File(tmpDir, "foo.txt");
```
Không có bất kỳ constructor nào của File thực sự tạo ra một tệp hoặc thư mục, và việc tạo một đối tượng File cho một tệp không tồn tại không phải là một lỗi. Đối tượng File chỉ là một con trỏ cho một tệp hoặc thư mục mà bạn có thể muốn đọc, ghi hoặc kiểm tra các thuộc tính. Ví dụ, bạn có thể sử dụng phương thức exists() để tìm hiểu xem tệp hoặc thư mục có tồn tại không.

#### Path localization

Một vấn đề khi làm việc với các tệp trong Java là các đường dẫn được mong đợi tuân thủ các quy ước của hệ thống tệp địa phương. Hai điểm khác biệt là hệ thống tệp Windows sử dụng "gốc" hoặc các ký tự ổ đĩa (ví dụ, C:) và ký tự gạch chéo ngược (\) thay vì dấu gạch chéo xuống (/) được sử dụng trong các hệ thống khác.

Java cố gắng bù đắp cho các khác biệt này. Ví dụ, trên các nền tảng Windows, Java chấp nhận các đường dẫn có cả dấu gạch chéo xuống hoặc gạch chéo ngược. (Tuy nhiên, trên các nền tảng khác, nó chỉ chấp nhận dấu gạch chéo xuống.)

Cách tốt nhất là đảm bảo bạn tuân theo các quy ước về tên tệp của hệ thống tệp chủ. Nếu ứng dụng của bạn có một giao diện đồ họa người dùng (GUI) mở và lưu các tệp theo yêu cầu của người dùng, bạn nên có thể xử lý chức năng đó bằng lớp Swing JFileChooser. Lớp này đóng gói một hộp thoại lựa chọn tệp đồ họa. Các phương thức của JFileChooser xử lý các tính năng tên tệp phụ thuộc vào hệ thống cho bạn.

Nếu ứng dụng của bạn cần xử lý các tệp cho riêng mình, thì mọi thứ sẽ phức tạp hơn một chút. Lớp File chứa một số biến tĩnh để làm cho nhiệm vụ này có thể. File.separator định nghĩa một chuỗi mà xác định ký tự phân tách tệp trên máy chủ địa phương (ví dụ, / trên các hệ thống Unix và Macintosh và \ trên các hệ thống Windows); File.separatorChar cung cấp cùng thông tin nhưng là dưới dạng char.

Bạn có thể sử dụng thông tin phụ thuộc vào hệ thống này một số cách. Có lẽ cách đơn giản nhất để địa phương hóa các đường dẫn là chọn một quy ước mà bạn sử dụng bên trong, chẳng hạn như dấu gạch chéo xuống (/), và thực hiện một thay thế chuỗi để thay thế cho ký tự phân tách địa phương:
```java
// chúng tôi sẽ sử dụng gạch chéo xuống làm tiêu chuẩn
String path = "mail/2004/june/merle";
path = path.replace('/', File.separatorChar);
File mailbox = new File(path);
```
Hoặc bạn có thể làm việc với các thành phần của một đường dẫn và xây dựng đường dẫn địa phương khi bạn cần:
```java
String[] path = { "mail", "2004", "june", "merle" };
StringBuffer sb = new StringBuffer(path[0]);
for (int i = 1; i < path.length; i++)
    sb.append(File.separator + path[i]);
File mailbox = new File(sb.toString());
```
Một điều cần nhớ là Java giải thích ký tự gạch chéo ngược (\) trong mã nguồn như một ký tự thoát khi được sử dụng trong một String. Để có được một dấu gạch chéo ngược trong một String, bạn phải sử dụng \\.

Để xử lý vấn đề với các hệ thống tệp có nhiều "gốc" (ví dụ, C:\ trên Windows), lớp File cung cấp phương thức tĩnh listRoots(), trả về một mảng các đối tượng File tương ứng với các thư mục gốc của hệ thống tệp. Một lần nữa, trong một ứng dụng GUI, một hộp thoại chọn tệp đồ họa bảo vệ bạn khỏi vấn đề này hoàn toàn.

#### File operations

Khi chúng ta có một đối tượng File, chúng ta có thể sử dụng nó để yêu cầu thông tin và thực hiện các hoạt động tiêu chuẩn trên tệp hoặc thư mục mà nó đại diện. Một số phương thức cho phép chúng ta đặt câu hỏi về File. Ví dụ, isFile() trả về true nếu File đại diện cho một tệp thông thường, trong khi isDirectory() trả về true nếu đó là một thư mục. isAbsolute() cho biết liệu File đó có đóng gói một đường dẫn tuyệt đối hay một đặc tả đường dẫn tương đối. Một đường dẫn tuyệt đối là một khái niệm phụ thuộc vào hệ thống, có nghĩa là đường dẫn không phụ thuộc vào thư mục làm việc của ứng dụng hoặc bất kỳ khái niệm về gốc làm việc hoặc ổ đĩa nào (ví dụ, trong Windows, nó là một đường dẫn đầy đủ bao gồm chữ cái ổ đĩa: c:\\Users\pat\foo.txt).

Các thành phần của đường dẫn File có sẵn thông qua các phương thức sau: getName(), getPath(), getAbsolutePath() và getParent(). getName() trả về một chuỗi cho tên tệp mà không có thông tin thư mục. Nếu File có một đặc tả đường dẫn tuyệt đối, getAbsolutePath() trả về đường dẫn đó. Ngược lại, nó trả về đường dẫn tương đối được nối với thư mục làm việc hiện tại (cố gắng làm cho nó trở thành một đường dẫn tuyệt đối). getParent() trả về thư mục cha của tệp hoặc thư mục. Chuỗi được trả về bởi getPath() hoặc getAbsolutePath() có thể không tuân theo các quy ước về chữ hoa/thường của hệ thống tệp dưới lying. Bạn có thể lấy đường dẫn tệp hệ thống của riêng mình hoặc "can‐ onical" bằng cách sử dụng phương thức getCanonicalPath(). Trong Windows, ví dụ, bạn có thể tạo một đối tượng File mà getAbsolutePath() của nó là C:\Autoex‐ ec.bat nhưng getCanonicalPath() của nó là C:\AUTOEXEC.BAT; cả hai đều thực sự chỉ đến cùng một tệp. Điều này hữu ích để so sánh tên tệp có thể đã được cung cấp với các quy ước chữ hoa/không hoa khác nhau hoặc để hiển thị chúng cho người dùng.

Bạn có thể lấy hoặc đặt thời gian sửa đổi của một tệp hoặc thư mục bằng các phương thức lastModified() và setLastModified(). Giá trị là một số long là số mili giây kể từ thời điểm epoch (1 tháng 1, 1970, 00:00:00 GMT). Chúng ta cũng có thể lấy kích thước của tệp trong byte bằng phương thức length().

Dưới đây là một đoạn mã mẫu in một số thông tin về một tệp:
```java
File fooFile = new File("/tmp/boofa");
String type = fooFile.isFile() ? "File " : "Directory ";
String name = fooFile.getName();
long len = fooFile.length();
System.out.println(type + name + ", " + len + " bytes ");
```
Nếu đối tượng File tương ứng với một thư mục không tồn tại, chúng ta có thể tạo thư mục đó bằng mkdir() hoặc mkdirs(). Phương thức mkdir() tạo ra nhiều nhất một cấp thư mục, vì vậy bất kỳ thư mục trung gian nào trong đường dẫn phải tồn tại trước. mkdirs() tạo ra tất cả các cấp thư mục cần thiết để tạo ra đường dẫn đầy đủ của đặc tả File. Trong cả hai trường hợp, nếu thư mục không thể được tạo ra, phương thức sẽ trả về false. Sử dụng renameTo() để đổi tên một tệp hoặc thư mục và delete() để xóa một tệp hoặc thư mục.

Mặc dù chúng ta có thể tạo một thư mục bằng đối tượng File, điều này không phải là cách phổ biến nhất để tạo một tệp; điều đó thường được thực hiện một cách ngầm định khi chúng ta dự định ghi dữ liệu vào đó bằng FileOutputStream hoặc FileWriter, như chúng ta sẽ thảo luận trong một khoảnh khắc. Ngoại lệ là phương thức createNewFile(), có thể được sử dụng để thử tạo ra một tệp mới với độ dài là không. Phương thức này có ích vì hoạt động được đảm bảo là "nguyên tử" đối với tất cả các tệp khác trong hệ thống tệp. createNewFile() trả về một giá trị Boolean cho biết liệu tệp đã được tạo ra hay không. Điều này đôi khi được sử dụng như một tính năng khóa nguyên thủy - người tạo ra tệp đầu tiên "thắng". (Gói NIO hỗ trợ các khóa tệp thực sự, như chúng ta sẽ thấy sau này.) Điều này hữu ích khi kết hợp với deleteOnExit(), làm đánh dấu tệp sẽ tự động bị xóa khi Java VM thoát. Sự kết hợp này cho phép bạn bảo vệ các tài nguyên hoặc tạo một ứng dụng chỉ có thể chạy trong một phiên bản duy nhất vào một thời điểm. Một phương thức tạo tệp khác liên quan đến lớp File chính là phương thức tĩnh createTempFile(), tạo ra một tệp trong một vị trí cụ thể bằng cách sử dụng một tên duy nhất được tạo tự động. Điều này cũng hữu ích khi kết hợp với deleteOnExit().

Phương thức toURL() chuyển đổi một đường dẫn tệp thành một đối tượng URL của tệp. URL là một trừu tượng cho phép bạn chỉ đến bất kỳ loại đối tượng nào ở bất kỳ đâu trên Mạng. Chuyển đổi một tham chiếu File thành URL có thể hữu ích để đảm bảo tính nhất quán với các tiện ích tổng quát hơn mà xử lý URL. Xem Chương 14 để biết chi tiết. URL của tệp cũng trở nên phổ biến hơn với NIO File API nơi chúng có thể được sử dụng để tham chiếu các loại hệ thống tệp mới được triển khai trực tiếp trong mã Java.

Phương thức       | Kiểu trả về | Mô tả
-------------------|-------------|---------------------------------------------
canExecute()       | Boolean     | Tệp có thể thực thi được không?
canRead()          | Boolean     | Tệp (hoặc thư mục) có thể đọc được không?
canWrite()         | Boolean     | Tệp (hoặc thư mục) có thể ghi được không?
createNewFile()    | Boolean     | Tạo một tệp mới.
createTempFile(String pfx, String sfx) | File | Phương thức tĩnh để tạo một tệp mới, với tiền tố và hậu tố được chỉ định, trong thư mục tệp tạm mặc định.
delete()           | Boolean     | Xóa tệp (hoặc thư mục).
deleteOnExit()     | Void        | Khi thoát, hệ thống chạy Java sẽ xóa tệp.
exists()           | Boolean     | Tệp (hoặc thư mục) có tồn tại không?
getAbsolutePath() | String      | Trả về đường dẫn tuyệt đối của tệp (hoặc thư mục).
getCanonicalPath()| String      | Trả về đường dẫn tuyệt đối, chuẩn hóa của tệp (hoặc thư mục).
getFreeSpace()     | long        | Lấy số byte không gian chưa được phân bổ trên phân vùng chứa đường dẫn này hoặc 0 nếu đường dẫn không hợp lệ.
getName()          | String      | Trả về tên của tệp (hoặc thư mục).
getParent()        | String      | Trả về tên của thư mục cha của tệp (hoặc thư mục).
getPath()          | String      | Trả về đường dẫn của tệp (hoặc thư mục).
getTotalSpace()    | long        | Lấy kích thước của phân vùng chứa đường dẫn tệp trong byte hoặc 0 nếu đường dẫn không hợp lệ.
getUseableSpace()  | long        | Lấy số byte không gian chưa được phân bổ, có thể truy cập được của người dùng, trên phân vùng chứa đường dẫn này hoặc 0 nếu đường dẫn không hợp lệ. Phương thức này cố gắng xem xét quyền ghi của người dùng.
isAbsolute()       | boolean     | Tên tệp (hoặc tên thư mục) có phải là tuyệt đối không?
isDirectory()      | boolean     | Thực thể là một thư mục không?
isFile()           | boolean     | Thực thể là một tệp không?
isHidden()         | boolean     | Thực thể có bị ẩn không? (Phụ thuộc vào hệ thống.)
lastModified()     | long        | Trả về thời gian sửa đổi lần cuối cùng của tệp (hoặc thư mục).
length()           | long        | Trả về độ dài của tệp.
list()             | String[]    | Trả về danh sách các tệp trong thư mục.
listFiles()        | File[]      | Trả về nội dung của thư mục dưới dạng mảng các đối tượng File.
listRoots()        | File[]      | Trả về mảng các hệ thống tệp gốc nếu có (ví dụ, C:/, D:/).
mkdir()            | boolean     | Tạo thư mục.
mkdirs()           | boolean     | Tạo tất cả các thư mục trong đường dẫn.
renameTo(File dest) | boolean    | Đổi tên tệp (hoặc thư mục).
setExecutable()    | boolean     | Đặt quyền thực thi cho tệp.
setLastModified()  | boolean     | Đặt thời gian sửa đổi cuối cùng của tệp (hoặc thư mục).
setReadable()      | boolean     | Đặt quyền đọc cho tệp.
setReadOnly()      | boolean     | Đặt tệp thành trạng thái chỉ đọc.
setWriteable()     | boolean     | Đặt quyền ghi cho tệp.
toPath()           | java.nio.file.Path | Chuyển đổi File thành một NIO File Path (xem NIO File API). (Không nên nhầm lẫn với getPath().)
toURL()            | java.net.URL | Tạo một đối tượng URL cho tệp (hoặc thư mục).

### File Streams

Ồ, có lẽ bạn đã chán ngấy với việc nghe về tệp tin rồi mà chúng ta chưa thậm chí viết một byte nào cả! Nhưng giờ thì thú vị bắt đầu. Java cung cấp hai luồng cơ bản để đọc và ghi vào các tệp tin: FileInputStream và FileOutputStream. Các luồng này cung cấp chức năng cơ bản của InputStream và OutputStream dựa trên byte mà được áp dụng vào việc đọc và ghi tệp tin. Chúng có thể kết hợp với các luồng lọc mà đã được mô tả trước đó để làm việc với các tệp tin theo cách tương tự như việc giao tiếp với các luồng khác.

Bạn có thể tạo một FileInputStream từ một đường dẫn chuỗi hoặc một đối tượng File:

```java
FileInputStream in = new FileInputStream("/etc/passwd");
```

Khi bạn tạo một FileInputStream, hệ thống runtime của Java sẽ cố gắng mở tệp tin được chỉ định. Do đó, các constructor của FileInputStream có thể ném ra một FileNotFoundEx nếu tệp tin được chỉ định không tồn tại hoặc một IOException nếu xảy ra lỗi I/O khác. Bạn phải bắt các ngoại lệ này trong mã của bạn. Bất cứ khi nào có thể, đó là một ý tưởng tốt để tạo thói quen sử dụng cấu trúc try-with-resources mới của Java 7 để tự động đóng các tệp tin khi bạn đã hoàn tất với chúng:

```java
try (FileInputStream fin = new FileInputStream("/etc/passwd")) {
    // ...
    // Fin sẽ được đóng tự động nếu cần khi thoát khỏi mệnh đề try.
}
```

Khi luồng được tạo lần đầu tiên, phương thức available() của nó và phương thức length() của đối tượng File nên trả về cùng một giá trị.

Để đọc các ký tự từ một tệp tin như một Reader, bạn có thể bao bọc một InputStreamReader xung quanh một FileInputStream. Nếu bạn muốn sử dụng kế hoạch mã hóa ký tự mặc định cho nền tảng, bạn có thể sử dụng lớp FileReader thay thế, được cung cấp như một tiện ích. FileReader chỉ là một FileInputStream được bao bọc trong một InputStreamReader với một số giá trị mặc định. Với một lý do nào đó mà điên rồ, bạn không thể chỉ định một mã hóa ký tự cho FileReader sử dụng, vì vậy có lẽ tốt nhất là bỏ qua nó và sử dụng InputStreamReader với FileInputStream.

Các bạn trẻ có một lớp tiện ích nhỏ gọn được gọi là `ListIt`, giúp gửi nội dung của một tệp tin hoặc thư mục đến đầu ra tiêu chuẩn:

```java
import java.io.*;

class ListIt {
    public static void main(String args[]) throws Exception {
        File file = new File(args[0]);
        if (!file.exists() || !file.canRead()) {
            System.out.println("Không thể đọc " + file);
            return;
        }
        if (file.isDirectory()) {
            String[] files = file.list();
            for (String file : files)
                System.out.println(file);
        } else {
            try {
                Reader ir = new InputStreamReader(new FileInputStream(file));
                BufferedReader in = new BufferedReader(ir);
                String line;
                while ((line = in.readLine()) != null)
                    System.out.println(line);
            } catch (FileNotFoundException e) {
                System.out.println("Tệp đã biến mất");
            }
        }
    }
}
```

`ListIt` tạo một đối tượng File từ đối số dòng lệnh đầu tiên và kiểm tra xem Tệp có tồn tại và có thể đọc được không. Nếu Tệp là một thư mục, `ListIt` sẽ xuất các tên tệp trong thư mục. Ngược lại, `ListIt` sẽ đọc và xuất tệp, từng dòng một.

Đối với việc ghi tệp, bạn có thể tạo một FileOutputStream từ một đường dẫn chuỗi hoặc một đối tượng File. Tuy nhiên, khác với FileInputStream, các constructor của FileOutputStream không ném ra một FileNotFoundException. Nếu tệp được chỉ định không tồn tại, FileOutputStream sẽ tạo ra tệp. Các constructor của FileOutputStream có thể ném ra một IOException nếu xảy ra lỗi I/O khác, vì vậy bạn vẫn cần xử lý ngoại lệ này.

Nếu tệp được chỉ định tồn tại, FileOutputStream sẽ mở nó để ghi. Khi bạn gọi phương thức write() sau đó, dữ liệu mới sẽ ghi đè lên nội dung hiện tại của tệp. Nếu bạn cần thêm dữ liệu vào một tệp tin đã tồn tại, bạn có thể sử dụng một hình thức của constructor chấp nhận một cờ ghi thêm:

```java
FileInputStream fooOut = new FileOutputStream(fooFile); // ghi đè fooFile
FileInputStream pwdOut = new FileOutputStream("/etc/passwd", true); // ghi thêm
```

Một cách khác để thêm dữ liệu vào các tệp là với RandomAccessFile, mà chúng ta sẽ thảo luận trong thời gian sớm.

Tương tự như việc đọc, để ghi các ký tự (thay vì byte) vào một tệp, bạn có thể bọc một OutputStreamWriter xung quanh một FileOutputStream. Nếu bạn muốn sử dụng kế hoạch mã hóa ký tự mặc định, bạn có thể sử dụng lớp FileWriter thay thế, được cung cấp như một tiện ích.

Ví dụ sau đọc một dòng dữ liệu từ đầu vào tiêu chuẩn và viết nó vào tệp /tmp/foo.txt:

```java
String s = new BufferedReader(new InputStreamReader(System.in)).readLine();
File out = new File("/tmp/foo.txt");
FileWriter fw = new FileWriter(out);
PrintWriter pw = new PrintWriter(fw);
pw.println(s);
pw.close();
```

Chú ý cách chúng tôi bọc FileWriter trong một PrintWriter để thuận tiện việc viết dữ liệu. Ngoài ra, để trở thành một công dân tốt của hệ thống tệp tin, chúng ta đã gọi phương thức close() khi chúng ta đã hoàn thành với FileWriter. Ở đây, việc đóng PrintWriter đóng Writer cơ sở cho chúng tôi. Chúng ta cũng có thể đã sử dụng try-with-resources ở đây.

### RandomAccessFile

Lớp `java.io.RandomAccessFile` cung cấp khả năng đọc và ghi dữ liệu tại một vị trí xác định trong một tệp. `RandomAccessFile` thực hiện cả hai giao diện `DataInput` và `DataOutput`, vì vậy bạn có thể sử dụng nó để đọc và ghi chuỗi và các loại nguyên thủy tại các vị trí trong tệp giống như nó là một `DataInputStream` và `DataOutputStream`. Tuy nhiên, vì lớp này cung cấp quyền truy cập ngẫu nhiên, thay vì tuần tự, vào dữ liệu tệp, nó không phải là một lớp con của `InputStream` hoặc `OutputStream`.

Bạn có thể tạo một `RandomAccessFile` từ một đường dẫn chuỗi hoặc một đối tượng `File`. Constructor cũng nhận một đối số chuỗi thứ hai mà xác định chế độ của tệp. Sử dụng chuỗi "r" cho một tệp chỉ đọc hoặc "rw" cho một tệp đọc/ghi.

```java
try {
    RandomAccessFile users = new RandomAccessFile("Users", "rw");
} catch (IOException e) { ... }
```

Khi bạn tạo một `RandomAccessFile` trong chế độ chỉ đọc, Java cố gắng mở tệp được chỉ định. Nếu tệp không tồn tại, `RandomAccessFile` ném ra một `IOException`. Tuy nhiên, nếu bạn đang tạo một `RandomAccessFile` trong chế độ đọc/ghi, đối tượng sẽ tạo ra tệp nếu nó không tồn tại. Constructor vẫn có thể ném ra một `IOException` nếu xảy ra một lỗi I/O khác, vì vậy bạn vẫn cần xử lý ngoại lệ này.

Sau khi bạn đã tạo một `RandomAccessFile`, gọi bất kỳ phương thức đọc và ghi thông thường nào, giống như bạn đã làm với `DataInputStream` hoặc `DataOutputStream`. Nếu bạn cố gắng ghi vào một tệp chỉ đọc, phương thức ghi sẽ ném ra một `IOException`.

Điều làm cho `RandomAccessFile` đặc biệt là phương thức `seek()`. Phương thức này nhận một giá trị long và sử dụng nó để thiết lập vị trí đọc và ghi byte trong tệp. Bạn có thể sử dụng phương thức `getFilePointer()` để lấy vị trí hiện tại. Nếu bạn cần thêm dữ liệu vào cuối tệp, sử dụng `length()` để xác định vị trí đó, sau đó `seek()` đến đó. Bạn có thể ghi hoặc seek vượt ra ngoài cuối tệp, nhưng bạn không thể đọc vượt ra ngoài cuối tệp. Phương thức `read()` sẽ ném ra một `EOFException` nếu bạn cố gắng làm điều này.

Dưới đây là một ví dụ về việc viết dữ liệu cho một cơ sở dữ liệu đơn giản:

```java
users.seek(userNum * RECORDSIZE);
users.writeUTF(userName);
users.writeInt(userID);
...
```

Trong ví dụ ngây thơ này, chúng ta giả định rằng độ dài chuỗi cho `userName`, cùng với bất kỳ dữ liệu nào sau nó, vừa với kích thước bản ghi được chỉ định.

### Resource Paths

Một phần lớn trong việc đóng gói và triển khai một ứng dụng là xử lý tất cả các tệp nguồn phải đi kèm với nó, chẳng hạn như các tệp cấu hình, đồ họa và dữ liệu ứng dụng. Java cung cấp một số cách để truy cập vào các tài nguyên này. Một cách là đơn giản là mở tệp và đọc các byte. Một cách khác là xây dựng một URL trỏ đến một vị trí phổ biến trong hệ thống tệp hoặc qua mạng. (Chúng ta sẽ thảo luận về làm việc với URL chi tiết trong Chương 14.)

Vấn đề của các phương pháp này là chúng thường dựa vào sự hiểu biết về vị trí và gói của ứng dụng, điều này có thể thay đổi hoặc gây ra lỗi nếu nó được di chuyển. Điều thực sự cần thiết là một cách thức thông thống để truy cập vào các tài nguyên liên quan đến ứng dụng của chúng ta, bất kể cách cài đặt nó như thế nào. Phương thức `getResource()` của lớp `Class` và classpath Java chỉ cung cấp điều này. Ví dụ:

```java
URL resource = MyApplication.class.getResource("/config/config.xml");
```

Thay vì xây dựng một tham chiếu Tệp đến một đường dẫn tệp tuyệt đối, hoặc dựa vào việc tạo thông tin về thư mục cài đặt, phương thức `getResource()` cung cấp một cách thông thống để có được tài nguyên liên quan đến classpath của ứng dụng. Một tài nguyên có thể được định vị hoặc là tương đối đối với một tệp lớp cụ thể hoặc toàn bộ classpath hệ thống. `getResource()` sử dụng classloader để tải các tệp lớp của ứng dụng để tải dữ liệu. Điều này có nghĩa là không quan trọng ứng dụng lớp nào sử dụng phương thức `getResource()`, miễn là nó từ một class loader có tệp tài nguyên trong classpath của nó. Ví dụ:

```java
URL data = AnyClass.getResource("/config/config.xml");
```

Một URL tương đối không bắt đầu bằng một dấu gạch chéo (ví dụ: mydata.txt). Trong trường hợp này, việc tìm kiếm bắt đầu tại vị trí của tệp lớp mà `getResource()` được gọi. Nói cách khác, đường dẫn là tương đối với gói của tệp lớp mục tiêu. Ví dụ, nếu tệp lớp foo.bar.MyClass được đặt tại đường dẫn foo/bar/MyClass.class trong một thư mục hoặc JAR nào đó của classpath và tệp mydata.txt nằm trong cùng một thư mục (foo/bar/mydata.txt), chúng ta có thể yêu cầu tệp qua MyClass với:

```java
URL data = MyClass.getResource("mydata.txt");
```

Trong trường hợp này, lớp và tệp đến từ cùng một thư mục logic. Chúng ta nói "logic" vì việc tìm kiếm không giới hạn trong thành phần classpath từ mà lớp được tải. Thay vào đó, cùng một đường dẫn tương đối được tìm kiếm trong mỗi thành phần của classpath - giống như một đường dẫn tuyệt đối - cho đến khi nó được tìm thấy.

Một ứng dụng có thể tìm kiếm các tài nguyên như sau:

```java
package mypackage;

import java.net.URL;
import java.io.IOException;

public class FindResources {
    public static void main(String[] args) throws IOException {
        // tuyệt đối từ classpath
        URL url = FindResources.class.getResource("/mypackage/foo.txt");
        // tương đối với vị trí class
        url = FindResources.class.getResource("foo.txt");
        // một tài liệu tương đối khác
        url = FindResources.class.getResource("docs/bar.txt");
    }
}
```

Lớp FindResources thuộc gói mypackage, vì vậy tệp lớp của nó sẽ nằm trong một thư mục mypackage nào đó trên classpath. FindResources định vị tài liệu foo.txt bằng một URL tuyệt đối và sau đó là một URL tương đối. Cuối cùng, FindResources sử dụng một đường dẫn tương đối để truy cập một tài liệu trong thư mục mypackage/docs. Trong mọi trường hợp, chúng ta sử dụng đối tượng Class của FindResources bằng cách sử dụng notation static .class. Một cách khác, nếu chúng ta có một thể hiện của đối tượng, chúng ta có thể sử dụng phương thức getClass() của nó để truy cập đối tượng Class.

Một lần nữa, `getResource()` trả về một URL cho bất kỳ loại đối tượng nào bạn tham chiếu. Điều này có thể là một tệp văn bản hoặc tệp thuộc tính mà bạn

muốn đọc như một luồng, hoặc có thể là một tệp hình ảnh hoặc âm thanh hoặc một đối tượng khác nào đó. Bạn có thể mở một luồng tới URL để phân tích dữ liệu một cách tự mình hoặc chuyển URL đó cho một API xử lý URL. Chúng ta sẽ thảo luận URL chi tiết trong Chương 14. Chúng tôi cũng nên nhấn mạnh rằng việc tải các tài nguyên theo cách này hoàn toàn bảo vệ ứng dụng của bạn khỏi các chi tiết về cách nó được đóng gói hoặc triển khai. Bạn có thể bắt đầu với ứng dụng của bạn trong các tệp rời rạc và sau đó đóng gói nó vào một tệp JAR và các tài nguyên vẫn sẽ được tải. Các applet Java (sẽ được thảo luận trong một chương sau) thậm chí có thể tải các tệp theo cách này qua mạng vì class loader của applet xem máy chủ là một phần của classpath của nó.