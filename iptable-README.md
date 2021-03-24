1. Iptables là gì?
**Iptables** là một hệ thống tường lửa (Firewall) tiêu chuẩn được cấu hình, tích hợp mặc định trong hầu hết các bản phân phối của hệ điều hành Linux (CentOS, Ubuntu…). **Iptables** hoạt động dựa trên việc phân loại và thực thi các package ra/vào theo các quy tắc được thiết lập từ trước.

2. Các khái niệm

Trước tiên, ta sẽ xem danh sách một số rules của iptables cơ bản trên 1 server (CentOS)
Run: *iptables -L -v*

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
2194K  615M ACCEPT     all  --  any    any     anywhere             anywhere             ctstate RELATED,ESTABLISHED
 1643 98496 ACCEPT     all  --  lo     any     anywhere             anywhere
94433   14M INPUT_direct  all  --  any    any     anywhere             anywhere
94433   14M INPUT_ZONES_SOURCE  all  --  any    any     anywhere             anywhere
94433   14M INPUT_ZONES  all  --  any    any     anywhere             anywhere
    0     0 DROP       all  --  any    any     anywhere             anywhere             ctstate INVALID
84828   14M REJECT     all  --  any    any     anywhere             anywhere             reject-with icmp-host-prohibited

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  any    any     anywhere             anywhere             ctstate RELATED,ESTABLISHED
    0     0 ACCEPT     all  --  lo     any     anywhere             anywhere
    0     0 FORWARD_direct  all  --  any    any     anywhere             anywhere
    0     0 FORWARD_IN_ZONES_SOURCE  all  --  any    any     anywhere             anywhere
    0     0 FORWARD_IN_ZONES  all  --  any    any     anywhere             anywhere
    0     0 FORWARD_OUT_ZONES_SOURCE  all  --  any    any     anywhere             anywhere
    0     0 FORWARD_OUT_ZONES  all  --  any    any     anywhere             anywhere
    0     0 DROP       all  --  any    any     anywhere             anywhere             ctstate INVALID
    0     0 REJECT     all  --  any    any     anywhere             anywhere             reject-with icmp-host-prohibited

Chain OUTPUT (policy ACCEPT 2197K packets, 629M bytes)
 pkts bytes target     prot opt in     out     source               destination
2197K  629M OUTPUT_direct  all  --  any    any     anywhere             anywhere


Chúng ta có thể thấy các Chains: INPUT, FORWARD, OUTPUT. Mỗi chain gồm các cột: target, prot, opt, source, destination. Vậy ý nghĩa của chúng là gì?

Trong iptables được chia thành 3 table:
	* Table mangle: Bảng này để thay đổi QoS (Quality of Service) của gói tin như thay đổi các tham số TTL, TOS,.. Nó có các chains là : FORWARD, INPUT, OUTPUT, POSTROUTING, và PREROUTING.
	* Table NAT: dùng để thay đổi địa chỉ nguồn, địa chỉ đích, port nguồn, port đích của gói tin. Nó có 3 chains là: OUTPUT, POSTROUTING, và PREROUTING.
	* Table filter: dùng để áp dụng các policy lọc gói tin qua đó cho phép gói tin vào, qua và ra khỏi hệ thống. Nó bao gồm các chains: FORWARD, INPUT, và OUTPUT.

- Chain: Mặc định mỗi table sẽ có các chain, mỗi chain sẽ có policy quyết định trạng thái của gói tin. Policy sẽ có 2 trạng thái là ACCEPT hoặc DROP, mặc định là ACCEPT.

	Iptables được định nghĩa trong 5 chain trong quá trình xử lí gói tin: PREROUTING, INPUT, FORWARD, POSTROUTING và OUTPUT. Khi tạo 1 rule thì sẽ được gán vào các Chain này.
		* Chain INPUT: áp dụng cho các kết nối đi vào.
		* Chain FORWARD: áp dụng cho các kết nối đã được trỏ đến một vị trí khác.
		* Chain OUTPUT: áp dụng cho các kết nối ra ngoài từ máy chủ.
		* Chain PREROUTING: vừa mới tiến vào từ network interface. Nó sẽ được thực thi trước khi quá trình routing diễn ra, thường dùng cho DNAT (destination NAT)
		* Chain POSTROUTING: đi ra ngoài hoặc được forward sau khi quá trình routing hoàn tất, chỉ trước khi nó tiến vào đường truyền, thường dùng cho SNAT (source NAT)

