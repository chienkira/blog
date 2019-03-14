---
title: "Tự Tạo Chương Trình CLI Của Chính Mình Không Đụng Hàng"
date: 2019-03-13T23:04:57+09:00
draft: no
tags: [cli, python]
language: vi
toc: true
authors: [chienkira]
cover: /blog/images/kira-cli-weather-demo.png
---

**Lập trình viên không sớm thì muộn cũng sẽ yêu cái máy tính không khác gì yêu vợ. Rồi ngày qua ngày tiếp xúc với cửa sổ dòng lệnh, mắt lập trình viên dần thấy quen và ưng cái giao diện command line (CLI) hơn cả GUI màu sắc :joy:**

*CLI: command line interface*

*Đây là live action cái CLI mình đã làm thử ra.*
![](/blog/images/kira-cli-weather-demo.gif)

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

* **argument**: là parameter bắt buộc, nếu không có app CLI sẽ trả về lỗi. Ví dụ như `pip install` sẽ lỗi, còn nếu chỉ định thêm argument `requests` thì lệnh `pip install requests` sẽ chạy được, vậy đó.
* **option**: là parameter không bắt buộc - optional. Cách chỉ định nó là sử dụng một cặp key - value, ví dụ như `git commit --message "init"` thì phần `--message "init"` chính là option.

Với những app CLI phức tạp hơn ví dụ như aws-cli, ta còn thấy tất cả các command được nhóm vào chung một entry point là aws. Ví dụ thử phân tách các thành phần trong lệnh `aws s3 ls`, `aws` được gọi là group hay entry point, `s3` là command còn `ls` sẽ là argument.

# Xây dựng CLI app của riêng mình

