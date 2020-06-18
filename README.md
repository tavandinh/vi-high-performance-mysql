# Bản dịch cuốn sách high performance MYSQL

Bản dịch từ cuốn sách  high performance MYSQL, bản dịch này được viết bằng hexo. Nội dung của trang được viết với định dạng Markdown và chứa trong thư mục `source/posts`. Mọi đóng góp dù lớn hay nhỏ cũng đều được hoan nghênh.

Để tham gia phát triển phiên bản này, các bạn chỉ cần làm theo các bước sau:

1. Fork repo này
1. Cài đặt tất cả các gói phụ thuộc (dependencies): `npm i`
1. Mở server dev ở [`localhost:4000/archives`](http://localhost:4000/archives/): `npm start`

Bây giờ bạn có thể chỉnh sửa các file trong thư mục `source/posts` và refresh trình duyệt để kiểm tra kết quả chỉnh sửa.

## Quy ước khi đóng góp

Vì phiên bản này mới chỉ được bắt đầu, chúng ta sẽ tập trung vào các file chưa dịch thay vì chỉnh sửa các file đã dịch xong (tuy vậy, nếu có sai sót gì nghiêm trọng ở các file đã dịch, rất hoan nghênh sự đóng góp của các bạn). Để bắt đầu dịch một file mới, vui lòng thực hiện theo quy ước sau:

* Chỉ dịch một file cho mỗi PR
* Trước khi bắt tay vào dịch một file, hãy kiểm tra trên [trang PR](https://github.com/vuejs-vn/vuejs.org/pulls) xem có ai đã hoặc đang dịch file đó hay chưa
* Tạo một branch mới từ `master` với tên là đường dẫn đến file đó, ví dụ `posts/7_Advanced_MySQL_Features.md`
* Tạo một thay đổi nhỏ để submit PR vào `master`, với tên PR là `[CHORE] /path/to/file.md`


