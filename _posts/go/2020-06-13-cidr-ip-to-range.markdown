---
layout: post
title: "CIDR ip to range"
date: 2020-06-13 20:05:00 +0800
categories: go
---
将CIDR类型ip转换为range

#### ipv6 转换
``` go
package main

import (
	"fmt"
	"net"
)

// ipv6Cidr2Range 将带掩码的ipv6转换成[[]byte startIp, []byte endIp]形式
func ipv6Cidr2Range(ipv6Cidr string) (net.IP, net.IP, error) {
	_, ipnet, err := net.ParseCIDR(ipv6Cidr)
	if err != nil {
		return nil, nil, err
	}

	start := ipnet.IP.Mask(ipnet.Mask).To16()
	end := make(net.IP, len(start))
	copy(end, start)

	// maskSize 网络位长度
	maskSize, _ := ipnet.Mask.Size()

	// maskSize向上取整后第一个byte开始全部取0xff
	for index := (maskSize + 7) >> 3; index < 16; index++  {
		end[index] = 0xff
	}

	// 对于maskSize长度%8不为0的，即末尾不满足1byte部分需要额外处理——末尾8-(maskSize%8)bit全取1，前maskSize%8位取start中的值
	if maskSize % 8 != 0 {
		 end[((maskSize + 7) >> 3) - 1] |= (1 << uint(8 - maskSize % 8)) - 1
	}

	return start, end, nil
}

func main() {
	tests := [][3]string{
		{"2001:db8:abcd:0012::0/0", "::", "ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"},
		{"2001:db8:abcd:0012::0/64", "2001:db8:abcd:12::", "2001:db8:abcd:12:ffff:ffff:ffff:ffff"},
		{"f000::000f/2", "c000::", "ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"},
		{"c000::000f/1", "8000::", "ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"},
		{"c000::000f/3", "c000::", "dfff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"},
		{"c000::000f/4", "c000::", "cfff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"},
		{"d000::000f/16", "d000::", "d000:ffff:ffff:ffff:ffff:ffff:ffff:ffff"},
		{"d000:8000::000f/17", "d000:8000::", "d000:ffff:ffff:ffff:ffff:ffff:ffff:ffff"},
		{"d000:7000::000f/17", "d000::", "d000:7fff:ffff:ffff:ffff:ffff:ffff:ffff"},
		{"d000:7000::000f/18", "d000:4000::", "d000:7fff:ffff:ffff:ffff:ffff:ffff:ffff"},
		{"d000:7000::000f/19", "d000:6000::", "d000:7fff:ffff:ffff:ffff:ffff:ffff:ffff"},
		{"d800:7000::000f/7", "d800::", "d9ff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"},
		// 注意：xxx:000f:xxx 这种会被转换成 xxx:f:xxx
		{"d800:7000::000f/128", "d800:7000::f", "d800:7000::f"},
		{"d800:000f::000f/32", "d800:f::", "d800:f:ffff:ffff:ffff:ffff:ffff:ffff"},
	}

	for _, test := range tests {
		start, end, err := ipv6Cidr2Range(test[0])
		if err != nil {
			fmt.Printf("error: %+v\n", err)
		}
		if start == nil || start.String() != test[1] {
			fmt.Printf("error, cur start ipv6 is %s, expect start ipv6 is %s\n", start, test[1])
		}
		if end == nil || end.String() != test[2] {
			fmt.Printf("error, cur start ipv6 is %s, expect start ipv6 is %s\n", end, test[2])
		}
	}
}
```

#### ipv4 转换

``` go
// Ip2long 将ip转换成unit32格式
func Ip2long(ipStr string) uint32 {
	ip := net.ParseIP(ipStr)
	if ip == nil {
		return 0
	}
	ip = ip.To4()
	return binary.BigEndian.Uint32(ip)
}

// Ip2range 将带掩码的ip转换成[uint32 startIp, uint32 endIp]形式
func Ip2range(ipWithMast, ReqId string) ([]uint32, error) {
	vipSegs := strings.Split(ipWithMast, "/")
	ip := vipSegs[0]

	var startIp uint32
	var endIp uint32

	if len(vipSegs) > 1 {
		mask, err := strconv.Atoi(vipSegs[1])
		if err != nil {
			err := log.Logger.Error("[ReqId: %s], mask type conversion failed", ReqId)
			return nil, err
		}
		startIp = Ip2long(ip) & (((1 << uint32(mask)) - 1) << uint32(32-mask))
		endIp = Ip2long(ip) | ((1 << uint32(32-mask)) - 1)

	} else {
		startIp = Ip2long(ip)
		endIp = Ip2long(ip)
	}
	rangIp := []uint32{
		startIp,
		endIp,
	}
	return rangIp, nil
}
```

