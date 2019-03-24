---
title: "Áp Dụng Circle Ci"
date: 2019-03-23T21:40:43+09:00
draft: no
tags: [CI, Circle CI, docker]
language: vi
toc: true
authors: [chienkira]
cover: https://circleci.com/circleci-logo-horizontal-twitter.png
---

**Lúc đầu trang blog này mình định deploy bằng tay vì nó đơn giản, thao tác cũng chẳng có gì - chạy cái shell script xong trong nháy mắt thôi. Nhưng mà tuần này chưa có gì hay ho để viết nên mình quyết định cài đặt CI cho em nó rồi lấy ý để viết bài này giới thiệu về CI luôn.**

# Giới thiệu CI và Circle CI

## CI
*CI viết tắt của Continuous Integration (Tích hợp liên tục)*
Trong quy trình làm phần mềm, lỗi lầm lớn nhất có thể xảy ra không phải là khi developer code ra cái gì tởm lợm! Ta biết nó tởm lợm, ta sửa lại nó vậy là xong. Nhưng một thao tác sai dù nhỏ trong lúc deloy có thể dẫn đến tai nạn khôn lường. Không có hai em gái nào xinh giống hệt được nhau :)) tương tự vậy không có gì đảm bảo rằng trong hàng ngàn hàng vạn lần bạn thực hiện deploy, không có một lỗi lầm thao tác nào xảy ra.

Vì lý do đó, nhà nhà người người hướng đến một cách thức triển khai phần mềm ưu việt hơn. Với lợi thế là an toàn hơn, quá trình lại được tự động hóa, CI (và cả CD nữa) đang là xu thế thực sự. Nói đến đây thì cũng chưa thể rõ CI thực tế nó là cái gì cả phải không ạ? CI nó giống như kiểu design pattern là lý thuyết thôi áp dụng nó ra sao quan trọng với ta hơn. Vậy nên mình sẽ giải thích tiếp thông qua việc giới thiệu tool Circle CI ở ngay bên dưới nhé.

## Circle CI

Như ở trên đề cập, nó là 1 tool để giúp ta hiện thực hóa CI. Có nhiều tool CI khác cũng nổi tiếng nữa (Travis CI, Jenkins...), lý do mình chọn Circle CI chỉ đơn giản vì trên công ty làm với nó nhiều rồi nên có kinh nghiệm thôi.

Bản chất Circle CI là sử dụng docker, trong cấu hình Circle CI ta sẽ chỉ định các docker `image` sẽ sử dụng và các `job`, trong các job lại có các `step`, trong các step là cụ thể các `command`. Ngoài ra còn có cấu hình `filter` giúp ta linh hoạt điều chỉnh sao cho chỉ run các job khi có merge/push vào 1 số branch nhất định vân vân.

Mô tả quá trình run 1 job trên Circle CI:

1. Developer chỉ cần push hoặc merge vào 1 branch, Circle CI tự động biết event đó và khởi động lên job đã được cài đặt tương ứng.
2. Ban đầu Circle CI pull docker image về và run lên trên môi trường cloud của nó
3. Tiếp theo nó chạy các `step` đã được cài đặt trong docker container, thông thường step đầu tiên luôn là `checkout` tức là git checkout lấy source về (mặc định lưu trong thư mục `~/project`)
4. Các step tiếp theo được chạy tùy vào độ sáng tạo của bạn, ví dụ job để build thì thường là `npm install` rồi `npm run abcxyz` hay job để deploy thì có thể là `aws s3 sync` hay `serverless deploy`...
5. Sau khi tất cả các step đã chạy xong, job kết thúc. Nếu exit code của job là error thì mặc định ta sẽ nhận được mail thông báo failed nữa.

Nói tóm lại, sau khi cài đặt và cấu hình ta chỉ việc dev còn các công việc như build, chạy test, deploy vân vân được tự động hóa hoàn toàn và chạy tức thì trên môi trường cloud mạnh mẽ miễn phí của Circle CI.

# Áp dụng Circle CI cho chienkira/blog