- Rule: Bao gồm 1 hoặc nhiều quy tắc để xác định packet nào sẽ chịu ảnh hưởng và target (hành động) được thực thi vào packet đó.

*Như vậy cấu trúc của 1 ip tables như sau: Iptables > table > Chains > Rules.*

- target: Là hành động cho packet khi tạo rule. 
	* ACCEPT: chấp nhận và cho phép gói tin đi vào hệ thống.
	* DROP: loại gói tin, không có gói tin trả lời.
	* REJECT: loại gói tin những có trả lời table gói tin khác. Ví dụ: trả lời table 1 gói tin “connection reset” đối với gói TCP hoặc “destination host unreachable” đối với gói UDP và ICMP.
	* LOG: chấp nhận gói tin nhưng có ghi lại log.
	Packet sẽ được đi qua tất cả các rules đặt ra mà không dừng lại ở bất kì rule nào đúng. Trường hợp gói tin không khớp với rules nào mặc định sẽ được chấp nhận.

- prot: Là viết tắt của chữ Protocol, nghĩa là giao thức. Tức là các giao thức sẽ được áp dụng để thực thi quy tắc này. Ở đây chúng ta có 3 lựa chọn là all, tcp hoặc udp. Các ứng dụng như SSH, FTP, sFTP,..đều sử dụng giao thức kiểu TCP.
- in: chỉ ra rule sẽ áp dụng cho các gói tin đi vào từ interface nào, ví dụ lo, eth0, eth1 hoặc any là áp dụng cho tất cả interface.
- out: tương tự  IN, chỉ ra rule sẽ áp dụng cho các gói tin đi ra từ interface nào.
- destination: địa chỉ của lượt truy cập được phép áp dụng quy tắc. 

3. Các tùy chọn
- Chỉ định thông số Iptables.
	* Chỉ định tên table: -t ,
	* Chỉ định loại giao thức: -p ,
	* Chỉ định card mạng vào: -i ,
	* Chỉ định card mạng ra: -o ,
	* Chỉ định địa chỉ IP nguồn: -s <địa_chỉ_ip_nguồn>,
	* Chỉ định địa chỉ IP đích: -d <địa_chỉ_ip_đích>, tương tự như –s.
	* Chỉ định cổng nguồn: –sport ,
	* Chỉ định cổng đích: –dport , tương tự như –sport

- Thao tác với chain trong Iptables.
	* Tạo chain mới: IPtables -N
	* Xóa hết các rule đã tạo trong chain: IPtables -X
	* Đặt chính sách cho các chain `built-in` (INPUT, OUTPUT & FORWARD): IPtables -P , ví dụ: IPtables -P INPUT ACCEPT để chấp nhận các packet vào chain INPUT
	* Liệt kê các rule có trong chain: IPtables -L
	* Xóa các rule có trong chain (flush chain): IPtables -F
	* Reset bộ đếm packet về 0: IPtables -Z

-Thao tác với rule trong Iptables.
	* Thêm rule: -A (append)
	* Xóa rule: -D (delete)
	* Thay thế rule: -R (replace)
	* Chèn thêm rule: -I (insert)

4. Các lệnh cơ bản trong Iptables.
- Tạo 1 rule mới
# iptables -A INPUT -i lo -j ACCEPT

Kết quả sao khi tạo rule:
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  lo     any     anywhere             anywhere

 - Xóa 1 rule
 *iptables -D INPUT <dòng cần xóa>*

 - Thêm 1 rule vào 1 dòng tùy ý
 *iptables -I INPUT <dòng muốn thêm rule> <điều kiện>*

- Lưu và khởi động lại iptable để apple các thay đổi
*service iptables save*
*service iptables restart*

5. Ví dụ.
Lưu ý: Nếu đang sử dụng CentOS thì trước tiên disable firewalld theo command sau:
*sudo systemctl stop firewalld*
*sudo systemctl disable firewalld*

**Cài đặt iptables-service:**

*sudo yum install -y iptables-services
sudo systemctl start iptables
sudo systemctl status iptables
sudo systemctl enable iptables*

