# Project 7: Dựng MongoDB replicaset
Yêu cầu:
* Dựng MongoDB replicaset
* Insert DB
* Benchmark
* Test failover

### Tìm hiểu ###
Database Replication là một kỹ thuật thiết kế sử dụng trong việc mở rộng (scale) cơ sở dữ liệu bằng cách duy trì các bản sao của dữ liệu trên nhiều server cơ sở dữ liệu khác nhau

Có 2 loại Data Replication dựa trên thời gian chuyển đổi dữ liệu giữa các node. Giả sử hệ thống phân tán của chúng ta có 3 server cơ sở dữ liệu A, B, C

* Synchronous: Máy khách (Client) gửi dữ liệu đến server A và dữ liệu sẽ được sao chép đến server B và server C. Sau khi dữ liệu hoàn thành sao chép, các thông báo hoàn thành sẽ được gửi về cho server A và server A sẽ gửi thông báo đến Máy khách
  - Ưu điểm: Đảm bảo dữ liệu trên tất cả các server cơ sở dữ liệu là giống nhau - tính đồng nhất giữa các server 
  - Khuyết điểm: Server A mất thêm một khoảng thời gian để đợi dữ liệu được đồng bộ và gửi hồi đáp lại cho Máy khách. Nếu có một server nào đó không có hồi đáp hoàn thành sao chép dữ liệu thi toàn bộ quá trình sẽ bị rollback (quay trở lại ban đầu)

* Asynchronous: Máy khách (Client) gửi dữ liệu đến server A, server A sẽ phản hồi các thao tác cho Máy khách, sau đó dữ liệu thay đổi mới được đồng bộ trên server B và C
  - Ưu điểm: Phản hồi gửi đến Máy khách là ngay lập tức
  - Khuyết điểm: Có thể dẫn đến sự không đồng bộ khi công việc sao chép dữ liệu đến các server khác xảy ra các vấn đề về mạng

Các loại Replication Environment

* Multi-Master Replication: Mỗi node đều đóng vai trò là master node nơi mà Máy khách có thể thực hiện cả việc đọc và ghi dữ liệu. Giữa các node có thể được đồng bộ bằng cả 2 cách nêu trên (Synchronous và Asynchronous). Nhưng khi sử dụng Asynchronous vấn đề về xung đột dữ liệu rất dễ xảy ra khi các Máy khách cập nhật cùng một đơn vị dữ liệu nhưng trên các master-node khác nhau
* Master-Slave Replication: Máy khách chỉ có quyền đọc/ghi dữ liệu trên master-node. Các slave-node xung quanh sẽ sao chép lại dữ liệu từ master-node, dữ liệu ở slave-node chỉ dùng để đọc. Việc sao chép cũng có thể sử dụng 2 cách trên

### Mô hình LAB ###

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/5p4646kuyh_2.png)

Mô hình gồm 3 server database sử dụng hệ điều hành Ubuntu 20.04
* mongo0: IP 192.168.18.153
* mongo1: IP 192.168.18.52
* mongo2: IP 192.168.18.166

### Yêu cầu 1: Dựng MongoDB replicaset
#### Bước 1: Cài đặt MongoDB trên cả 3 server

Cài đặt

`# curl -fsSL https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add -`

`# echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/4.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.4.list`

`# sudo apt update -y `

`# sudo apt install mongodb-org -y`

Khởi động dịch vụ 

`# sudo systemctl start mongod`

Kiểm tra trạng thái dịch vụ

`# sudo systemctl status mongod`

Cho phép dịch vụ khởi động cùng hệ thống mỗi lần boot lên 

`# sudo systemctl enable mongod`

Kiểm tra trạng thái kết nối database, nếu thấy `"ok":1` là thành công 

`# mongo --eval 'db.runCommand({ connectionStatus: 1 })'`

#### Bước 2: Cấu hình một MongoDB Replica Set 

**Edit file hosts trên cả 3 server**

`# sudo vi /etc/hosts`
```
192.168.18.153  mongo0.replset.member
192.168.18.52   mongo1.replset.member
192.168.18.166  mongo2.replset.member
```
**Enable tính năng replication trong file config MongoDB của mỗi server**

