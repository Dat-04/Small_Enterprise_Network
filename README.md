**Small Enterprise Network**

## 1. Tổng quan (Overview)

Dự án này mô phỏng một hạ tầng mạng doanh nghiệp quy mô nhỏ sử dụng các công nghệ của Cisco. Bài lab tập trung triển khai các kiến trúc mạng chuẩn CCNA bao gồm: phân mảnh mạng (VLAN), định tuyến động, dịch vụ mạng tập trung và thực thi các chính sách bảo mật đa lớp (Port Security & ACLs).

**Các công nghệ cốt lõi:**

* Kiến trúc mạng Multi-VLAN (VLAN 10 & VLAN 20)
* Định tuyến Inter-VLAN (Mô hình Router-on-a-Stick)
* Dịch vụ DHCP Relay (`ip helper-address`)
* Dự phòng mạng Layer 2 với STP (Đường kết nối chéo vật lý)
* Định tuyến động với OSPF (Single Area 0)
* Biên dịch địa chỉ mạng NAT Overload (PAT)
* Bảo mật cổng Layer 2 Port Security (Sticky MAC)
* Lọc lưu lượng dựa trên Access Control List (Chính sách bảo mật ACL)
* Giả lập môi trường Internet thông qua Upstream Router của nhà mạng

## 2. Sơ đồ mạng (Network Topology)

                                [ Internet Server ] (8.8.8.8)
                                        |
                                    [ R2 (ISP) ] 
                                   (Gig0/1) .1
                                        |
                            10.0.0.0/30 | OSPF Area 0
                                        |
                                   (Gig0/0) .2
                            [ R1 (HQ Gateway & NAT) ]
                                   (Gig0/1) 
                                        |
                                  (Đường Trunk)
                                        |
                                     [ SW1 ] (Switch Trung tâm)
                                     /     \
                        (Đường Trunk) /       \ (Đường Trunk)
                                   /         \
                 (VLAN 10 - STAFF)             (VLAN 20 - IT)
                     [ SW0 ] <== STP Dự phòng ==> [ SW2 ]
                     /     \                     /    \
                 PC0      PC1               Laptop0  DHCP+DNS Server
                (DHCP)   (DHCP)             (DHCP)   (192.168.20.10)

