##Linux 下SQLITE数据库无法在NFS/CIFS之上运行的解决办法
我在debain下mount了samba(CIFS)共享， 然后运行mindlna，让minidlna stream samba上的视频和音乐， 无奈minidlan拒绝工作。我启用debug模式，发现在minidlna启动时，sqlite数据库建表失败。于时我修改数据库的存放路径到本地的ext4文件系统上，minidlna工作良好。

由此推断可能是CIFS的问题， 于是我使用 `sqlite /mnt/media/test.db`测试，`/mnt/media`是samba的挂载点。我输入如下的SQL语句测试：

	create table t1 (a int, b int);

直接报如下错误：

	table is locked
我使用google搜索了一圈，发现很多人有此问题， 但都没有解决方案，最终在一个英文论坛上发现有人提到添加`nolock`参数， 但是提问的人没有回复效果。

我于是添加`nolock`参数、重新挂载samba来测试一下效果：

	umount /mnt/media
	mount -t cifs -o user=myuser,password=mypass,nolock //172.16.1.5/share /mnt/media
然后，再次使用`sqlite /mnt/media/test.db`测试，`create table`，`insert`, `delete`, `update` 等SQL操作都可以成功，不会有任何错误了。把minidlna的数据库路径修改到samba上也没有问题了。

所以`nolock`参数可以解决此问题，于是修改`/etc/fstab`添加`nolock`让系统启动时自动使用`nolock`参数挂载:

	//172.16.1.6/share	/mnt/media	cifs	defaults,user=myuser,password=mypass,nolock 0 0

