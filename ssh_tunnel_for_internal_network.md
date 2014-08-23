#使用ssh遂道远程访问家中的设备
##背景

我家中有一台树莓派(raspberrypi)，我用它作下载机，还有一台WiFiDisk用来做NAS。白天上班时我想知道家中的设备运行情况，可是我家是长城宽带，是一个大局域网，ADSL拔号拿到的ip看似是公网ip，实际上还是内网ip，所以DDNS和端口映射就无效了。

要解决这种远程访问局域网的问题，我知道至少有两种方案，一种是使用ngrok，另一种是使用ssh遂道。ngrok使用的是别人提供的服务器，我有自己的vps，所以我选择了使用ssh遂道中转。

##知识复习

我们来复习一下ssh遂道的三种方式:

1. Local->Remote (ssh -L) 本地监听一个端口，此端口的所有请求通过ssh遂道转发到远端的某个端口;
2. Remote->Local (ssh -R) 远端临听一个端口，此端口的所有请求通过ssh遂道转发到本地网络的某个端口;
3. Dynamic  (ssh -D) 本地监听一个端口，此端口做为sock5代理，所有连接通过远端服务转发出去，ssh翻墙通常使用这种方式。

我使用的第二种方式，在树莓派上运行ssh连接到vps，在vps上申请一个端口，然后转发到本地的22端口。要使用这种方式必须要先配置ssh无密码登陆，具体的配置自行[搜索](https://www.google.com)， 如果不配置无密码登陆就无法实现断线后自动重连。

##命令
我使用的是如下命令

	ssh -R $VPS_IP:2222:127.0.0.1:22 -l $USERNAME -N -T -o ServerAliveInternal=20 $VPS_IP

以上命令，`-R $VPS_IP:2222:127.0.0.1:22`的意思是在vps上临听2222端口，转发到本机的22端口，`-N`表示不启动shell，`-T`表示不分配tty，`-o ServerAliveInternal=20`表示每20s发送一个消息，用来检测是否断线。我第一次就是忘了`-o`参数，结果在家中测试可以连接，等我到办公室时，已经无法连接了，因为底层已经断了，便是没有消息发送，上层无法检测到是否断线。

##ssh配置
vps上需要修改ssh配置文件`/etc/ssh/sshd_config`，修改其中的`GatewayPorts`为`yes`或`clientspecial`， 此参数黙认为`no`，当为`no`的时候分配的端口只能bind到127.0.0.1上，
是无法通过远程连接的。修改后需要重启sshd，执行命令:
	
	service sshd restart

##完整脚本

	VPS_IP=1.1.1.1
	USERNAME=myname
	REMOTE_PORT=2222
	LOCAL_PORT=2222
	LOCAL_HOST=127.0.0.1
	ID_FILE=~/my_id   # private key file
	
	while true
	do
		ssh -i $ID_FILE \
			-N -T \
			-R $VPS_IP:$REMOTE_PORT:$LOCAL_HOST:$LOCAL_PORT \
			-l $USERNAME \
			-o ServerAliveInternal=20 \
			-o ConnectTimeout=20 \
			$VPS_IP
		sleep 10
	done

将此脚本加到启动脚本中即可。

此脚本只是转发到本机的ssh端口，当然还可以转发到其它机器。例如，家中的笔忘本电脑打开了远程桌面，假设ip是192.168.1.100，使用`-R $VPS_IP:3322:192.168.1.100:3389`参数, 就可以通过vps的3322端口访问到家中笔记本电脑的远程桌面了。

当然，在家中访问办公室电脑也可以使用同样的方法，在办公室运行ssh即可。此方法不需要向网络管理员申请端口映射。自己可以随机控制端口，你只需要有自己的vps即可。
