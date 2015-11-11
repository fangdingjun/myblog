通过openvpn使用搬瓦工ipv6
=======================

买了个搬瓦工的便宜vps，发现可以分配3个ipv6地址，于是折腾`ipv6 in ipv4 tunnel`，终于折腾成功了。记录下过程。

搬瓦工的ipv6是/64的地址段，但是只让你用3个地址，这3个地址我们可以自定义。

假如分配到的ipv6前缀是  `2001:db8:1:123`, 我们自定义成如下地址

	2001:db8:1:123::1
	2001:db8:1:123::a0:1
	2001:db8:1:123::a0:1000

第一个地址用在`venet0`上，第二个用在`tun0`上，第三个给客户端用。
之所用`a0:1`, `a0:1000` 是因为给openvpn 一个  /64 的地址，openvpn自动给自己用`:1`，给第一个客户端用`:1000`,这样就不用手动配ip地址了。

分配完成后重户vps，进入系统会发现三个ipv6地址全在`venet0`，我们删除两个, 用以下命令

	ip addr del 2001:db8:1:123::a0:1 dev venet0
	ip addr del 2001:db8:1:123::a0:1000 dev venet0

最好是把以上命令放入/etc/rc.local中，系统启动时自动执行。

修改openvpn配置文件，增加ipv6配置

	server-ipv6 2001:db8:1:123::a0:0/64
	push "route-ipv6 ::/0"


打开ipv6转发设置

	net.ipv6.conf.all.forwarding=1
	net.ipv6.conf.default.forwarding=1

把以上写入/etc/sysctl.conf，用sysctl -p生效，或者用如下命令临时生效，重启后失效

	sysctl -w net.ipv6.conf.all.forwarding=1
	sysctl -w net.ipv6.conf.default.forwarding=1

防火墙允许转发

	ip6tables -P FORWARD ACCEPT

或者根据条件允许转发

	ip6tables -P FORWARD DROP
	ip6tables -A FORWARD -p tcp --dport 80 -j ACCEPT
    ip6tabels -A FORWARD -p tcp --dport 443 -j ACCEPT

重启openvpn服务端，客户端重新连接，	在客户端机器上ping 地址`2001:db8:1:123::1`,	`2001:db8:1:123::a0:1`，
`ipv6.google.com`。

如果都能通，配置就完成了。

配置文件
=======

以下是我的server端配置文件

	server 10.0.8.0 255.255.255.0
	topology subnet
	server-ipv6 2001:db8:1:123::a0:0/64
	port 11948
	proto tcp
	dev tun
	push "redirect-gateway def1 local"
	push "topology subnet"
	push "dhcp-option DNS 8.8.8.8"
	push "dhcp-option DNS 4.2.2.2"
	push "ping 5"
	push "ping-restart 30"
	push "route-ipv6 ::/0"
	push "route 23.xxx.xxx.xxx 255.255.255.255 net_gateway"
	keepalive  5 30
	duplicate-cn
	persist-key
	persist-tun
	group nogroup
	user nobody
	tls-auth ta.key
	ca ca.crt
	cert server.crt
	key server.key
	dh dh.key

以下是客户端配置文件

	client 
	proto tcp
	dev tun
	verb 3
	remote 127.0.0.1 11948 tcp
	http-proxy 127.0.0.1 8080
	http-proxy-retry 10
	http-proxy-timeout 10	
	keepalive 10 60
	route 192.168.1.0 255.255.255.0 net_gateway
	script-security 2
	#route-up d:/add_route.bat
	#route-pre-down d:/del_route.bat
	ca ca.crt
	cert client.crt
	key client.key
	tls-auth ta.key


###参考链接

- [http://unix.stackexchange.com/questions/136211/routing-public-ipv6-traffic-through-openvpn-tunnel](http://unix.stackexchange.com/questions/136211/routing-public-ipv6-traffic-through-openvpn-tunnel)
- [http://serverfault.com/questions/237851/how-can-i-setup-openvpn-with-ipv4-and-ipv6-using-a-tap-device](http://serverfault.com/questions/237851/how-can-i-setup-openvpn-with-ipv4-and-ipv6-using-a-tap-device)