---
title: "Tự Tạo Chương Trình CLI Của Chính Mình Không Đụng Hàng"
date: 2019-03-06T11:04:57+09:00
draft: true
tags: [cli, python]
language: vi
toc: true
authors: [chienkira]
---

**Lập trình viên không sớm thì muộn cũng sẽ yêu cái máy tính không khác gì yêu vợ. Rồi ngày qua ngày tiếp xúc với cửa sổ dòng lệnh, mắt lập trình viên dần thấy quen và ưng cái giao diện command line (CLI) hơn cả GUI màu sắc :))**

*CLI: command line interface*

# Mở đầu
Từ vài năm trước, sau khi chuyển qua sử dụng Mac thì thói quen sử dụng command line của mình đã được cải thiện rõ rệt. Mình nhận thấy hiệu quả công việc thực sự được nâng cao nếu vừa biết sử dụng thuần thục command line lại vừa chăm chỉ cài đặt alias, viết script tự động hóa vân vân.

Đó là 1 quan điểm rất "nghiêm túc" :)) còn thực tế mà nói thì lợi ích của nó phải kể đến 2 cái này nữa.

* với cửa sổ dòng lệnh, bạn nghiễm nhiên được "oai" lên một chút trong mắt của mọi người xung quanh. 
* bật cửa sổ dòng lệnh lên chuyển tab qua qua lại lại, chỉ thế thôi là đã đánh lừa được sếp rằng mình đang làm việc chăm chỉ rồi.

Cái này là chia sẻ hoàn toàn thực tế, hiện tại chính mình cũng đang hàng ngày trên công ty mở vài tab terminal để làm việc riêng :smiley: Thế rồi suốt ngày làm việc với em CLI, mình nảy ra ý tưởng tự tạo cho riêng mình một cái. Lại đúng lúc cần rèn luyện skill python, mình quyết định bắt tay vào làm bằng python luôn.

Sản phẩm ra lò là em [github/kira-cli](https://github.com/chienkira/kira-cli) này. Sau đây mình muốn chia sẻ lại các kiến thức học được và quá trình để làm ra app CLI này.

# Tìm hiểu đặc điểm của CLI app

App CLI được gọi lên và thực thi bằng cách gõ vào tên của nó từ cửa sổ terminal/console. Ví dụ khi ta gõ lệnh `pip` thì chính là ta đang gọi lên app CLI có tên là `pip`.

Trừ những app siêu đơn giản, hầu hết với các app CLI khi sử dụng ta phải chỉ định thêm các **parameters** đằng sau tên của nó. Có 2 loại parameters là:

* **argument**: là parameter bắt buộc, nếu không có app CLI sẽ trả về lỗi. Ví dụ như `pip install` sẽ lỗi, còn nếu chỉ định thêm argument `requests` thì lệnh `pip install requests` sẽ chạy, vậy đó.
* **option**: là parameter không bắt buộc - optional. Cách chỉ định nó là sử dụng một cặp key - value, ví dụ như `git commit --message "init"` thì phần `--message "init"` chính là chỉ định option message.

Với những app CLI phức tạp hơn ví dụ như aws-cli, ta thấy tất cả các command được nhóm vào chung một entry point là aws. Ví dụ thử phân tách các thành phần trong lệnh `aws s3 ls`, `aws` được gọi là group hay entry point, `s3` là command còn `ls` sẽ là argument.

# Xây dựng CLI app của riêng mình

## Định hình
Mình muốn app CLI này phải là ở mức sử dụng được thực sự, không phải là demo hay là kiểu làm cho có.

* nó phải có thể gọi ra ở mọi nơi, giống như ở đâu cũng có thể gõ `git` hay là `aws` và quẩy ấy
* nó phải thực hiện một chức năng có ích và thực tế chứ không phải kiểu "đồ chơi" dỏm
* nó phải được thiết kế sao cho sau đó có thể dev thêm dần các chức năng hữu ích khác
* giống như các app CLI chất lượng nó phải hỗ trợ print ra màn hình có màu, dễ nhìn

## Quyết định
Ở Tokyo, thời gian này là lúc thời tiết thay đổi rất thất thường. Ngày ấm, ngày lạnh và ngày mưa xen kẽ nhau do đó mình thường xuyên phải xem thông tin dự báo thời tiết.

Vậy là ra rồi! Mình quyết định bước đầu sẽ làm một app CLI với chức năng đầu tiên là check thông tin thời tiết. Cụ thể hóa yêu cầu hơn nhé.

* cho phép chỉ định địa điểm muốn check thời tiết (tokyo hay saitama vân vân)
* hiển thị được cả thông tin nhiệt độ và vận tốc gió (ở Nhật, nhiệt độ không thấp quá mà gió to thì vẫn rét sun ch*m)
* giống các app thời tiết, nó phải hiển thị ra được dự báo của khoảng 1 tuần

## Khảo sát

1. Nguồn thông tin thời tiết  
Không tự dự báo được nên chắc chắn phải nương tựa vào API của một provider nào đó rồi. Mình tìm thấy https://darksky.net/, API dễ dùng đầy đủ thông tin, nội dung dự báo cũng khớp với app Yahoo! Weather.

2. Hiển thị màu sắc đẹp đẽ trên cửa sổ dòng lệnh  
Ở trang https://github.com/vinta/awesome-python mình tìm thấy vị anh hùng để hiện thực hóa việc này.  
Đó là https://pypi.org/project/colorama/. Xem qua document cũng vô cùng dễ hiểu. Ta có thể chỉ định màu nền và màu của chữ khi print ra cửa sổ dòng lệnh.

3. Cài đặt được để có thể gọi đến ở mọi nơi  
Tìm kiếm cái này là cái mình tốn thời gian nhất. Vì không có hình dung về nó nên cũng không có keywords chính xác để search. Phải qua một vài trang hỏi đáp thì mình mới tìm ra em nó.
Tool mình chọn sử dụng là [**setuptools**](https://setuptools.readthedocs.io/en/latest/setuptools.html).
    > Setuptools is a collection of enhancements to the Python distutils that allow developers to more easily build and distribute Python packages, especially ones that have dependencies on other packages.

## Tiến hành

1. Thiết kế cấu trúc source  
    ```bash
    |--Pipfile
    |--setup.cfg
    |--setup.py
    |--weather
    |  |--cli.py
    |  |--functions.py
    ```
    

2. Bước đầu cài đặt cho setuptools

3. Tạo CLI app thực thụ với Click

4. Hoàn thành logic lấy thông tin thời tiết

