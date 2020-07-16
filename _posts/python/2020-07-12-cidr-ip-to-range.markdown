---
layout: post
title: "CIDR ip to range"
date: 2020-07-12 20:05:00 +0800
categories: python
---
将CIDR类型ip转换为range

#### ipv6 转换

##### 使用 netaddr(python 2.7 版本以上才支持) 库实现 ipv6 cidr 转 range
``` python
# -*- coding: UTF-8 -*-
from netaddr import IPAddress
from netaddr import IPNetwork


def convert_ipv6_cidr_to_range(addr):
    """
    get integer value start_ip and end_ip of net address
    [d800:000f::000f/32] --- return 255211775190703847597530955573826158592, 340282366920938463463374607431768211455
    :param addr 
    :return start_ip and end_ip
    """
    ip_range = IPNetwork(addr)
    return ip_range.first, ip_range.last


def test_convert_ipv6_cidr_to_range():
    """
    test function convert_ipv6_cidr_to_range
    """
    test_cases = [
        ["2001:db8:abcd:0012::0/0", "::", "ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["2001:db8:abcd:0012::0/64", "2001:db8:abcd:12::", "2001:db8:abcd:12:ffff:ffff:ffff:ffff"],
        ["f000::000f/2", "c000::", "ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["c000::000f/1", "8000::", "ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["c000::000f/3", "c000::", "dfff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["c000::000f/4", "c000::", "cfff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d000::000f/16", "d000::", "d000:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d000:8000::000f/17", "d000:8000::", "d000:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d000:7000::000f/17", "d000::", "d000:7fff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d000:7000::000f/18", "d000:4000::", "d000:7fff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d000:7000::000f/19", "d000:6000::", "d000:7fff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d800:7000::000f/7", "d800::", "d9ff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d800:7000::000f/128", "d800:7000::f", "d800:7000::f"],
        ["d800:000f::000f/32", "d800:f::", "d800:f:ffff:ffff:ffff:ffff:ffff:ffff"],
    ]
    for test_case in test_cases:
        net_addr = test_case[0]
        start_ip_target = test_case[1]
        end_ip_target = test_case[2]

        ip_range = IPNetwork(net_addr)
        start_ip = str(IPAddress(ip_range.first, 6).ipv6())
        end_ip = str(IPAddress(ip_range.last, 6).ipv6())
        if (start_ip_target != start_ip) or (end_ip_target != end_ip):
            print "convert_ipv6_cdir_to_range result error. net_addr: %s, (start_ip: %s, end_ip: %s) " \
                  "(start_ip_target: %s, end_ip_target: %s)" \
                  % (net_addr, start_ip, end_ip, start_ip_target, end_ip_target)


if __name__ == '__main__':
    test_convert_ipv6_cidr_to_range()
```


##### python2.6版本下的实现

python2.6版本下不能使用netaddr库, github 上也没找到相关实现，自己撸了一个

