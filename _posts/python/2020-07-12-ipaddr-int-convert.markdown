---
layout: post
title: "ipaddr int convert"
date: 2020-07-12 20:05:00 +0800s
categories: python
---

':'分字符串形式的ipv6地址和10进制形式的ipv6地址之间相互转换实现

``` python
# -*- coding: UTF-8 -*-
import socket
import ipaddress
from binascii import hexlify


def convert_ipv6_to_int(ipv6_addr):
    """
    convert_ipv6_to_int: 将:分字符串形式的ipv6地址转换为10进制形式的ipv6地址
    :param ipv6_addr ipv6 address, string
    :return integer ipv6 address
    """
    return int(hexlify(socket.inet_pton(socket.AF_INET6, ipv6_addr)), 16)


def convert_int_to_ipv6(ipv6_int):
    """
    convert_int_to_ipv6: 将10进制格式的ipv6地址转换为极简大写字符串格式的ipv6地址
    :param ipv6_int 10进制数字格式的ipv6地址
    :return [error, ipv6_addr] ipv6_addr 为极简大写字符串格式的ipv6地址
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

    try:
        ipv6_obj = ipaddress.IPv6Address(unicode(ipv6_addr))
    except Exception as e:
        error = 'illegal ipv6 addr, ipv6_int: %d, ipv6_addr: %s, error: %s' % (ipv6_int, ipv6_addr, e)
        return error, None
    return None, ipv6_obj.compressed.upper()


def test_convert_int_to_ipv6():
    """
    test_convert_int_to_ipv6: 测试函数convert_int_to_ipv6
    """
    test_cases = [
        '::',
        '::f',
        '::1',
        '::0',
        '0a::',
        'a0::',
        'fe80:0000:0000:0000:021b:77ff:fbd6:7860',
        'fe80::021b:77ff:fbd6:7860',
        'fe80::21b:77ff:fbd6:7860',
        'fe80::fbd6:7860'
    ]
    for case in test_cases:
        ipv6_int = convert_ipv6_to_int(case)
        error, ipv6_addr_result = convert_int_to_ipv6(ipv6_int)
        if not (error is None):
            print error
            continue
        ipv6_addr_target = ipaddress.IPv6Address(unicode(case)).compressed.upper()
        if ipv6_addr_result != ipv6_addr_target:
            print 'error, case: %s, ipv6_int: %d, ipv6_addr_result: %s, ipv6_addr_target: %s' % \
                  (case, ipv6_int, ipv6_addr_result, ipv6_addr_target)


if __name__ == '__main__':
    test_convert_int_to_ipv6()
```

