**Small Enterprise Network**

## 1. Tổng quan dự án

Dự án này mô phỏng một hạ tầng mạng doanh nghiệp quy mô nhỏ sử dụng các công nghệ của Cisco. Bài lab tập trung triển khai các kiến trúc mạng chuẩn CCNA bao gồm: phân mảnh mạng VLAN, định tuyến động, dịch vụ mạng tập trung và thực thi các chính sách bảo mật đa lớp thông qua Port Security và ACL.

**Các công nghệ cốt lõi:**

* Kiến trúc mạng Multi-VLAN bao gồm VLAN 10 và VLAN 20
* Định tuyến Inter-VLAN theo mô hình Router-on-a-Stick
* Dịch vụ DHCP Relay ứng dụng lệnh ip helper-address
* Dự phòng mạng Layer 2 với STP thông qua đường kết nối chéo vật lý
* Định tuyến động với giao thức OSPF Single Area 0
* Biên dịch địa chỉ mạng NAT Overload chuẩn PAT
* Bảo mật cổng Layer 2 Port Security với cấu hình Sticky MAC
* Lọc lưu lượng dựa trên Access Control List để thực thi chính sách bảo mật
* Giả lập môi trường Internet thông qua Upstream Router của nhà mạng

## 2. Sơ đồ mạng

                                [ Internet Server - 8.8.8.8 ]
                                        |
                                    [ R2 - ISP ] 
                                     Gig0/1 .1
                                        |
                            10.0.0.0/30 | OSPF Area 0
                                        |
                                     Gig0/0 .2
                           [ R1 - HQ Gateway & NAT ]
                                     Gig0/1 
                                        |
                                  Đường Trunk
                                        |
                               [ SW1 - Core Switch ]
                                     /     \
                         Đường Trunk /       \ Đường Trunk
                                   /         \
                   VLAN 10 STAFF                 VLAN 20 IT
                     [ SW0 ] <== STP Backup ==> [ SW2 ]
                     /     \                     /    \
                 PC0      PC1               Laptop0  DHCP+DNS Server
                DHCP      DHCP               DHCP     192.168.20.10

