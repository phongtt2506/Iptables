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
- Chỉ định thông số Ipthables.
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