Sau đây mình sẽ sử dụng Circle CI để cài đặt CI giúp mình tự động deploy và publish trang blog mỗi khi có push thay đổi gì đó lên github.

*※ Blog của mình viết bằng hugo, publish thông qua github pages*

## Chọn docker image để làm việc

Đầu tiên là phải tìm em docker image ngon ngon để xài.

Chúng ta nên tránh lấy mấy image offical hào nhoáng kiểu linux hay node vân vân. Bọn nó xịn xò thật, tin tưởng được thật nhưng mà "nguyên thủy" thấy ghê. Nếu dùng mấy image đó thì để dùng được việc ta lại phải cài thêm ông A anh B, dẫn đến mỗi job trên Circle CI chỉ riêng đoạn pull docker image xong install các tool thì cũng đến mấy phút rồi... :scream:

Kinh nghiệm của mình là lên thẳng [docker hub](https://hub.docker.com/) rồi tìm cái image nào có sẵn các hàng mình cần kéo về xài. Cẩn thận hơn thì check qua dockerfile của nó xem base từ image nào, run các lệnh gì cho đảm bảo.

Yêu cầu của image lần này là phải có hugo và git. (nhiều image trên thị trường ko có git nên phải cẩn thận, ko có git là lệnh `checkout` của Circle CI chết ngay :hand:)

Ok search thôi, từ khóa 'hugo git' và link trang kết quả search đây: https://hub.docker.com/search?q=hugo%20git&type=image.  
Image đầu tiên trong danh sách kết quả mình nhìn thấy là ưng cái bụng liền.

> Container with Hugo, Git & Bash installed. Made to work with wercker. 

Check qua thử docker file, base là `alpine` => rất nhẹ nữa. Ok để chắc nữa thì mình sẽ pull image này về trên máy check hàng em nó!

* Run em nó: 
    ```bash
    docker run --rm -it andthensome/alpine-hugo-git-bash /bin/sh
    ```

* Vào trong em nó rồi thì check hàng:
    ```bash
    / # git --version
    git version 2.13.5
    / # hugo version
    Hugo Static Site Generator v0.31.1 linux/amd64 BuildDate: 2017-11-27T11:26:24Z
    ```

Cái version hugo hơi cũ chắc phải upgrade lên nhưng mà tổng thể có vẻ ổn rồi, quyết định dùng image này!

## Cài đặt Circle CI

### 1. Đăng nhập vào Circle CI

Đầu tiên cần phải đăng ký/đăng nhập vào Circle CI.

Mở link https://circleci.com/vcs-authorize/ và chọn **Login with github**, đăng nhập xong màn hình giao diện web của Circle CI sẽ được mở ra.
Trên giao diện này ta có thể browse các project trên tài khoản github của mình và team, setup Circle CI cho các em nó, theo dõi các job đã chạy vân vân và vân vân. Mình đánh giá giao diện trực quan và dễ sử dụng, đầy đủ thông tin.

### 2. Setup Circle CI cho project

Chọn menu "ADD PROJECTS" ở bên tay trái, trong list các repositories được hiển thị ra mình chọn blog vì lần này mình muốn cài đặt Circle CI cho nó.
![](/blog/images/ci_1.png)

Ở màn hình tiếp theo, trong mục language nếu như không có ngôn ngữ project bạn sử dụng thì cũng đừng lo lắng. Việc chọn lấy một ngôn ngữ chỉ là để web nó gợi ý ra nội dung sample cho file `config.yml` thôi. Mình luôn chọn ngôn ngữ `Other`.

Nhìn tiếp xuống bên dưới có thể thấy hướng dẫn các bước tiếp theo cần làm để hoàn tất cài đặt Circle CI. Tóm tắt lại là nó bảo ta hãy tạo file `.Circleci/config.yml` ở trong thư mục gốc của repository, tham khảo nội dung sample dưới đây và chỉnh sửa file config.yml rồi cuối cùng ấn nút `Start building`.

Hướng dẫn là thế nhưng mà dại gì làm y như lời nó, mình vừa làm vừa mò mà. Thế nên các bạn cứ tự tin ấn luôn nút `Start building` để hoàn tất việc setup Circle CI cho project nhé. Mình luôn làm thế, dĩ nhiên lần đầu build Circle CI sẽ báo ngay ra lỗi sau, nhưng không sao cả. 

> No configuration was found in your project. Please refer to https://Circleci.com/docs/2.0/ to get started with your configuration.

![](/blog/images/ci_2.png)

Từ bây giờ mỗi khi push code lên github, CI job sẽ tự động được khởi động lên rồi.
Vậy chờ gì nữa, qua bước tiếp theo cấu hình cụ thể các job và step cho Circle CI thôi.

### 3. Cấu hình Circle CI

Trước hết tạo file config.yml như Circle CI đã dạy.
```bash
mkdir .circleci
touch .circleci/config.yml
```

Nội dung ban đầu file config.yml mình sẽ để như sample của Circle CI:
```yaml
version: 2
jobs:
  build:
    docker:
      - image: debian:stretch
    steps:
      - checkout
      - run:
          name: Greeting
          command: echo Hello, world.
```

Tiếp theo mình sẽ sửa lại chỗ docker image, mình thay thế vào cái image mình tìm thấy ở trên.
```yaml
    docker:
      - image: andthensome/alpine-hugo-git-bash
```

Mình chỉ muốn chạy job khi có push lên branch master thôi nên mình sẽ thêm cấu hình này vào job:
```yaml
jobs:
  build:
    branches:
      only:
        - master
```

Blog của mình được publish thông qua github pages, chỉ cần push nội dung lên branch cố định mà github đã quy định là `gh-pages` thì tự khắc content sẽ được serve ở domain https://chienkira.github.io/blog.  
Hiện tại, việc build blog (sinh ra nội dung html tĩnh của blog) và push nó lên branch `gh-pages` đã được script hóa trong file [publish.sh](https://github.com/chienkira/blog/blob/master/publish.sh) rồi. Do đó trong Circle CI mình hi vọng chỉ cần chạy file script này là xong luôn. Thêm step vào `config.yml` như sau:

```yaml
...
    docker:
      - image: andthensome/alpine-hugo-git-bash
    steps:
      - run:
          name: build site and push to github-pages
          command: sh publish.sh
```

Rồi thử commit và xem Circle CI chạy ra sao nhé.
`git commit -am "add circle ci" && git push`

Check trên màn hình Circle CI có thể thấy job được khởi động như nó được thiết kế nhưng nó đã failed.
![](/blog/images/ci_3.png)
Click vào xem chi tiết kiểm tra có thể biết chuyện gì đã xảy ra trong docker container.
Ở đây mình gặp lỗi như bên dưới - blog của mình dùng theme *chảnh*, nó đòi phải có hugo version cao hơn. :smiley:

> ERROR 2019/03/23 14:44:12 Current theme does not support Hugo version 0.31.1. Minimum version required is 0.45

Google một hồi mình thêm được step sau để nâng cấp hugo lên version 0.50 vào cái image alpine này.
```yaml
      - run:
          name: upgrade hugo
          command: |
            apk update && apk add ca-certificates && update-ca-certificates && apk add openssl && apk add openssh-client
            wget https://github.com/gohugoio/hugo/releases/download/v0.50/hugo_0.50_linux-64bit.tar.gz
            tar xzf hugo_0.50_linux-64bit.tar.gz -C /usr/local/bin/ 
```
Và thêm 1 step để config git nữa là mình hoàn chỉnh xong cấu hình Circle CI.
*Các bạn cần có thể tham khảo file config.yml cuối cùng ở đây: https://github.com/chienkira/blog.*

Commit lần nữa rồi kiểm tra web Circle CI, job đã chạy thành công xanh lè. :smiley:
![](/blog/images/ci_4.png)

Kể từ bài này là blog của mình đã được publish tự động rồi. Mặc dù lần này, blog đơn giản nên công việc CI không có gì nhiều và phức tạp nhưng mục đích tự động hóa thì đã thành công.
Circle CI là công cụ miễn phí và tuyệt vời, nếu bạn có project nào có thể áp dụng hãy trải nghiệm thử một lần nhé.