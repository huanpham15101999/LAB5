# Project 7: Dựng MongoDB replicaset
Yêu cầu:
* Dựng MongoDB replicaset
* Insert DB
* Benchmark
* Test failover

### Tìm hiểu ###
* Database Replication là một kỹ thuật/nguyên lý thiết kế sử dụng trong việc mở rộng (scale) cơ sở dữ liệu bằng cách duy trì các bản sao của dữ liệu trên nhiều server cơ sở dữ liệu khác nhau

* Có 2 loại data replication dựa trên thời gian chuyển đổi dữ liệu giữa các node. Giả sử hệ thống phân tán của chúng ta có 3 server cơ sở dữ liệu A, B, C

**Synchronous**
Máy khách (Client) gửi dữ liệu đến server A và dữ liệu sẽ được sao chép đến server B và server C. Sau khi dữ liệu hoàn thành sao chép, các thông báo hoàn thành sẽ được gửi về cho server A và server A sẽ gửi thông báo đến Máy khách
Ưu điểm: Đảm bảo dữ liệu trên tất cả các server cơ sở dữ liệu là giống nhau - tính đồng nhất giữa các server 
Khuyết điểm: Server A mất thêm một khoảng thời gian để đợi dữ liệu được đồng bộ và gửi hồi đáp lại cho Máy khách. Nếu có một server nào đó không có hồi đáp hoàn thành sao chép dữ liệu thi toàn bộ quá trình sẽ bị rollback (quay trở lại ban đầu)

**Asynchronous**
Máy khách (Client) gửi dữ liệu đến server A, server A sẽ phản hồi các thao tác cho Máy khách, sau đó dữ liệu thay đổi mới được đồng bộ trên server B và C
Ưu điểm: Phản hồi gửi đến Máy khách là ngay lập tức
Khuyết điểm: Có thể dẫn đến sự không đồng bộ khi công việc sao chép dữ liệu đến các server khác xảy ra các vấn đề về mạng
* Các loại Replication Environment

**Multi-Master Replication**
Mỗi node đều đóng vai trò là master node nơi mà Máy khách có thể thực hiện cả việc đọc và ghi dữ liệu. Giữa các node có thể được đồng bộ bằng cả 2 cách nêu trên (Synchronous và Asynchronous). Nhưng khi sử dụng Asynchronous vấn đề về xung đột dữ liệu rất dễ xảy ra khi các Máy khách cập nhật cùng một đơn vị dữ liệu nhưng trên các master-node khác nhau

**Master-Slave Replication**
Máy khách chỉ có quyền đọc/ghi dữ liệu trên master-node. Các slave-node xung quanh sẽ sao chép lại dữ liệu từ master-node, dữ liệu ở slave-node chỉ dùng để đọc. Việc sao chép cũng có thể sử dụng 2 cách trên

### Mô hình LAB ###
### Yêu cầu 1 : Dựng MongoDB replicaset

