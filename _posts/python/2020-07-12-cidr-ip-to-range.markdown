---
layout: post
title: "CIDR ip to range"
date: 2020-06-13 20:05:00 +0800
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


##### python2.6版本下的实现——python2.6版本下不能使用netaddr库

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

