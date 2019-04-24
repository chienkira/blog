---
title: "Thủ Thuật Unpack Trong Python"
date: 2019-04-05T10:38:57+09:00
draft: no
tags: [tech, python, programing, TIL]
language: vi
toc: false
authors: [chienkira]
---

**Ký tự `*` ngoài là toán tử multiplication (phép nhân) và string replication ra, trong Python nó còn có một tác dụng khác khá xịn xò - unpack (một số người còn gọi là splat).**

## Ký hiệu `*`

Unpack chỉ có thể áp dụng lên một object loại iterable, để áp dụng việc cần làm là đặt ký hiệu `*` lên liền ngay trước object đó.

> Ví dụ:  
> Dùng trực tiếp với biến: `*list_variable`  
> Dùng với biểu thức: `*[i*2 for i in some_array]`

Phép unpack với ký hiệu `*` sẽ sinh ra *"các phần tử"*, được tách ra từ iterable object ban đầu.

Ví dụ như

- Nếu áp dụng lên một object kiểu list, ta sẽ nhận được các phần tử có trong list đó.
- Nếu áp dụng lên một object kiểu dictionary, ta sẽ nhận được các keys (chỉ key thôi, không có value) có trong dictionary đó.

Cùng xem qua đoạn code ví dụ sau để hình dung rõ hơn công dụng của unpack.

```python
# Method test nhận vào 3 params
def test(a, b, c):
    print("a =", a)
    print("b =", b)
    print("c =", c)

# object này có 3 phần tử
list = [1, 2, 3]

test(list)
# Sẽ bị lỗi TypeError: test() missing 2 required positional arguments: 'b' and 'c'

test(*list)
# a = 1
# b = 2
# c = 3
```
Unpack đã giúp tách độc lập các phần tử có trong object `list` ra, do đó method `test` nhận vào được đúng 3 params a, b và c tương ứng với 3 phần tử trong object `list`.

Đến đây thì nếu liên tưởng chút, ta sẽ thấy cách viết quen thuộc sau hiện lên ở trong đầu.
```python
def my_method(*args):
    #do something
```
Vâng chính là nó rồi đó ạ, `*args` quen thuộc ở đây bản chất là một trick sử dụng phép unpack. Unpack biến `args` tương đương với chuỗi các param truyền vào method. Vì thế mà trong method khi truy cập biến `args`, ta nhận được list chứa toàn bộ các params. Mình hay gọi cái này là phép nghịch đảo của unpack cho dễ hiểu.

Nhắc đến `*args` rồi thì lại phải nhắc đến họ hàng của nó là `**kwargs` :)) Vậy thì ta chuyển sang mục tiếp theo *Ký hiệu `**`* thôi!

## Ký hiệu `**`

Unpack khi sử dụng ký hiệu `**` có hiệu quả khác với khi sử dụng ký hiệu `*` ở chỗ, nó chuyên tách object kiểu dictionary thành các cặp key-value độc lập.

Nó giúp việc truyền một dict vào một method mà chỉ nhận keyword params dễ dàng hơn rất nhiều.

Đoạn code demo ví dụ sau sẽ dễ hiểu hơn mọi giải thích dài dòng :smiley:
```python
# Method test nhận vào 3 keyword params
def test(a=a, b=b, c=c):
    print("a =", a)
    print("b =", b)
    print("c =", c)

# dict này có 3 key, thứ tự có thể không cần giống thứ tự keyword trong method test
dict = {"a": 11, "c": 22, "b": 33}

test(dict)
# Sẽ bị lỗi TypeError: test() missing 2 required positional arguments: 'b' and 'c'

test(**dict)
# a = 11
# b = 22
# c = 33
```

## Unpack trong các ngôn ngữ khác

Javascript ES 6 cũng đã hỗ trợ phép unpack nhưng với tên gọi khác là **spread**, và sử dụng ký hiệu `...`.
```javascript
const numbers = [5, 6, 8, 9, 11, 999]
Math.max(...numbers)
```

Còn trong ruby thì gọi là phép splat, về mặt ký hiệu và cách dùng nó giống như python.
```ruby
people = ["Rudy", "Sarah", "Thomas"]
say *people
```

---

Sử dụng lâu rồi nhưng không hiểu bản chất bên dưới của nhiều đoạn code là phép unpack này, thật là xấu hổ quá. Bài này chủ yếu mang tính sắp xếp lại kiến thức của bản thân. Nếu bạn nào trót vào đọc và thấy chả có khỉ gì thì rất xin lỗi ạ!