Netkit lab MPLS VPN
======
Chắc hẳn những ai đọc bài viết này của tôi cũng đã dùng qua Packet tracer hoặc GNS3. Hôm nay tôi xin giới thiệu đến các bạn một công cụ khác đó là Netkit.

Mục đích của bài viết sẽ là giới thiệu cho các bạn bộ công cụ này, và thông qua một bài lab MPLS VPN sẽ hướng dẫn cho các bạn một phần nào đó cách thực hiện một bài Lab về mạng bằng Netkit.

## 1. Giới thệu bộ công cụ Netkit

#### 1.1. Khái niệm

- Netkit là một bộ công cụ cho phép giả lập và vận hành thử nghiệm hệ thống mạng với chi phí thấp. Nó cho phép tạo ra các thiết bị mạng ảo như Router, Switch, PC,… dễ dàng kết nối với nhhau chỉ trên một host.

- Chức năng giống GNS3, Packet Tracer,…

#### 1.2. Đặc điểm

- Dựa trên nhân Linux

- Cần xây dựng một mô hình mạng cụ thể từ trước để thuận tiện cho việc đấu nối các thiết bị

- Mỗi bài lab nằm trong một thư mục, các file cấu hình, các thiết bị đều nằm trong đó

- Mọi công việc chuyển mạch, định tuyến đều thực hiện trên Linux

#### 1.3. Các thành phần

- Một file lab.conf mô tả mô hình mạng

- Một thư mục chứa các file của mỗi thiết bị giả lập

- Các file .startup và .shutdown mô tả hành động được thực hiện bởi các máy ảo khi chúng được khởi động hoặc tắt

- Một file lab.dep mô tả sự phụ thuộc quan hệ khi khởi động của các thiết bị ảo (có thể có hoặc không)

- Một thành phần không bắt buộc nữa của netkit đó là thư mục _test chứa đoạn mã để kiểm tra bài lab có thực hiện đúng hay không.

#### 1.4. Cài đặt

##### Môi trường:

- Máy ảo Ubuntu 12.04 Server x64

##### Cài đặt

SSH vào máy ảo Ubuntu bằng tài khoản root

Tải các gói cần thiết

    wget http://wiki.netkit.org/download/netkit/netkit-2.8.tar.bz2
    wget http://wiki.netkit.org/download/netkit-filesystem/netkit-filesystem-i386-F5.2.tar.bz2
    wget http://wiki.netkit.org/download/netkit-kernel/netkit-kernel-i386-K2.8.tar.bz2

Cài đặt các gói

    apt-get update -y
    apt-get install -y xorg openbox
    apt-get install -y ia32-libs  
    apt-get install -y libc6-i386
    
Xả nén

    tar -xvf netkit-2.8.tar.bz2
    tar -xvf netkit-filesystem-i386-F5.2.tar.bz2
    tar -xvf netkit-kernel-i386-K2.8.tar.bz2

Sửa file /root/.profile và thêm vào các dòng sau:
    
    export NETKIT_HOME=~/netkit
    export MANPATH=:$NETKIT_HOME/man
    export PATH=$NETKIT_HOME/bin:$PATH
    . $NETKIT_HOME/bin/netkit_bash_completion
    