```

## 3. Quy hoạch địa chỉ IP (IP Addressing Plan)

**Bảng gán địa chỉ IP thiết bị**

| Thiết bị | Cổng (Interface) | Địa chỉ IP / Subnet | Vùng mạng / Vai trò |
| --- | --- | --- | --- |
| **R1** | G0/1.10 | 192.168.10.1 /24 | Gateway vùng VLAN 10 |
| **R1** | G0/1.20 | 192.168.20.1 /24 | Gateway vùng VLAN 20 |
| **R1** | G0/0 | 10.0.0.2 /30 | Đường WAN nối sang ISP |
| **R2** | G0/2 | 10.0.0.1 /30 | Đường WAN nối về Công ty |
| **R2** | G0/1 | 8.8.8.1 /24 | Gateway mạng Internet |
| **Server** | Fa0 | 192.168.20.10 /24 | Máy chủ DHCP+DNS nội bộ |
| **Server** | Fa0 | 8.8.8.8 /24 | Máy chủ Web ngoài Internet |

## 4. Phân chia VLAN & Dự phòng Layer 2

* **VLAN 10:** STAFF (Dải mạng `192.168.10.0/24`)
* **VLAN 20:** IT_SERVER (Dải mạng `192.168.20.0/24`)
* **Cấu hình Trunking:** Thiết lập cổng Trunk trên các kết nối giữa R1, SW1, SW0, và SW2.
* **Dự phòng STP:** Sử dụng một sợi cáp chéo kết nối trực tiếp giữa SW0 và SW2. Giao thức Spanning Tree Protocol (STP) sẽ tự động khóa một cổng để chống bão mạng (Broadcast Storm), đồng thời sẵn sàng mở cổng để chuyển hướng dữ liệu nếu đường link chính lên SW1 bị đứt.

## 5. Bảo mật Layer 2 (Port Security)

Cấu hình trực tiếp trên các cổng Access của SW0 (nơi kết nối với các máy tính phòng STAFF):

* **Chế độ cổng:** Access
* **Sticky MAC:** Tự động học và dính chặt địa chỉ MAC của thiết bị đầu tiên kết nối vào cổng.
* **Số lượng MAC tối đa:** Giới hạn `1` địa chỉ MAC trên mỗi cổng.
* **Hành vi vi phạm:** `Shutdown` (Cổng Switch sẽ tự động sập nguồn và khóa lại nếu phát hiện có máy tính lạ hoặc thiết bị giả mạo cắm vào).

## 6. Định tuyến Inter-VLAN & OSPF

* **Router-on-a-Stick:** Router R1 sử dụng các cổng ảo sub-interfaces (`G0/1.10` và `G0/1.20`) để bóc tách và định tuyến lưu lượng qua lại giữa các VLAN.
* **Định tuyến động OSPF (Area 0):** Loại bỏ hoàn toàn định tuyến tĩnh cơ bản. Router R1 và R2 tự động thiết lập mối quan hệ láng giềng (Neighbor) để trao đổi bảng định tuyến, giúp thông mạng giữa LAN nội bộ doanh nghiệp và mạng Internet giả lập.

## 7. Dịch vụ DHCP & DNS Tập trung

* **DHCP Server:** Hệ thống sử dụng máy chủ DHCP vật lý đặt trong vùng VLAN 20 (`192.168.20.10`).
* **DHCP Relay Agent:** Cổng ảo `G0/1.10` trên Router R1 được cấu hình lệnh `ip helper-address 192.168.20.10` để chuyển tiếp các gói tin xin IP (Broadcast) từ vùng VLAN 10 sang cho máy chủ DHCP xử lý.
* **Dịch vụ DNS:** Cấu hình trên máy chủ để phân giải tên miền `google.com` về địa chỉ IP `8.8.8.8`.

## 8. Biên dịch địa chỉ mạng NAT (Truy cập Internet)

* **PAT (NAT Overload):** Cấu hình trên Router R1.
* **Vùng nội bộ (Inside):** Các Sub-interfaces quản lý VLAN (`G0/1.10`, `G0/1.20`).
* **Vùng ngoại vi (Outside):** Cổng WAN nối ra Internet (`G0/0`).
* Tính năng này cho phép toàn bộ các máy trạm trong mạng nội bộ sử dụng địa chỉ IP tư nhân (Private IP) có thể ra ngoài lướt Internet thông qua một địa chỉ IP công cộng duy nhất (`10.0.0.2`).

## 9. Chính sách Bảo mật mạng (ACL)

Sử dụng **Extended ACL số 100** áp dụng theo chiều ngõ vào (Inbound) trên cổng ảo `G0/1.10` của Router R1:

* **Hạn chế đối với VLAN 10 (STAFF):**
* ❌ Bị chặn hoàn toàn quyền kiểm tra lỗi `ping` (ICMP) sang dải mạng VLAN 20 (Ngăn chặn nhân viên tự ý quét hoặc phá hoại hệ thống máy chủ IT).


* **Quyền hạn của VLAN 10 (STAFF):**
* ✅ Được phép truy cập ra mạng Internet bên ngoài (Ví dụ: lướt Web/Ping tới `8.8.8.8`).
* ✅ Được phép gửi và nhận gói tin xin cấp IP từ DHCP Server bình thường.



## 10. Kịch bản Kiểm thử & Nghiệm thu (Verification)

* **Kiểm tra DHCP:** Các máy tính ở VLAN 10 nhận chính xác địa chỉ IP tự động dạng `192.168.10.x` nhờ tính năng DHCP Relay Agent trên Router.
* **Kiểm tra OSPF:** Gõ lệnh `show ip route` trên Router R1 hiển thị các đường mạng có ký tự **`O`** ở đầu (đường mạng học được từ OSPF của Router R2).
* **Kiểm tra Port Security:** Rút dây mạng của PC0 ra, cắm vào một Laptop lạ. Khi Laptop lạ phát tín hiệu ping, cổng Switch lập tức chuyển sang trạng thái sập nguồn (Đèn cổng biến thành màu ĐỎ).
* **Kiểm tra Tường lửa ACL:** Đứng từ PC0 gõ lệnh `ping 192.168.20.10` (Server) sẽ báo lỗi `Destination host unreachable`. Tuy nhiên, gõ `ping 8.8.8.8` thì nhận được phản hồi `Reply` thành công.

## 11. Kết luận

Bài lab mô phỏng thành công một mô hình thiết kế mạng doanh nghiệp thực tế, tối ưu hóa chi phí phần cứng. Dự án làm nổi bật khả năng triển khai dịch vụ mạng tập trung (DHCP Relay), đảm bảo tính sẵn sàng cao ở tầng vật lý (STP dự phòng), tự động hóa bản đồ đường đi (OSPF), và thực thi các chính sách an ninh nghiêm ngặt ở cả tầng mạng vật lý lẫn luận lý (Port Security & ACLs).
