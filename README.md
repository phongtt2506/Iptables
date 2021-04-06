# Iptables là gì?
**Iptables** là một hệ thống tường lửa (Firewall) tiêu chuẩn được cấu hình, tích hợp mặc định trong hầu hết các bản phân phối của hệ điều hành Linux (CentOS, Ubuntu…). **Iptables** hoạt động dựa trên việc phân loại và thực thi các package ra/vào theo các quy tắc được thiết lập từ trước.

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
# Các lệnh cơ bản trong Iptables.
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

# Ví dụ.
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

- Thêm rule để host có ip 192.168.100.100 có thể ping tới server bằng lệnh sau.
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
Như vậy host 192.168.100.100 sẽ có thể ping tới server, các host còn lại thì không.
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
Xem thêm nhiều cách defined rules tại đây: https://linux.die.net/man/8/iptables
