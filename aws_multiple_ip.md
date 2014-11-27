#aws多ip问题

公司的服务器切换到亚马逊aws服务器, 我们的服务需两个公网ip, 在aws上申请到了两个公网ip, 并且配置成功。

aws服务器使用的是内网ip通过NAT转换成公网ip的方式.

映射关系如下:

`$public_ip1 ---> $private_ip1 ----> eth0`

`$public_ip2 ---> $private_ip2 ----> eth1`

但是使用过程发现只有一个公网可以通。

当黙认网关设置在`eth0`上时，`$public_ip1`可以通, `$public_ip2`不通；当黙认网关设置在`eth1`上时`$public_ip2`可以通，`$public_ip1`不通。

于是我做了如下操作:

把黙认网关设置在eth0上， 在公网ip为`123.123.123.123`的机器上`ping $public_ip2`，然后在服务器上使用`ip route show cache`查看路由表缓存，发现服务器到`123.123.123.123`的路由走的是`eth0`, 也就是说`$public_ip2`接收到的数据包，返回的时候走的是`eth0`，源ip会被NAT成`$public_ip1`， 所以会`$public_ip2`不通。

问题的原因已经明白了，现在要做的就是怎样让从`eth1`进入的数据包从`eth1`返回，`eth0`接收到的数据包从`eth0`返回。

这里我使用Linux的iproute2功能，让`eth0`和`eth1`查找不同的路由表

先执行如下命令，添加两条路由表别名

	echo 200 t_eth0 >> /etc/iproute2/rt_tables
	echo 201 t_eth1 >> /etc/iproute2/rt_tables

然后在路由表中添加路由规则

	ip route add default via $gw_eth0 dev eth0 table t_eth0
	ip route add default via $gw_eth1 dev eth1 table t_eth1

再添加路由查找规则

	ip rule add from $private_ip1 table t_eth0
	ip rule add from $private_ip2 table t_eth1

以上规则的意思是，当源ip为`$private_ip1`时，查路由表`t_eth0`，在表`t_eth0`中，黙认走`eth0`;
当源ip为`$private_ip2`时，查路由表`t_eth1`，表`t_eth1`中，黙认走`eth1`。

这样设置之后，不管黙认网关设置在哪个网卡上，两个公网ip都可以通了。