```

## 3. Quy hoạch địa chỉ IP

**Bảng gán địa chỉ IP thiết bị**

| Thiết bị | Cổng Interface | Địa chỉ IP / Subnet | Vùng mạng và Vai trò |
| --- | --- | --- | --- |
| **R1** | G0/1.10 | 192.168.10.1 /24 | Gateway vùng VLAN 10 |
| **R1** | G0/1.20 | 192.168.20.1 /24 | Gateway vùng VLAN 20 |
| **R1** | G0/0 | 10.0.0.2 /30 | Đường WAN nối sang ISP |
| **R2** | G0/2 | 10.0.0.1 /30 | Đường WAN nối về Công ty |
| **R2** | G0/1 | 8.8.8.1 /24 | Gateway mạng Internet |
| **Server** | Fa0 | 192.168.20.10 /24 | Máy chủ DHCP+DNS nội bộ |
| **Server** | Fa0 | 8.8.8.8 /24 | Máy chủ Web ngoài Internet |

## 4. Phân chia VLAN và Dự phòng Layer 2

* **VLAN 10:** Tên vùng STAFF, sử dụng dải mạng 192.168.10.0/24
* **VLAN 20:** Tên vùng IT_SERVER, sử dụng dải mạng 192.168.20.0/24
* **Cấu hình Trunking:** Thiết lập cổng Trunk trên các kết nối giữa R1, SW1, SW0, và SW2.
* **Dự phòng STP:** Sử dụng một sợi cáp chéo kết nối trực tiếp giữa SW0 và SW2. Giao thức Spanning Tree Protocol sẽ tự động khóa một cổng để chống hiện tượng bão mạng Broadcast Storm, đồng thời sẵn sàng mở cổng để chuyển hướng dữ liệu nếu đường link chính lên SW1 bị đứt.

## 5. Bảo mật Layer 2 với Port Security

Cấu hình trực tiếp trên các cổng Access của SW0, đây là nơi kết nối với các máy tính phòng STAFF:

* **Chế độ cổng:** Thiết lập ở mức Access
* **Sticky MAC:** Hệ thống tự động học và dính chặt địa chỉ MAC của thiết bị đầu tiên kết nối vào cổng.
* **Số lượng MAC tối đa:** Giới hạn chuẩn 1 địa chỉ MAC trên mỗi cổng.
* **Hành vi vi phạm:** Thiết lập trạng thái Shutdown. Cổng Switch sẽ tự động sập nguồn và khóa lại nếu phát hiện có máy tính lạ hoặc thiết bị giả mạo cắm vào hệ thống nội bộ.

## 6. Định tuyến Inter-VLAN và OSPF

* **Router-on-a-Stick:** Router R1 sử dụng các cổng ảo sub-interfaces G0/1.10 và G0/1.20 để bóc tách và định tuyến lưu lượng qua lại giữa các VLAN.
* **Định tuyến động OSPF Area 0:** Loại bỏ hoàn toàn định tuyến tĩnh cơ bản. Router R1 và R2 tự động thiết lập mối quan hệ láng giềng Neighbor để trao đổi bảng định tuyến, qua đó giúp thông mạng giữa LAN nội bộ doanh nghiệp và mạng Internet giả lập một cách tự động.

## 7. Dịch vụ DHCP và DNS Tập trung

* **DHCP Server:** Hệ thống sử dụng máy chủ DHCP vật lý đặt trong vùng VLAN 20 tại địa chỉ tĩnh 192.168.20.10.
* **DHCP Relay Agent:** Cổng ảo G0/1.10 trên Router R1 được cấu hình lệnh ip helper-address trỏ về 192.168.20.10 để chuyển tiếp các gói tin xin IP Broadcast từ vùng VLAN 10 sang cho máy chủ DHCP xử lý.
* **Dịch vụ DNS:** Cấu hình trực tiếp trên máy chủ nội bộ để phân giải tên miền google.com về địa chỉ IP 8.8.8.8.

## 8. Biên dịch địa chỉ mạng NAT

* **PAT hay NAT Overload:** Cấu hình trực tiếp trên Router R1.
* **Vùng nội bộ Inside:** Các Sub-interfaces quản lý vùng VLAN bao gồm G0/1.10 và G0/1.20.
* **Vùng ngoại vi Outside:** Cổng WAN nối ra Internet là G0/0.
* Tính năng này cho phép toàn bộ các máy trạm trong mạng nội bộ đang sử dụng địa chỉ IP tư nhân Private IP có thể ra ngoài lướt Internet thông qua một địa chỉ IP công cộng Public IP duy nhất là 10.0.0.2.

## 9. Chính sách Bảo mật mạng ACL

Sử dụng Extended ACL số 100 áp dụng theo chiều ngõ vào Inbound trên cổng ảo G0/1.10 của Router R1:

* **Hạn chế đối với vùng STAFF VLAN 10:**
* Bị chặn hoàn toàn quyền gửi gói tin ping ICMP sang dải mạng VLAN 20. Thao tác này giúp ngăn chặn nhân viên tự ý rà quét hoặc phá hoại hệ thống máy chủ IT quan trọng.


* **Quyền hạn của vùng STAFF VLAN 10:**
* Được phép truy cập ra mạng Internet bên ngoài, cụ thể là có thể lướt Web hoặc Ping tới địa chỉ 8.8.8.8.
* Được phép gửi và nhận gói tin xin cấp IP từ DHCP Server bình thường mà không bị gián đoạn.



## 10. Kịch bản Kiểm thử và Nghiệm thu

* **Kiểm tra DHCP:** Các máy tính ở vùng VLAN 10 nhận chính xác địa chỉ IP tự động dạng 192.168.10.x nhờ tính năng DHCP Relay Agent đang hoạt động trên Router R1.
* **Kiểm tra OSPF:** Gõ lệnh show ip route trên Router R1 sẽ hiển thị các đường mạng có ký tự O ở đầu. Đây chính là các tuyến đường mạng được tự động học từ giao thức OSPF của Router R2.
* **Kiểm tra Port Security:** Rút dây mạng của máy PC0 ra, cắm vào một máy tính xách tay lạ. Ngay khi thiết bị lạ này phát tín hiệu ping, cổng Switch lập tức chuyển sang trạng thái sập nguồn và đèn tín hiệu cổng biến thành màu đỏ ngòm.
* **Kiểm tra Tường lửa ACL:** Đứng từ máy PC0 gõ lệnh ping 192.168.20.10 sẽ bị báo lỗi Destination host unreachable. Tuy nhiên, khi gõ lệnh ping 8.8.8.8 thì vẫn nhận được phản hồi Reply thành công từ máy chủ web.

## 11. Kết luận

Bài lab đã mô phỏng thành công một mô hình thiết kế mạng doanh nghiệp thực tế, qua đó giúp tối ưu hóa chi phí phần cứng mạng. Dự án làm nổi bật khả năng triển khai hệ thống dịch vụ mạng tập trung DHCP Relay, đảm bảo tính sẵn sàng cao ở tầng vật lý thông qua STP dự phòng, tự động hóa quá trình xây dựng bản đồ đường đi mạng với OSPF, và thực thi các chính sách an ninh mạng nghiêm ngặt ở cả tầng vật lý lẫn luận lý nhờ kết hợp Port Security và ACL.