``` python
# -*- coding: UTF-8 -*-
import ipaddress


def convert_ipv6_cidr_to_range(addr):
    """
    convert_ipv6_cidr_to_range: 将cidr形式的ipv6地址转换为区间形式的ipv6地址
    [d800:000f::000f/32] --- return true(function call succeeded) "D800:F::", "D800:F:FFFF:FFFF:FFFF:FFFF:FFFF:FFFF"
    :param addr
    :return error, start_ip, end_ip
    """
    try:
        ipv6_interface_obj = ipaddress.IPv6Interface(unicode(addr))
    except Exception as e:
        error = "illegal ipv6 cidr: %s, err: %s" % (addr, e)
        return error, '', ''

    ipv6_and_netmask_str = ipv6_interface_obj.exploded.encode()
    ipv6_and_netmask = ipv6_and_netmask_str.split("/")
    ipv6_list = list("".join(str(ipv6_and_netmask[0]).split(":")))
    mask_size = int(ipv6_and_netmask[1])

    for index in range(0, len(ipv6_list)):
        ipv6_list[index] = int(ipv6_list[index], 16)

    start_ip = ipv6_list[:]
    end_ip = ipv6_list[:]

    # start_ip 和 end_ip 用的 int 数组，每个元素代表 4bit 的ip
    # mask_size 向上取整后第一个元素开始开始 end_ip 全部取 0xf，start_ip 全部取 0x0
    for index in range((mask_size + 3) >> 2, 32):
        start_ip[index] = 0x0
        end_ip[index] = 0xf

    # 对于 mask_size 长度 %4 不为 0 的，即末尾不满足 4bit 部分需要额外处理——
    # 对于 start_ip，mask_size 外的全部取 0；对于 end_ip，mask_size 外的全部取 1
    if mask_size % 4 != 0:
        start_ip[((mask_size + 3) >> 2) - 1] = \
            (start_ip[((mask_size + 3) >> 2) - 1] >> (4 - mask_size % 4)) << (4 - mask_size % 4)
        end_ip[((mask_size + 3) >> 2) - 1] |= (1 << (4 - mask_size % 4)) - 1

    count = 1
    start_ip_int = ''
    end_ip_int = ''
    for index in range(0, len(start_ip)):
        start_ip_byte = hex(start_ip[index])[2]
        end_ip_byte = hex(end_ip[index])[2]

        start_ip_int = start_ip_int + start_ip_byte
        end_ip_int = end_ip_int + end_ip_byte

        if count % 4 == 0 and count != 32:
            start_ip_int = start_ip_int + ':'
            end_ip_int = end_ip_int + ':'
        count += 1

    try:
        start_ip_obj = ipaddress.IPv6Interface(unicode(start_ip_int))
        end_ip_obj = ipaddress.IPv6Interface(unicode(end_ip_int))
    except Exception as e:
        error = "illegal ipv6 range, start_ip_obj: %s, end_ip_obj: %s, err: %s" % (start_ip_int, end_ip_int, e)
        return error, '', ''

    return None, str(start_ip_obj.compressed).split("/")[0].upper(), str(end_ip_obj.compressed).split("/")[0].upper()


def test_convert_ipv6_cidr_to_range():
    """
    test_convert_ipv6_cidr_to_range: 测试函数 convert_ipv6_cidr_to_range
    """
    test_cases = [
        ["2001:db8:abcd:0012::0/0", "::", "ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["2001:db8:abcd:0012::0/64", "2001:db8:abcd:12::", "2001:db8:abcd:12:ffff:ffff:ffff:ffff"],
        ["f000::000f/2", "c000::", "ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["c000::000f/1", "8000::", "ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["c000::000f/3", "c000::", "dfff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["c000::000f/4", "c000::", "cfff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d000::000f/16", "d000::", "d000:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d000:8000::000f/17", "d000:8000::", "d000:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d000:7000::000f/17", "d000::", "d000:7fff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d000:7000::000f/18", "d000:4000::", "d000:7fff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d000:7000::000f/19", "d000:6000::", "d000:7fff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d800:7000::000f/7", "d800::", "d9ff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d800:7000::000f/128", "d800:7000::f", "d800:7000::f"],
        ["d800:7000::000f/127", "d800:7000::e", "d800:7000::f"],
        ["d800:7000::000f/125", "d800:7000::8", "d800:7000::f"],
        ["d800:000f::000f/32", "d800:f::", "d800:f:ffff:ffff:ffff:ffff:ffff:ffff"],
    ]
    for test_case in test_cases:
        net_addr = test_case[0]
        start_ip_target = test_case[1].upper()
        end_ip_target = test_case[2].upper()

        _, start_ip, end_ip = convert_ipv6_cidr_to_range(net_addr)
        if (start_ip_target != start_ip) or (end_ip_target != end_ip):
            print "convert_ipv6_cidr_to_range result error. net_addr: %s, (start_ip: %s, end_ip: %s) " \
                  "(start_ip_target: %s, end_ip_target: %s)" \
                  % (net_addr, start_ip, end_ip, start_ip_target, end_ip_target)


if __name__ == '__main__':
    test_convert_ipv6_cidr_to_range()
```

