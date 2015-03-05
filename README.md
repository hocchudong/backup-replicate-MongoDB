# backup-replicate-MongoDB
### Quản lý dữ liệu Ceilometer thu thập về hệ thống
- Backup dữ liệu

Celometer sẽ giám sát từng máy ảo trong OpenStack và lấy mẫu về hệ thống theo định kỳ trong một khoảng thời gian được khai báo trong cấu hình. Đây là dịch vụ phục vụ lưu log và tính cước cho hệ thống vì vậy cần sao lưu dữ liệu dự phòng. Dữ liệu Ceilometer thu thập về sẽ lưu vào cơ sở dữ liệu là MongoDb. 
Tạo bản sao lưu dữ liệu:
```sh
mongodump --host mongodb.example.net --port 27017
```
 <img src=http://i.imgur.com/pvu8kCG.png width="80%" height="80%" border="1">
 
Trong câu lệnh mongodb.example.net là máy server chứa database MongoDB. Bản sao dữ liệu sẽ được sinh ngay tại thư mục hiện tại, để lưu bản sao về thư mục định nghĩa trước ta thêm tham số –out như sau
```sh
mongodump --host mongodb.example.net --port 27017 –out /data/backup
```
-Khôi phục dữ liệu


Để khôi phục dữ liệu vừa backup ở trên ta dùng mongorestore
```sh
mongorestore –port <port number><path to the backup>
```
 Để khôi phục dữ liệu trên
```sh
mongorestore –port 27017 /root/dump
```

- Quản lý dữ liệu

Ceilometer mặc định sẽ lấy mẫu mười phút một lần, để giám sát máy ảo tốt hơn ta sẽ lấy mẫu một phút một lần, nếu trong hệ thống có nhiều máy ảo thì như vậy dữ liệu Ceilometer thu thập được sau một khoảng thời gian dài sẽ rất lớn chiếm dụng nhiều tài nguyên của hệ thống. Vì vậy ta sẽ xây dựng giải pháp xóa dữ liệu database theo định kỳ như sáu tháng trước hoặc một năm trước.

Script:
```sh
#!/bin/bash

ngay=`date +"20%y-%m-%dT%T.0Z" --date="-1 day -7 hour"`


/usr/bin/expect - << EOF
   spawn mongo
        expect ">"
        send "use ceilometer\r"
        expect ">"
        send "db.meter.remove({recorded_at:{\\\$gte : new Date('$ngay')}})\r"
        expect ">"
        send "exit\r"
        interact
EOF
```
Script trên sẽ xóa dữ liệu trong database một ngày trước
 

Dữ liệu trước khi xóa:
  <img src=http://i.imgur.com/bmX6ctW.png width="80%" height="80%" border="1">
 
Dữ liệu sau khi xóa:
 <img src=http://i.imgur.com/sUuRCxd.png width="80%" height="80%" border="1">

### 1.2 Di chuyển dữ liệu
Trong hệ thống Ceilometer được cài trên các node của OpenStack databaseđược
lưu tại node controller, controller vừa lưu trữ dữ liệu vừa xử lý tác vụ dẫn đến chậm, giảm tính sẵn sàng của hệ thống, vì vậy cần có giải pháp tách riêng database Ceilometer thu thập được sang một server mới. Để làm được việc này ta cần thực hiện các bước sau
Bước 1: Sử dụng mongodump  để tạo bản sao dữ liệu
Bước 2: Chuyển bản sao sang máy chứa dữ liệu
Bước 3: Sử dụng mongorestore để khôi phục dữ liệu đang có lên máy mới
Bước 4: Sửa file cấu hình /etc/ceilometer/ceilometer.conf  kết nối database vào sever mới
[database]
```sh
# The SQLAlchemy connection string used to connect to the
# database (string value)
connection = mongodb://ceilometer:passdb@server_new_ip:27017/ceilometer
```
Khởi động lại dịchvụ

##1.3 Tạo bản sao dữ liệu với Replica Set
  Trong hệ thống một máy lưu cơ sở dữ liệu Ceilometer thu thập có tính dự phòng dữ liệu thấp, trường hợp sever lưu cơ sở dữ liệu này hỏng hoặc mất dữ liệu thì hệ thống sẽ mất hoàn toàn dữ liệu, ta có thể lưu dữ liệu dự phòng bằng mongodump theo định kỳ một ngày hoặc một tháng tuy nhiên giải pháp này sẽ không bảo toàn được dữ liệu. 

 Vì vậy cần xây dựng giải pháp giúp dự phòng dữ liệu theo thời gian thực, giải pháp replication sử dụng Replica Set của mongoDB được lựa chọn.

Mô hình :

<img src=http://i.imgur.com/hr9Zid3.png width="80%" height="80%" border="1">
 
Bước 1: Cài đặt Mongo trên các node primary, secondary1, secondary2
Bước 2: Chỉnh sửa file cấu hình /etc/mongodb.conf trên từng node. Ta có thể chỉnh sửa port và tên của replica, tất cả các node cùng tên replica. Nội dung file cấu hình:
```sh
dbpath=/var/lib/mongodb
logpath=/var/log/mongodb/mongodb.log
logappend=true
nojournal = true
port = 27017
bind_ip = machine_ip (for eg. 100.11.11.11)
fork = true
replSet = rsName   
```
Bước 3: Khởi chạy dịch vụ trên mỗi node:
```sh
/bin/mongod --config /pathToFile/mongod.conf    
```
Bước 4: Trên node ta muốn chọn làm node primary truy cập vào mongodb
```sh 
bin/mongo –host ip_of_the_machine
```
Ở trong dấu nhắc lệnh trong MongoDB ta chạy lệnh 
```sh
>rs.initiate()
```
Bước 5: Ta thêm vào ip của các node secondary:
```sh
>rs.add(“ip_of_secondary_node_1”)
```
```sh
>rs.add(“ip_of_secondary_node_2”)
```
Kiểm tra trạng thái 
```sh
Rs.status()
```
<img src=http://i.imgur.com/VlmUeIF.png width="80%" height="80%" border="1">

Kiểm tra từng máy và sự đồng bộ database
- Node primary
 
<img src=http://i.imgur.com/SFNp88z.png width="80%" height="80%" border="1">

Tạo mới database:
<img src=http://i.imgur.com/vbmF5Q4.png width="80%" height="80%" border="1">
 
- Node secondary
 
<img src=http://i.imgur.com/PtDQyiB.png width="80%" height="80%" border="1">

Ta thấy dữ liệu đã được đồng bộ từ máy primary sang các máy secondary. Các máy secondary không có quyền tạo database mới và khi máy primary hỏng thì một trong các máy secondary sẽ lên thay thế