Các thiết lập iptables với 1 server
*sudo iptables -F
sudo iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP
sudo iptables -A INPUT -p tcp ! --syn -m state --state NEW -j DROP
sudo iptables -A INPUT -p tcp --tcp-flags ALL ALL -j DROP
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -I INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -P OUTPUT ACCEPT
sudo iptables -P INPUT DROP
sudo iptables-save | sudo tee /etc/sysconfig/iptables
sudo systemctl restart iptables
sudo iptables -L -n*

Lệnh *sudo iptables -F* sẽ xóa toàn bộ iptables hiện có. Nên hãy lưu ý khi dùng lệnh này.

*sudo iptables -P OUTPUT ACCEPT* : Cho phép toàn bộ OUTPUT.

*sudo iptables -P INPUT DROP* : Chặn toàn bộ INPUT.

*sudo iptables-save | sudo tee /etc/sysconfig/iptables* : Lưu lại cấu hình iptables được chỉnh sửa trong session hiện tại.

Ví dụ, thêm Rule chặn ping tới server.
Khi chưa tạo rule:
PING 10.11.200.187 (10.11.200.187) 56(84) bytes of data.
64 bytes from 10.11.200.187: icmp_seq=1 ttl=64 time=0.201 ms
64 bytes from 10.11.200.187: icmp_seq=2 ttl=64 time=0.233 ms
64 bytes from 10.11.200.187: icmp_seq=3 ttl=64 time=0.280 ms
64 bytes from 10.11.200.187: icmp_seq=4 ttl=64 time=0.163 ms
64 bytes from 10.11.200.187: icmp_seq=5 ttl=64 time=0.245 ms

Thêm rule chặn ping bằng các dòng lệnh sau:
*sudo iptables -A INPUT -p icmp --icmp-type echo-request -j REJECT
sudo iptables save*

Kiểm tra sau khi đã thêm rule:

PING 10.11.200.187 (10.11.200.187) 56(84) bytes of data.
From 10.11.200.187 icmp_seq=1 Destination Port Unreachable
From 10.11.200.187 icmp_seq=2 Destination Port Unreachable
From 10.11.200.187 icmp_seq=3 Destination Port Unreachable
From 10.11.200.187 icmp_seq=4 Destination Port Unreachable
From 10.11.200.187 icmp_seq=5 Destination Port Unreachable

Bổ sung rule, cho 1 host có thể ping tới server
- Kiểm tra lại iptable của chain INPUT
*iptable -L -v*
Chain INPUT (policy ACCEPT 6 packets, 1206 bytes)
 pkts bytes target     prot opt in     out     source               destination
  540 45360 REJECT     icmp --  any    any     anywhere             anywhere             icmp echo-request reject-with icmp-port-unreachable
Tất cả các host sẽ không ping được server.

- Thêm rule để 1 host có thể ping tới server.
iptables -I INPUT 1 -s <host> -p icmp --icmp-type echo-request -j ACCEPT

*chỉ số 1 trong rule là chèn rule vào dòng 1.*

- Kiểm tra lại Rule
Chain INPUT (policy ACCEPT 8 packets, 1608 bytes)
 pkts bytes target     prot opt in     out     source               destination
    5   420 ACCEPT     icmp --  any    any     10.11.200.185        anywhere             icmp echo-request
  740 62160 REJECT     icmp --  any    any     anywhere             anywhere             icmp echo-request reject-with icmp-port-unreachable

Như vậy host 10.11.200.185 sẽ có thể ping tới server, các host còn lại thì khong.
PING 10.11.200.187 (10.11.200.187) 56(84) bytes of data.
64 bytes from 10.11.200.187: icmp_seq=1 ttl=64 time=0.211 ms
64 bytes from 10.11.200.187: icmp_seq=2 ttl=64 time=0.253 ms
64 bytes from 10.11.200.187: icmp_seq=3 ttl=64 time=0.219 ms
64 bytes from 10.11.200.187: icmp_seq=4 ttl=64 time=0.264 ms
64 bytes from 10.11.200.187: icmp_seq=5 ttl=64 time=0.271 ms
64 bytes from 10.11.200.187: icmp_seq=6 ttl=64 time=0.272 ms