`# sudo vi /etc/mongod.conf`

mongo0:
```
# network interfaces
net:
  port: 27017
  bindIp: 127.0.0.1,mongo0.replset.member
...
# replication
replication:
  replSetName: "rs0"
```
mongo1:
```
# network interfaces
net:
  port: 27017
  bindIp: 127.0.0.1,mongo1.replset.member
...
# replication
replication:
  replSetName: "rs0"
```
mongo2:
```
# network interfaces
net:
  port: 27017
  bindIp: 127.0.0.1,mongo2.replset.member
...
# replication
replication:
  replSetName: "rs0"
```
Restart dịch vụ

`# sudo systemctl restart mongod`

Kiểm tra kết nối giữa các server database trong Replica Set

Trên mongo0 kiểm tra kết nối tới mongo1 và mongo2

`# nc -zv mongo1.replset.member 27017`

`# nc -zv mongo2.replset.member 27017`

Kết quả thành công 

`Connection to mongo1.replset.member 27017 port [tcp/*] succeeded!`

`Connection to mongo2.replset.member 27017 port [tcp/*] succeeded!`

**Khởi tạo Replica Set và add các thành viên vào**
 
Trên mongo0 

`# mongo`

`> rs.initiate({_id: "mongo_rs", members:[{_id:1, host: "mongo1:27017", priority:3},{_id:2, host: "mongo2:27017", priority:2}, {_id:3, host: "mongo3:27017", priority: 1}]})`

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/yleemw40re_Screenshot%20from%202022-05-18%2013-43-16.png)

Kiểm tra trạng thái Replica Set `> rs.status()`

Lúc này 

* mongo0.replset.member giữ vai trò PRIMARY (priority = 3)
* mongo1.replset.member giữ vai trò SECONDARY (priority = 2)
* mongo2.replset.member giữ vai trò SECONDARY (priority = 1)

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/u5qkkjp5bc_Screenshot%20from%202022-05-18%2014-08-11.png)

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/vcbx9nm0td_Screenshot%20from%202022-05-18%2014-10-42.png)

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/6cxhbq1kd4_Screenshot%20from%202022-05-18%2014-11-40.png)

Xem config Replica Set `> rs.conf()`

### Yêu cầu 2: Tạo Database trên mongo0 (primary) và kiểm tra đồng bộ trên mongo1 và mongo2 (secondary) 

**Trên server mongo0 (primary)**

Tạo Database và Collection 

`rs0:PRIMARY> use huan`

`rs0:PRIMARY> db.createCollection("danhsach")`

Kiểm tra Database và Collection vừa tạo

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/dum6vg7isn_Screenshot%20from%202022-05-18%2015-08-54.png)

**Kiểm tra server mongo1 và mongo2 (secondary) ta thấy dữ liệu đã được đồng bộ**

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/tzdy098r0z_Screenshot%20from%202022-05-18%2015-14-03.png)

*Chú ý: Dùng lệnh rs.secondaryOk() để cho phép query trên node secondary*

### Yêu cầu 3: Test failover

**Stop service trên server mongo0 (primary)**

`# sudo systemctl stop mongod`

**Kiểm tra Replica Set status trên mongo1 và mongo2**

`> rs.status()`

Ta thấy mongo1 với id=1 lúc này đã nắm vai trò **primary**, mongo2 với id=2 nắm vai trò **secondary** (do priority mongo1 = 2 > priority mongo2 = 1)

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/5dpmfjpm7m_Screenshot%20from%202022-05-18%2015-30-49.png)

**Start lại service trên mongo0 và check lại**

Lúc này mongo0 với id=0 lại quay lại vai trò **primary** (do priority = 3 là lớn nhất)

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/3sxdxgn81h_Screenshot%20from%202022-05-18%2015-39-41.png)

**=> Failover OK**

*Link tham khảo*
*https://www.digitalocean.com/community/tutorials/how-to-configure-a-mongodb-replica-set-on-ubuntu-20-04*