## Định hình
App CLI lần này làm mình muốn nó phải là ở mức sử dụng được thực sự, không phải là demo hay là kiểu làm cho có.

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
Không tự dự báo được nên chắc chắn phải "nương tựa" vào API của một provider nào đó rồi. Mình tìm thấy https://darksky.net/, API dễ dùng đầy đủ thông tin, nội dung dự báo cũng khớp với app Yahoo! Weather.

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
    |--Pipfile.lock
    |--setup.cfg
    |--setup.py
    |--weather
    |  |--cli.py
    |  |--functions.py
    ```
    Mình dùng pipenv để quản lý dependencies nên mình có các file Pipfile.

    File setup.cfg và setup.py là những file cần thiết cho setuptools, giúp sau đó mình có thể cài đặt app CLI của mình lên máy thành một cli thực thụ. Cụ thể về setuptools mình sẽ làm việc sau với nó.

    Thư mục `weather/` là để chứa source liên quan đến chức năng weather. Sau này dev thêm các chức năng khác vào app CLI này thì sẽ tạo các thư mục mới tương tự như `weather/`. Trong `weather` mình dùng 2 file cli.py và functions.py. Ý tưởng là file cli.py sẽ quyết định interface cli: tên cli là gì, cần những param gì vân vân, còn logic thực sự của các command thì sẽ viết trong functions.py.

2. Bước đầu cài đặt cho setuptools

    Mình không định sẽ sử dụng CLI của mình theo kiểu củ chuối là `python kira-cli.py weather` nên cái tên setuptools xuất hiện ở đây. Sau khi dùng setuptools mình có thể cài đặt cli lên máy rồi gọi nó lên từ bất kỳ đâu với câu lệnh đơn giản `weather`.

    Về nguyên tắc setuptools yêu cầu một file setup.py, trong đó định nghĩa các thông tin ví dụ như tên, version, tác giả, danh sách các dependency... của package. Mình thì sử dụng thêm file setup.cfg (setuptools hỗ trợ), thay vì phải viết code python dài dòng trong setup.py mình chỉ cần define nội dung cần thiết vào file setup.cfg.

    setup.py sẽ trở thành đơn giản như thế này:
    ```python
    from setuptools import setup

    setup()
    ```

    setup.cfg thì đại khái như sau:
    ```python
    [metadata]
    name = kira-cli
    version = 1.0.0
    author = chienkira

    [options]
    packages = find:
    install_requires =
        click
        requests

    [options.entry_points]
    console_scripts =
        weather = weather.cli:start
    ```
    Chỗ console_scripts có nghĩa là khi lệnh `weather` được nhập vào, hàm `start` trong file `weather/cli.py` sẽ được chạy.

    Sau khi chuẩn bị xong cho setuptools, để cài đặt package dùng lệnh `python setup.py develop`. Ở đây ta dùng mode develop vì ta còn dev logic của app nữa nên mode develop sẽ giúp ta test được source đang edit 1 cách tức thì. Khi dev xong thì để cài đặt ổn định dùng lệnh `python setup.py install`.

    Đến đây là từ cửa sổ terminal/console lệnh `weather` có thể chạy được rồi!

3. Tạo CLI app thực thụ với Click

    Hình dung khi sử dụng app CLI ta sẽ gõ lệnh kiểu như sau:
    ```bash
    weather tokyo --interval daily --language ja
    ```
    Vậy thì việc bắt và đọc các argument, các options sẽ rất vất vả nếu làm thủ công. Ngoài ra chúng ta thường thấy các app CLI hỗ trợ lệnh `--help`, ví dụ như `git --help` và một tràng dài mô tả usage được in ra trên màn hình đúng không?

    Cả 2 vấn đề trên có thể giải quyết rất dễ dàng với lib [Click](https://github.com/pallets/click).
    Trong cli.py ta sử dụng các decorator Click cung cấp để mô tả app CLI như sau:
    ```python
    @click.command()
    @click.argument("city")
    @click.option("--language", help="Language Ex: ja, en.", default="en")
    @click.option("--interval", help="Interval (daily or hourly). Default is daily.", default="daily")
    def main(city: str, language: str, interval: str):
        """Kira-cli | display weather information that retrieve from darksky API"""
        data = get_weather_info(city, language, interval)
        pretty_print(data)
    ```
    Chỉ như vậy và app CLI của mình đã có thể hoạt động như một cli thực thụ.

    Thử nhé!
    ```bash
    $ weather

    Usage: weather [OPTIONS] CITY
    Try "weather --help" for help.

    Error: Missing argument "CITY".
    ```
    ```bash
    $ weather --help
    
    Usage: weather [OPTIONS] CITY

    Kira-cli | display weather information that retrieve from darksky API

    Options:
    --language TEXT  Language Ex: ja, en.
    --interval TEXT  Interval (daily or hourly). Default is daily.
    --help           Show this message and exit.
    ```
    Ồ cũng xịn không kém gì git hay aws-cli rồi! :smiley:

4. Hoàn thành logic lấy thông tin thời tiết

    Logic của nó không có gì đặc biệt.
    
    Mình tóm tắt lại là ban đầu dùng 1 API để lấy ra tọa độ địa lý của argument đầu vào city, sau đó gọi API của darksky với tọa độ đó để lấy về thông tin thời tiết.

    Cuối cùng dùng colorama và print khéo léo ra màn hình cho dễ nhìn thôi.

    Các bạn có thể xem chi tiết file functions.py ở github repository này. [github.com/chienkira/kira-cli/.../functions.py](https://github.com/chienkira/kira-cli/blob/master/weather/functions.py)

    Các bạn cũng có thể tự tải về cài vào máy chạy của mình dùng thử nhé. Chỉ cần máy có cài đặt python 3.
    ※ Chú ý là máy mac thì đang mặc định python 2 thôi nên sẽ lỗi, khuyên các bạn cài pyenv rồi đổi qua python 3.6.

    ```bash
    git clone https://github.com/chienkira/kira-cli.git && cd kira-cli
    python setup.py install
    # hoặc nếu ở máy bạn python 3 vẫn được cài ở `python3` thì chạy lệnh: `python3 setup.py install` 
    weather tokyo
    ```
    