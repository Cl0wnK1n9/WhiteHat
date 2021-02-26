# WEB02 WHITEHAT 3.0
Bài này dựa trên lỗ hổng của wordpress bản 5.0.0(CVE-2019-8943). Qua bài này có lẽ mọi người sẽ thấy được tầm quan trọng của việc chơi CTF. Có người bảo CTF không thực tế không nên tốn thời gian vào nó. CTF đúng chỉ là một cuộc chơi của giới ATTT nhưng chúng ta luôn cần những người giỏi trong cuộc chơi đó!
Writeup này mình sẽ cố gắng viết và giải thích kỹ tất cả mọi thứ nếu có gì không hiểu hoặc mình giải thích dài dòng quá mong được mọi người góp ý <3.

link: http://139.180.135.2:9002/

Đầu tiên đi qua các chức năng cơ bản của trang web xem có những chức năng gì

1. Chức năng Upload ảnh 

![img1](https://github.com/Cl0wnK1n9/WhiteHat/blob/main/img/Capture1.PNG)

Chức năng cho phép upload ảnh vào thư mục /uploads/ và tên của ảnh sẽ được mã hóa md5.

2. Chức năng tạo post

![img2](https://github.com/Cl0wnK1n9/WhiteHat/blob/main/img/Capture2.PNG)

Tạo 1 post bao gồm title, nội dung và cho phép nhúng ảnh vào. Ở phần ảnh thì chỉ cần nhập tên ảnh (tên đã mã hóa md5) thì code sẽ tự link đến /uploads/

3. Chức năng xem post

![img3](https://github.com/Cl0wnK1n9/WhiteHat/blob/main/img/Capture3.PNG)

Như cái tên chức năng được dùng để xem post mà bạn đã tạo thông qua postid ngoài ra còn inlcude thêm file theme mà chính xác ở đây là theme1.php và theme2.php là những file tồn tại theo mặc định.

4. Chức năng xoay ảnh 

![img4](https://github.com/Cl0wnK1n9/WhiteHat/blob/main/img/Capture4.PNG)

Well chức năng dùng để edit lại ảnh cho phép bạn xoay ảnh theo 1 góc nhất định

# STEP 1: Tìm source code
Sau một thôi một hồi chọc ngoáy quanh trang web mình nhận ra có 2 điểm: 
- ctrl + U cho ta biết flag sẽ nằm ở /flag.txt suy ra có khả năng bài này sẽ phải RCE 
- LFI cái này là mình đoán vì có tham số theme là 1 file PHP

=> Đoán cũng là một kỹ năng 

Sau đó mình thử thay đổi giá trị tại tham số theme các giá trị mình thử gồm `../uploads/{tên ảnh}.jpg` bla bla bla để xem liệu có ra gì không 

![img5](https://github.com/Cl0wnK1n9/WhiteHat/blob/main/img/Capture5.PNG)

Vậy là đây không phải là một bài LFI bình thường vì nó có kiểm tra giá trị theme hoặc làm gì đó để bảo vệ giá trị này.

Tiếp đến đối với giá trị postid mình có thử lỗi SQLi ở đây vì bạn hoàn toàn có thể ghi file bằng lỗi SQLi. 

Khi mà cả 2 phép thử kia đều không mang lại kết quả thì mình đã thử xóa cả 2 tham số đi xem liệu server có throw về lỗi gì không và từ đó mình sẽ tìm cách tiếp và ...

![img6](https://github.com/Cl0wnK1n9/WhiteHat/blob/main/img/Capture6.PNG)

Bingo! 

# STEP 2: Phân tích source code

Khi phân tích source mình sẽ đọc từ 'hàm main' trước để hiểu được luồng thực thi của chương trình xem những hàm nào sẽ được gọi rồi sau đó mới đào sâu vào phân tích kỹ từng hàm 1 vì lúc đó bạn đã có một cái nhìn tổng quát vè chương trình rồi nên việc tìm ra lỗi sẽ dễ hơn.

```
if (isset($_GET['post_id'])) {
    require_once('post_header.php');
    
    $post_id = $_GET['post_id'];
    global $post;
    $post = get_post($post_id);
    if (isset($_GET['theme'])) {
        $theme = $_GET['theme'];
    } else {
        $theme = 'theme1.php';
    }
    check_theme($theme);

    include('themes/'.$theme);
    echo("<button class='btn btn-primary' id='edit-btn'>Edit image</button>
<form action='post.php' method='post' width='50%'>
    <div class='form-group' id='edit-div'>
    <label for='exampleInputEmail1'>Degree</label>
    <input type='text' class='form-control' id='exampleInputEmail1' placeholder='90' name='degree'>
    <input type='hidden' class='form-control' id='exampleInputEmail2' value='$post_id' name='post_id'>
    <button type='submit' class='btn btn-primary'>Rotate</button>
  </div>");
    require_once('post_footer.php');
}
else if (isset($_POST['post_id']) && isset($_POST['degree'])) {
    $post_id = $_POST['post_id'];
    rotate_feature_img($post_id, (int) $_POST['degree']);
    header("Location: /post.php?post_id=$post_id&theme=theme1.php");
} else {
    show_source('post.php');
}

require_once('footer.php');

```

Web sẽ nhận tham số `post_id` và gọi đến hàm `get_post`, giá trị trả về sẽ được lưu trong một biến global. Tiếp theo là lấy giá trị của `theme` nếu không tồn tại thì sẽ tự động gán bằng giá trị `theme1.php`, sau đó hàm `check_theme` sẽ được gọi. Sau khi kiểm tra theme sẽ được đưa vào `include`, đến lúc này mình đã chắc chắn bài này là về LFI. Tiếp theo là đoạn code dùng đề edit lại ảnh trong chức năng xem post, ở đây sẽ gọi đến hàm `rotate_feature_img`. Sau khi có 1 cái nhìn tổng quát về flow của chương trình mình sẽ đi sâu vào 1 số hàm.

Điểm qua chức năng của 1 số hàm lặt vặt như `get_title`, `get_body` hai hàm này sẽ lấy giá trị bên trong biến global => giá trị trả về của biến post sẽ là một mảng các giá trị như title, body,... và hàm `get_img_src` sẽ lấy giá trị tại địa thư mục `/uploads/`. Ở hàm `get_img_src`cách mà ảnh được lấy về, ảnh được lấy thông qua http request chứ không lấy trực tiếp trên server đây là điểm khác biệt giữa việc sử dụng `http://127.0.0.1/imgFile.png` và `/imgFile.png` tức là một cái bạn sẽ thực hiện http request còn cái còn lại thì không. 

Đi sâu vào hàm đầu tiên mình để ý đến khi xem source chính là hàm `check_theme` đây là hàm khiến cho những phép thử LFI bên trên của mình fail.
```
function check_theme($theme) {
    if(!in_array($theme, scandir('./themes'))) {
        die("Invalid theme!");
    }
}
```
hàm này sẽ kiểm tra xem file theme mà được truyền vào có tồn tại trong thư mục `/themes/` hay không nếu không sẽ trả về giá trị invalid điều này có nghĩa bạn phải tìm cách upload được payload của mình vào thư mục này. Điều này có nghĩa là chức năng upload ảnh sẽ hoàn toàn không có tác dụng gì bởi vì tên file đã được mã hóa md5. Vậy mục đích tiếp theo sẽ là phải tìm cách ghi file vào thư mục `/themes/`.

Đến với hàm cuối cùng `rotate_feature_img`.

```
function rotate_feature_img($post_id, $degrees) {
    $post = get_post($post_id);
    $src_file = $post['feature_img'];
    $ext = strtolower(pathinfo($src_file, PATHINFO_EXTENSION));
    
    $src_file = "uploads/" . $src_file;   
    if ( !file_exists( $src_file ) ) {      
        $base_url = "http://" . $_SERVER['HTTP_HOST']; 
        $src = $base_url. "/" . $src_file;            
    } else {
        $src = $src_file;
    }
    try {
        $img = new Imagick($src); 
        $img->rotateImage('white', $degrees);
    } catch (Exception $e) {
        die('fail to rotate image');
    }
    @mkdir(dirname($src_file));
    @$img->writeImage($src_file);
}
```

ở hàm này `get_post` lại được gọi và lưu vào biến `$post` và biến này hoàn toàn khác với biến global kia nhé =))) lúc này mình kiểu wtf what for ???? Tác giả rảnh à :v cơ mà thôi kệ =))) đọc tiếp. Lấy phần mở rộng của file ảnh và chuyển về chữ thường xong ... không dùng .-. ok tác giả rảnh thật thề luôn. Sau đó kiểm tra xem file ảnh có tồn tại trong thư mục `/uploads/` hay không nếu có thì gán luôn vào `$src` nhưng nếu file không tồn tại thì sao? 
```$base_url = "http://" . $_SERVER['HTTP_HOST']; 
        $src = $base_url. "/" . $src_file;
```