Log out ssh và mở Xming, ssh lại vào máy chủ Ubuntu để phiên ssh mới nhận biến môi trường.
SSH với Xming có thể tham khảo [tại đây] (http://ducnc.blogspot.com/2014/08/remote-desktop-ubuntu-su-dung-vnc.html)

Kiểm tra xem đã cài đặt netkit thành công hay chưa

    cd $NETKIT_HOME
    ./check_configuration.sh

Nếu kết quả như hình sau là OK

<img src=http://i.imgur.com/Rj3Vqo9.png>

Trong phần tiếp theo tôi sẽ trình bày về một bài lab sử dụng netkit

## 2. Bài lab MPLS VPN với netkit

##### Mô hình triển khai

<img src=http://i.imgur.com/3oP4aOd.png>

Trong demo này tôi xây dựng hai VPN có địa chỉ giống nhau
- VPN 1 (VPN màu xanh dương) nối hai mạng 192.168.10.0/24 và 192.168.30.0/24 thông qua tuyến đường là các router A1 – E2 – E3 – E1 – A3
- VPN 2 (VPN màu xanh lá) nối hai mạng 192.168.10.0/24 và 192.168.30.0/24 thông qua tuyến đường tuyến đường A2 – E4 – E3 – E1 – A4
- Các router có nhiệm vụ định tuyến cho gói tin trong cùng VPN đến đúng địa chỉ mặc dù các mạng có địa chỉ giống nhau.

##### Một số khái niệm liên quan

- CE Router (Customer Edge Router): Router của khách hàng có nhiệm vụ định tuyến trong nội bộ doanh nghiệp và định tuyến ra mạng của ISP.
- PE Router (Provider Edge Router): Router biên của nhà cung cấp, kết nối trực tiếp với khách hàng, có nhiệm vụ gắn nhãn và bóc nhãn cho các kết nối VPN của khách hàng, định tuyến tới các Router khác trong mạng ISP
- Provider Router: Các Router của ISP có nhiệm vụ chuyển mạch nhãn.
- NHLFE (Next Hop Label Forwarding Entry): chứa thông tin làm cách nào để chuyển gói tin trong MPLS, bao gồm IP next hop và cách xử lý các nhãn (push, pop) trong stack.
- ILM (Incoming Label Map): nối nhãn từ gói tin chuyển đến với một NHLFE, có thể hiểu là cách xử lý một gói tin được chuyển tới.
- XC (Cross Connect): nối mỗi ILM với một NHLFE, có thể hiểu rằng nó nói cho router biết cách đổi nhãn như thế nào.

##### Phân tích quá trình kết nối trao đổi thông tin

Kết nối từ Router A1 đến Router A3 (VPN 1)

<img src=http://i.imgur.com/RxsORcW.png>

- A1 gửi một gói tin ICMP Echo Request tới địa chỉ A3
- Gói tin đi tới E2. E2 kiểm tra bảng định tuyến của nó và biết rằng gói tin này thuộc VPN 1 sau đó nó sẽ gán hai nhãn: một “inner label” 100 để xác định là VPN 1 và một “outer label” 1000 để định tuyến trong mạng MPLS
- E3 chuyển “outer label” thành 3000 và chuyển gói tin đến E1, “inner label” vẫn giữ nguyên.
- E1 sẽ có hai bảng định tuyến. Bảng 1 sẽ dùng để lưu thông tin về các tuyến đường đến các địa chỉ của VPN 1, bảng 2 là của VPN 2. Khi E1 nhận được gói tin từ E3 gửi đến, nó bóc nhãn đầu tiên (3000) và biết rằng nhãn thứ 2 (100) là của VPN 1, nó sẽ bỏ luôn nhãn này đi và tìm kiếm trong bảng 1 để chuyển gói tin đến A3. Gói tin lúc này là chính gói tin ban đầu mà A1 gửi đi.

<img src=http://i.imgur.com/UBV6TyC.png>

- Gói ICMP Reply được gửi từ A3 đến A1.
- E1 nhận được gói tin từ A1 gửi đến, nó căn cứ vào địa chỉ đích của gói tin và interface nhận gói tin (eth1) biết rằng đây là gói tin thuộc VPN 1, nó tìm trong bảng 1 để biết cách xử lý gói tin này. Sau đó E1 sẽ gán hai nhãn 100 và 4000 cho gói tin đó và chuyển đến E3 qua interface eth3.
- Tại E3 nhãn 4000 được thay bằng nhãn 4001 và chuyển tiếp tới E2.
- Tại E2 hai nhãn 4001 và 100 được bóc ra và gói tin ban đầu được chuyển đến A1.

Kết nối từ A2 đến A4 (VPN 2) cũng tương tự như kết nối của VPN1. Do tại mỗi Router căn cứ vào interface nhận gói tin và địa chỉ đích rồi sau đó mỗi gói mới được gắn nhãn cho đúng với MPLS VPN nên việc trùng lặp IP của hai VPN không thành vấn đề. Việc kết nối vẫn diễn ra bình thường mặc cho địa chỉ là như thế nào.

##### Thực hiện bài lab

Mô hình kết nối và địa chỉ của các Router

<img src=http://i.imgur.com/VKV0mnM.png>

Mô hình này được ánh xạ vào file cấu hình lab.conf trong github

Mở Xming và ssh vào máy ảo Ubuntu

Thực hiện các lệnh sau:

    apt-get install git -y
    git clone https://github.com/ducnc/Netkit-MPLS-VPN.git
    cd Netkit-MPLS-VPN/
    lstart

Lúc này Xming sẽ bật lên các cửa sổ tương ứng với mỗi router như hình sau:
<img src=http://i.imgur.com/gdSdPJR.png>

Kiểm tra sự hoạt động của VPN1. 

Từ A1 ta ping đến địa chỉ 192.168.30.30 (A3)

Trên A3 và A4 ta sử dụng lệnh tcpdump để theo dõi

<img src=http://i.imgur.com/OwcQlLZ.png>

Ta thấy trên A1 ping đến địa chỉ 192.168.30.30 thành công và trên A3 bắt được những gói ICMP này, A4 thì không bắt được gì cả.
Chứng tỏ VPN đã thực hiện đúng mặc dù bị trùng địa chỉ IP.

Nếu ta trên A2 ping 192.168.30.30 thì trên A4 cũng sẽ bắt được những gói tin này, còn A3 thì không.
Chứng tỏ hai VPN đều thực hiện đúng.

##### Lưu ý:

Để có thể quan sát dễ dàng hơn các gói tin được truyền qua các card mạng trên mỗi router ta thực hiện lệnh sau (ví dụ ở đây là router E1 card eth1)

    tcpdump -i eth1 -w /hosthome/E1-eth1.pcap

Lúc này tại thư mục /root/ của máy ảo Ubuntu sẽ có một file E1-eth1.pcap. Ta có thể copy file này ra máy thật và sử dụng wireshark để quan sát các gói tin
    
## 3. Lời cảm ơn

Cảm ơn các bạn đã đọc hết bài viết này và xin ghi nhận mọi ý kiến đóng góp

Liên hệ:
- Skype: khong_giong_ai
- Facebook: https://www.facebook.com/nguyencongduc
