---
title: "Virus scan with ClamAV | Docker setup guide"
date: 2020-10-24T12:58:10+09:00
draft: false
tags: [rails, clamav, security]
language: vietnamese
toc: true
authors: [chienkira]
cover: /blog/images/clamav.png
---

**Có lẽ hơi low-tech nên đợt vừa rồi mình mới lần đầu phải làm 1 hệ thống mà có yêu cần scan virus cho file. Cụ thể là sau khi được upload lên, phía server phải scan xem file có nhiễm virus không, nếu không thì mới đọc file và bắt đầu xử lý import vân vân mây mây... Thấy khá hay ho và là lạ nên bài viết này mình sẽ memo lại solution và hướng dẫn setup docker để dev nhé!**

## ClamAV?

Câu hỏi bật lên đầu tiên là, Ủa scan virus bằng cái gì trong khi server chạy linux?
Hiển nhiên là thắc mắc rồi, server chứ có như máy tính của chúng ta đâu mà cài vào Avast với chả BKAV... :))

Nhưng mà đừng lo, có software cả đó yên tâm nhé!

Đã có [ClamAV](https://www.clamav.net/) - Open source và hỗ trợ scan file bằng command line!
![](https://www.clamav.net/assets/clamav-trademark.png)

Theo như đồn đại của thiên hạ, đa số các tính năng quét virus trên các hệ thống lớn cũng đều đang sử dụng ClamAV.
Database virus cũng được cập nhật đều đặn nên các bạn cứ tự tin mà sử dụng ClamAV nhé.

Sau khi cài đặt xong, có 2 cách để sử dụng ClamAV.

1. One-time scanning

    - Sử dụng command `clamscan` để khởi động ClamAV và scan file
    - Không yêu cầu chạy daemon, không phát sinh persistent process tiêu hao tài nguyên server
    - Bù lại, thời gian scan file thì nhanh mà thời gian khởi động lên quá lâu (vài chục s) nên áp dụng vào web service thì không thực tế

2. Daemon scanning

    - Sử dụng command `clamdscan` để bảo daemon đang chạy sẵn scan 1 file
    - Yêu cầu daemon chạy ngầm bên dưới nên lúc nào cũng xơi khoảng 1GB memory của server :))
    - Thời gian từ lúc yêu cầu scan file và nhận kết quả rất nhanh! (dưới 1s)

Như mọi người thấy, nếu không phải mục đích là vọc chơi thử thì chúng ta có thể quên luôn cái one-time scanning.

Tranh thủ giới thiệu luôn thì ClamAV gồm có 2 daemon hoạt động chính.

1. clamd

    Đây là daemon để scan file cho chúng ta, lúc nào cũng xơi khoảng 1GB memory - hình như ClamAV nó nạp hết database virus vào memory thì phải :thinking:

2. freshclam

    Đây là daemon sẽ update database virus, đảm bảo scan không bỏ sót con "corona" nào nhé :))

## Sử dụng ClamAV từ ứng dụng Rails?

Để sử dụng ClamAV từ trong 1 app Rails không có gì khó cả.
Ruby có method `system` để bảo server chạy 1 command tùy ý, vì thế cách chân phương nhất có thể nghĩ đến là,
tự chắp ghép sinh ra câu lệnh để scan file rồi dùng method `system` để execute.

Kiểu kiểu như là

```
command = "clamdscan -c /etc/clamd.d/scan.conf #{file.path}"
system command
```

Tuy nhiên đoạn sinh ra câu lệnh cũng xấu xí, ngoài ra phải parse output của lệnh clamdscan để xác định kết quả là có virus hay không cũng lằng nhằng nên recommend mọi người gem [Clamby](https://github.com/kobaltz/clamby) luôn cho nhàn.

Ví dụ

```
scan_result = Clamby.safe?(path)
if scan_result
  return true
else
  File.delete(path)
  return false
end
```

## Docker setup?

Cuối cùng chia sẻ mọi ng cách cài đặt docker để sử dụng ClamAV vào development environment. :smiley:

Mình hay sử dụng alpine linux cho nhẹ, mà alpine linux mặc định chưa có setup để chạy daemon service 
nên việc đầu tiên là cài đặt cái gọi là openrc.

Đọc cũng không hiểu lắm nhưng đại ý openrc bản thân mặc định nó không chạy được trong docker container (server thật thì ok)
nên phải thêm kha khá lệnh để fix - góp nhặt trên mạng về.
Dưới đây là đoạn code cuối cùng mà giúp mình install openrc thành công.

*__Dockerfile__*
```
# Install deamon service for alpine linux
RUN apk add openrc &&\
# Tell openrc its running inside a container, till now that has meant LXC
    sed -i 's/#rc_sys=""/rc_sys="lxc"/g' /etc/rc.conf &&\
# Tell openrc loopback and net are already there, since docker handles the networking
    echo 'rc_provide="loopback net"' >> /etc/rc.conf &&\
# no need for loggers
    sed -i 's/^#\(rc_logger="YES"\)$/\1/' /etc/rc.conf &&\
# can't get ttys unless you run the container in privileged mode
    sed -i '/tty/d' /etc/inittab &&\
# can't set hostname since docker sets it
    sed -i 's/hostname $opts/# hostname $opts/g' /etc/init.d/hostname &&\
# can't mount tmpfs since not privileged
    sed -i 's/mount -t tmpfs/# mount -t tmpfs/g' /lib/rc/sh/init.sh &&\
# can't do cgroups
    sed -i 's/cgroup_add_service /# cgroup_add_service /g' /lib/rc/sh/openrc-run.sh &&\
# Tell openrc its ok to start
    mkdir -p /run/openrc && touch /run/openrc/softlevel && rc-status
```

Sau đó thêm step để install clamav thôi.

*__Dockerfile__*
```
# Install clamav and clamd deamon service
VOLUME [ "/sys/fs/cgroup" ]
RUN apk add clamav clamav-libunrar unrar && freshclam
```

*chú ý: freshclam cần dùng unrar nên cài nó cùng luôn ở đây.*

Vậy là xong setup cho docker image.
Bây giờ trong docker-compose.yml các bạn có thể thêm vào command để khởi chạy daemon clamd mỗi khi start container.
Các bạn có thể tham khảo như sau

*__docker-compose.yml__*
```
...
container_name: assessment_all_web
command: >
  sh -c " nohup rc-service clamd restart &> tmp/clamd.nohup.out &
          rm -f tmp/pids/server.pid;
          bundle install --jobs 4 &&
          bundle exec rails s -b 0.0.0.0"
depends_on:
...
```

*chú ý: nohup để việc start deamon cũng chạy ngầm ở dưới, tránh block làm chậm các lệnh đằng sau*

Cơ mà vì deamon khá nặng, xơi của người ta xấp xỉ 1GB memory nên mình recommend lúc nào cần dùng hãy khởi động nó lên thôi.
Thời gian đầu mình cũng config vào docker-compose.yml như vậy nhưng nhận ra là nặng quá mà không phải lúc nào cũng dev tính năng cần xài ClamAV nên về sau lại gỡ khỏi docker-compose.yml, để lại hướng dẫn chạy service vào Readme cho ai cần thì chạy thôi.