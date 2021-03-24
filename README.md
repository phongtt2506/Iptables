# Iptables là gì?
**Iptables** là một hệ thống tường lửa (Firewall) tiêu chuẩn được cấu hình, tích hợp mặc định trong hầu hết các bản phân phối của hệ điều hành Linux (CentOS, Ubuntu…). **Iptables** hoạt động dựa trên việc phân loại và thực thi các package ra/vào theo các quy tắc được thiết lập từ trước.

iptables đơn giản chỉ là một danh sách các rules được tổ chức theo dạng bảng. Mỗi một rule chứa một loạt các classifiers (iptables matches) và một connected action (iptables target).
Có 5 hook netfilter nằm bên trong Linux Kernal, nó cho phép kernel modules thực hiện các tác vụ đối với network stack.
Năm hooks như sau:
- NF_IP_PRE_ROUTING: Hook này sẽ được kích hoạt bởi bất kỳ lưu lượng truy cập đến nào ngay sau khi vào ngăn xếp mạng. Hook này được xử lý trước khi bất kỳ quyết định định tuyến nào được đưa ra liên quan đến nơi gửi gói.
- NF_IP_LOCAL_IN: Hook này được kích hoạt sau khi gói đến được định tuyến nếu gói được định sẵn cho hệ thống cục bộ.
- NF_IP_FORWARD: Hook này được kích hoạt sau khi gói đến được định tuyến nếu gói được chuyển tiếp đến máy chủ khác.
- NF_IP_LOCAL_OUT: Hook này được kích hoạt bởi bất kỳ lưu lượng truy cập ngoài được tạo cục bộ nào ngay khi nó chạm vào ngăn xếp mạng.
- NF_IP_POST_ROUTING: Hook này được kích hoạt bởi bất kỳ lưu lượng đi hoặc chuyển tiếp nào sau khi định tuyến đã diễn ra và ngay trước khi được đưa ra trên dây.

# 1. Các khái niệm

Trước tiên, ta sẽ xem danh sách một số rules của iptables cơ bản trên 1 server (CentOS)
Run: *iptables -L -v*

````
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
````

Chúng ta có thể thấy các Chains: INPUT, FORWARD, OUTPUT. Mỗi chain gồm các cột: target, prot, opt, source, destination. Vậy ý nghĩa của chúng là gì?

Trong iptables được chia thành 3 table:
* Table mangle: Bảng này để thay đổi QoS (Quality of Service) của gói tin như thay đổi các tham số TTL, TOS,.. Nó có các chains là : FORWARD, INPUT, OUTPUT, POSTROUTING, và PREROUTING.
* Table NAT: dùng để thay đổi địa chỉ nguồn, địa chỉ đích, port nguồn, port đích của gói tin. Nó có 3 chains là: OUTPUT, POSTROUTING, và PREROUTING.
* Table filter: dùng để áp dụng các policy lọc gói tin qua đó cho phép gói tin vào, qua và ra khỏi hệ thống. Nó bao gồm các chains: FORWARD, INPUT, và OUTPUT.

**- Chain:** Mặc định mỗi table sẽ có các chain, mỗi chain sẽ có policy quyết định trạng thái của gói tin. Policy sẽ có 2 trạng thái là ACCEPT hoặc DROP, mặc định là ACCEPT.

Iptables được định nghĩa trong 5 chain trong quá trình xử lí gói tin: PREROUTING, INPUT, FORWARD, POSTROUTING và OUTPUT. Khi tạo 1 rule thì sẽ được gán vào các Chain này.
- Chain INPUT: áp dụng cho các kết nối đi vào.
- Chain FORWARD: áp dụng cho các kết nối đã được trỏ đến một vị trí khác.
- Chain OUTPUT: áp dụng cho các kết nối ra ngoài từ máy chủ.
- Chain PREROUTING: vừa mới tiến vào từ network interface. Nó sẽ được thực thi trước khi quá trình routing diễn ra, thường dùng cho DNAT (destination NAT)
- Chain POSTROUTING: đi ra ngoài hoặc được forward sau khi quá trình routing hoàn tất, chỉ trước khi nó tiến vào đường truyền, thường dùng cho SNAT (source NAT)