file sẽ được gọi thông qua HTTP requests nhớ những gì mình nói bên trên chứ? Sau đó sử dụng Imagick để thực hiện xoay ảnh và lưu lại vào trong `$src_file` đương nhiên nếu file không phải là ảnh thì sẽ trigger exception.

# Step 3: Tìm lỗ
Như mình đã nói lỗ hổng chắc chắn là LFI và việc mình cần làm là tìm cách ghi file vào thư mục `/themes/` mà không phải bằng cách upload ảnh. Cách duy nhất chính là lợi dùng hàm `rotate_feature_img`. Điều đầu tiên chính là `$_SERVER['HTTP_HOST']` đây là biến dùng để lấy tên host của server kiểu như `127.0.0.1` hay `clownking.vn` nhưng lấy như nào? Biến này sẽ lấy giá trị dựa trên trường `host` trong http request (1). Điều này có nghĩa là nếu thay đổi giá trị của trường `host` thì giá trị của `$_SERVER['HTTP_HOST']` sẽ thay đổi theo. Lợi dụng lỗ hổng này mình có thể trỏ việc load ảnh để sửa về server của mình như bằng cách sửa host thành `myserver.com` và tên ảnh là `payload.jpg` --> `$src = "https://myserver.com/uploads/payload.jpg"`.
Tiếp theo để ý đến phần tạo file `@mkdir(dirname($src_file))`. File được tạo vào thư mục gốc của giá trị `$src_file` tức là sẽ lưu vào thư mục `/uploads/` tuy nhiên điều này sẽ hoàn toàn thay đổi nếu tên file nhập vào là `../themes/payload.jpg` và với payload này đường dẫn sẽ được trỏ về `https://myserver.com/uploads/../themes/payload.jpg` và giá trị của `$src_file` sẽ là `/uploads/../themes/payload.jpg` đồng nghĩa với việc file sẽ được ghi vào thư mục `/themes/`(2).
Sau (1) và (2) thì đạt được mục đích chính là ghi file vào trong thư mục `/themes/` và sử dụng LFI để thực thi file đó. Tuy nhiên file được chỉnh sửa bằng `imagick` chứ không phải được ghi trực tiếp, dữ liệu sẽ bị thay đổi T-T. Lúc này mình phải tìm xem inject code php vào như nào để giữ nguyên được ảnh và giữ nguyên được code. Để làm được điều này thì mình sử dụng 1 đoạn bash để so sánh 2 file 1 file là file gốc và file còn lại là file đã được edit (code mình để ở file c0mp4r3).

![img7](https://github.com/Cl0wnK1n9/WhiteHat/blob/main/img/Capture7.png)

Đây chính là đoạn dữ liệu duy nhất không bị thay đổi và sau khi goole thì mình biết nó là EXIF Data trường này dù có bị sửa hay xóa cx sẽ không ảnh hưởng đến ảnh PERFECT ! giờ chỉ cần đọc file thôi còn đọc thế nào hay sao không gọi shell thì các bạn tìm hiểu nốt nhé =)))) bài viết dài lắm rồi ! 

POC: 

![img8](https://github.com/Cl0wnK1n9/WhiteHat/blob/main/img/Capture8.png)
