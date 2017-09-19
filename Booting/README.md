# Quá trình chạy vào nhân (kernel boot) 

Chương này miêu tả quá trình boot nhân Linux. Như bạn thấy bên dưới,
chúng ta sẽ đi theo từng bài tương ứng với mỗi bước trong quá trình boot nhân Linux:

* [Từ bootloader đến kernel](linux-bootstrap-1.md) - sẽ miêu tả tất cả các bước từ lúc máy tính được bật đến mã máy đầu tiên của nhân được chạy.
* [Các bước đầu tiên trong đoạn code thiết lập nhân](linux-bootstrap-2.md) - sẽ miêu tả các bước đầu tiên trong đoạn mã thiết lập kernel. Bạn sẽ thấy quá trình khởi tạo bộ nhớ heap, truy xuất các thông tin như EDD, IST etc...
* [Khởi tạo chế độ video (video mode) và chuyển chúng sang chế độ protected](linux-bootstrap-3.md) - sẽ miêu tả quá trình khởi động chế độ video trong đoạn mã thiết lập nhân cũng như đoạn chuyển sang chế độ protected.
* [ Chuyển sang chế độ 64-bit](linux-bootstrap-4.md) - sẽ miêu tả các bước để chuyển sang chế độ 64-bit, cũng như chi tiết của chế độ này.
* [Giải nén nhân](linux-bootstrap-5.md) - sẽ miêu tả các chuẩn bị cần thiết trước khi giải nén nhân, cũng như chi tiết của nó chế độ này.