**- Rule:** Bao gồm 1 hoặc nhiều quy tắc để xác định packet nào sẽ chịu ảnh hưởng và target (hành động) được thực thi vào packet đó.
- *Như vậy cấu trúc của 1 ip tables như sau: Iptables > Table > Chains > Rules.*

**- Target:** Là hành động cho packet khi tạo rule. 
* ACCEPT: chấp nhận và cho phép gói tin đi vào hệ thống.
* DROP: loại gói tin, không có gói tin trả lời.
* REJECT: loại gói tin những có trả lời table gói tin khác. Ví dụ: trả lời table 1 gói tin “connection reset” đối với gói TCP hoặc “destination host unreachable” đối với gói UDP và ICMP.
* LOG: chấp nhận gói tin nhưng có ghi lại log.

	*Packet sẽ được đi qua tất cả các rules đặt ra mà không dừng lại ở bất kì rule nào đúng. Trường hợp gói tin không khớp với rules nào mặc định sẽ được chấp nhận.*

**- prot:** Là viết tắt của chữ Protocol, nghĩa là giao thức. Tức là các giao thức sẽ được áp dụng để thực thi quy tắc này. Ở đây chúng ta có 3 lựa chọn là all, tcp hoặc udp. Các ứng dụng như SSH, FTP, sFTP,..đều sử dụng giao thức kiểu TCP.

**- in:** chỉ ra rule sẽ áp dụng cho các gói tin đi vào từ interface nào, ví dụ lo, eth0, eth1 hoặc any là áp dụng cho tất cả interface.

**- out:** tương tự  IN, chỉ ra rule sẽ áp dụng cho các gói tin đi ra từ interface nào.

**- destination:** địa chỉ của lượt truy cập được phép áp dụng quy tắc. 

# 2. Các tùy chọn
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

- Thao tác với rule trong Iptables.
	* Thêm rule: -A (append)
	* Xóa rule: -D (delete)
	* Thay thế rule: -R (replace)
	* Chèn thêm rule: -I (insert)

# 3. Cách hoạt động iptables
Iptables hoạt động bằng cách so sánh packet với một danh sách các rules. Rule định nghĩa các tính chất mà packet cần có để match với rule kèm theo những hành động sẽ được thực thi với những matching packets.




# 3. Các lệnh cơ bản trong Iptables.
## - Tạo 1 rule mới
```
iptables -A INPUT -i lo -j ACCEPT
```
Kết quả sau khi tạo rule:
````
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  lo     any     anywhere             anywhere
````
## - Xóa 1 rule
```
iptables -D INPUT <dòng cần xóa>
```
## - Thêm 1 rule vào 1 dòng tùy ý
```
iptables -I INPUT <dòng muốn thêm rule> <options>
```
## - Lưu và khởi động lại iptable để apple các thay đổi
```
service iptables save
service iptables restart
```

# 4. Ví dụ.
Lưu ý: Nếu đang sử dụng CentOS thì trước tiên disable firewalld theo command sau:
```
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```
**Cài đặt iptables-service:**