#### 上面那个实现要依赖ipaddress库，并且不是很优雅(主要是依赖ipaddress库但是线上环境不允许)。再撸一个
```
# -*- coding: UTF-8 -*-

import socket
from binascii import hexlify


def convert_ipv6_to_int(ipv6_addr):
    """
    convert_ipv6_to_int: 将:分字符串形式的ipv6地址转换为10进制形式的ipv6地址
    :param ipv6_addr ipv6 address, string
    :return error, integer ipv6 address
    """
    try:
        addr = socket.inet_pton(socket.AF_INET6, ipv6_addr)
    except Exception as e:
        return e, -1
    return None, int(hexlify(addr), 16)


def convert_int_to_ipv6(ipv6_int):
    """
    convert_int_to_ipv6: 将10进制格式的ipv6地址转换为explode小写字符串格式的ipv6地址
    :param ipv6_int 10进制数字格式的ipv6地址
    :return [error, ipv6_addr] 注意：ipv6_addr为explode小写字符串格式的ipv6地址
    """
    ipv6_addr = ''
    index = 0
    while ipv6_int > 0:
        index += 1
        ipv6_addr = hex(ipv6_int % 16)[2] + ipv6_addr
        ipv6_int = ipv6_int >> 4
        if (index % 4 == 0) and (index != 32):
            ipv6_addr = ':' + ipv6_addr

    while index < 32:
        index += 1
        ipv6_addr = '0' + ipv6_addr
        if (index % 4 == 0) and (index != 32):
            ipv6_addr = ':' + ipv6_addr

    return None, ipv6_addr


def convert_ipv6_to_compressed(ipv6_addr):
    """
    convert_ipv6_to_compressed: ipv6 0 位压缩
    :param ipv6_addr string ipv6地址
    :return error, addr——compress大写格式ipv6地址
    """
    error, ipv6_addr_int = convert_ipv6_to_int(ipv6_addr)
    if error is not None:
        return error, None

    error, ipv6_addr_explod = convert_int_to_ipv6(ipv6_addr_int)
    if error is not None:
        return error, None

    hextets = ipv6_addr_explod.split(":")

    # 记录hextets中最长全0子数组开始下标
    best_doublecolon_start = -1
    # 记录hextets中最长全0子数组长度
    best_doublecolon_len = 0
    # 记录当前全0子数组开始下标
    doublecolon_start = -1
    # 记录当前全0子数组当前长度
    doublecolon_len = 0
    for index, hextet in enumerate(hextets):
        zero_index = 0
        while zero_index < min(3, len(hextet) - 1) and hextet[zero_index] == '0':
            zero_index += 1

        if zero_index > 0:
            hextets[index] = hextet[zero_index:]

        if hextets[index] == '0':
            doublecolon_len += 1
            if doublecolon_start == -1:
                doublecolon_start = index

            if doublecolon_len > best_doublecolon_len:
                best_doublecolon_len = doublecolon_len
                best_doublecolon_start = doublecolon_start
        else:
            doublecolon_len = 0
            doublecolon_start = -1

    if best_doublecolon_len > 1:
        best_doublecolon_end = (best_doublecolon_start +
                                best_doublecolon_len)

        if best_doublecolon_end == len(hextets):
            hextets += ['']
        hextets[best_doublecolon_start:best_doublecolon_end] = ['']

        if best_doublecolon_start == 0:
            hextets = [''] + hextets

    return None, ':'.join(hextets).upper()


def convert_ipv6_cidr_to_range(addr):
    """
    convert_ipv6_mask_to_range: 将cidr形式的ipv6地址转换为区间形式的ipv6地址
    [d800:000f::000f/32] --- return true(function call succeeded) "D800:F::", "D800:F:FFFF:FFFF:FFFF:FFFF:FFFF:FFFF"
    :param addr
    :return error, start_ip, end_ip
    """
    ipv6_and_netmask_arr = addr.split("/")
    ipv6_addr = ipv6_and_netmask_arr[0]
    mask = int(ipv6_and_netmask_arr[1])

    error, ipv6_addr_int = convert_ipv6_to_int(ipv6_addr)
    if error is not None:
        return "illegal ipv6 cidr: %s, err: illegal ipv6_addr, %s" % (addr, error), '', ''

    if (mask < 0) or (mask > 128):
        return "illegal ipv6 cidr: %s, err: illegal mask" % addr, '', ''

    host_mask = 128 - mask
    start_ip_int = (ipv6_addr_int >> host_mask) << host_mask
    end_ip_int = ipv6_addr_int >> host_mask

    while host_mask > 0:
        end_ip_int = (end_ip_int << 1) + 1
        host_mask -= 1

    return None, convert_ipv6_to_compressed(convert_int_to_ipv6(start_ip_int)[1])[1], convert_ipv6_to_compressed(
        convert_int_to_ipv6(end_ip_int)[1])[1]


def test_convert_ipv6_cidr_to_range():
    """
    test_convert_ipv6_mask_to_range: 测试函数 convert_ipv6_mask_to_range
    """
    test_cases = [
        ["2001:db8:abcd:0012::0/0", "::", "ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["2001:db8:abcd:0012::0/64", "2001:db8:abcd:12::", "2001:db8:abcd:12:ffff:ffff:ffff:ffff"],
        ["f000::000f/2", "c000::", "ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["c000::000f/1", "8000::", "ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["c000::000f/3", "c000::", "dfff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["c000::000f/4", "c000::", "cfff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d000::000f/16", "d000::", "d000:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d000:8000::000f/17", "d000:8000::", "d000:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d000:7000::000f/17", "d000::", "d000:7fff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d000:7000::000f/18", "d000:4000::", "d000:7fff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d000:7000::000f/19", "d000:6000::", "d000:7fff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d800:7000::000f/7", "d800::", "d9ff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"],
        ["d800:7000::000f/128", "d800:7000::f", "d800:7000::f"],
        ["d800:7000::000f/127", "d800:7000::e", "d800:7000::f"],
        ["d800:7000::000f/125", "d800:7000::8", "d800:7000::f"],
        ["d800:000f::000f/32", "d800:f::", "d800:f:ffff:ffff:ffff:ffff:ffff:ffff"],
    ]
    for test_case in test_cases:
        net_addr = test_case[0]
        start_ip_target = test_case[1].upper()
        end_ip_target = test_case[2].upper()

        _, start_ip, end_ip = convert_ipv6_cidr_to_range(net_addr)
        if (start_ip_target != start_ip) or (end_ip_target != end_ip):
            print("convert_ipv6_mask_to_range result error. net_addr: %s, "
                  "(start_ip: %s, end_ip: %s) (start_ip_target: %s, end_ip_target: %s)" %
                  (net_addr, start_ip, end_ip, start_ip_target, end_ip_target))


if __name__ == '__main__':
    test_convert_ipv6_cidr_to_range()
```