```
sudo yum install -y iptables-services
sudo systemctl start iptables
sudo systemctl status iptables
sudo systemctl enable iptables
````
Các thiết lập iptables cơ bản với 1 server
````
sudo iptables -F     #xóa toàn bộ iptables hiện có. Nên hãy lưu ý khi dùng lệnh này.
sudo iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP  	
sudo iptables -A INPUT -p tcp ! --syn -m state --state NEW -j DROP
sudo iptables -A INPUT -p tcp --tcp-flags ALL ALL -j DROP
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -I INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -P OUTPUT ACCEPT		#Chặn toàn bộ INPUT.
sudo iptables -P INPUT DROP		#Cho phép toàn bộ OUTPUT.
sudo iptables-save | sudo tee /etc/sysconfig/iptables  	#Lưu lại cấu hình iptables được chỉnh sửa trong session hiện tại.
sudo systemctl restart iptables
sudo iptables -L -n
````

Ví dụ, thêm Rule chặn ping tới server.
Khi chưa tạo rule, tất cả các host đều có thể ping tới server.
````
PING 192.168.100.200 (192.168.100.200) 56(84) bytes of data.
64 bytes from 192.168.100.200: icmp_seq=1 ttl=64 time=0.201 ms
64 bytes from 192.168.100.200: icmp_seq=2 ttl=64 time=0.233 ms
64 bytes from 192.168.100.200: icmp_seq=3 ttl=64 time=0.280 ms
64 bytes from 192.168.100.200: icmp_seq=4 ttl=64 time=0.163 ms
64 bytes from 192.168.100.200: icmp_seq=5 ttl=64 time=0.245 ms
````
Thêm rule chặn ping bằng các dòng lệnh sau:
````
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j REJECT | DROP
sudo service iptables save
````
Kiểm tra sau khi đã thêm rule với target REJECT:
````
PING 192.168.100.200 (192.168.100.200) 56(84) bytes of data.
From 192.168.100.200 icmp_seq=1 Destination Port Unreachable
From 192.168.100.200 icmp_seq=2 Destination Port Unreachable
From 192.168.100.200 icmp_seq=3 Destination Port Unreachable
From 192.168.100.200 icmp_seq=4 Destination Port Unreachable
From 192.168.100.200 icmp_seq=5 Destination Port Unreachable
````
	*Với target DROP thì sẽ không có phản hồi từ server.*


Bổ sung rule, cho 1 host có thể ping tới server
- Kiểm tra lại rules iptables.
````
iptable -L -v

Chain INPUT (policy ACCEPT 6 packets, 1206 bytes)
 pkts bytes target     prot opt in     out     source               destination
  540 45360 REJECT     icmp --  any    any     anywhere             anywhere             icmp echo-request reject-with icmp-port-unreachable
````
*Output trên: Tất cả các host sẽ không ping được server.*

- Thêm rule để 1 host có thể ping tới server bằng lệnh sau.
````
iptables -I INPUT 2 -s <host> -p icmp --icmp-type echo-request -j ACCEPT
````
*chỉ số 2 trong rule là chèn rule vào dòng 2. Nếu không thêm chỉ số sẽ mặc định được thêm vào dòng đầu tiên.*

- Kiểm tra lại Rule
````
Chain INPUT (policy ACCEPT 8 packets, 1608 bytes)
 pkts bytes target     prot opt in     out     source               destination
    5   420 ACCEPT     icmp --  any    any     10.11.200.185        anywhere             icmp echo-request
  740 62160 REJECT     icmp --  any    any     anywhere             anywhere             icmp echo-request reject-with icmp-port-unreachable
````
Như vậy host 10.11.200.185 sẽ có thể ping tới server, các host còn lại thì không.
````
PING 192.168.100.200 (192.168.100.200) 56(84) bytes of data.
64 bytes from 192.168.100.200: icmp_seq=1 ttl=64 time=0.211 ms
64 bytes from 192.168.100.200: icmp_seq=2 ttl=64 time=0.253 ms
64 bytes from 192.168.100.200: icmp_seq=3 ttl=64 time=0.219 ms
64 bytes from 192.168.100.200: icmp_seq=4 ttl=64 time=0.264 ms
64 bytes from 192.168.100.200: icmp_seq=5 ttl=64 time=0.271 ms
64 bytes from 192.168.100.200: icmp_seq=6 ttl=64 time=0.272 ms
````

## Ví dụ khác: ##

- Cho phép user dùng SMTP Server qua port mặc định là 25 và 465:
```
sudo iptables I INPUT -p tcp -m tcp --dport 25 -j ACCEPT
sudo iptables -I INPUT -p tcp -m tcp --dport 465 -j ACCEPT
```
Hoặc với cùng dịch vụ ta có thể mở multiport theo cú pháp sau:
```
sudo iptables -I INPUT -p tcp -m multiport --dport 25,465 -j ACCEPT
```
- Mở port web server 80/443
```
sudo iptables -I INPUT -p tcp -m tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT -p tcp -m tcp --dport 443 -j ACCEPT
```

**Sau khi đã thiết lập đầy đủ, bao gồm mở các port cần thiết hay hạn chế các kết nối, bạn cần block toàn bộ các kết nối còn lại và cho phép toàn bộ các kết nối ra ngoài từ server.**
```
sudo iptables -P OUTPUT ACCEPT		#Chặn toàn bộ INPUT.
sudo iptables -P INPUT DROP		#Cho phép toàn bộ OUTPUT.
```

Kiểm tra lại các rules:
```
sudo iptables -L -v hoặc sudo iptables -L -n
```
Lưu iptables:
```
sudo service iptables save
```