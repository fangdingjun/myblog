#让你的web server支持http/2.0

`http/2.0` 是下一代的[http](http://zh.wikipedia.org/wiki/%E8%B6%85%E6%96%87%E6%9C%AC%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE)协议（[查看wikipedia](http://zh.wikipedia.org/wiki/HTTP/2 "wikipedia")），它
比`http/1.1`具有更高的效率，目前还没有正式发布，还处在修改阶段，新最的标准是`draft-17`即[h2-17](http://tools.ietf.org/html/draft-ietf-httpbis-http2-17).

目前`chrome`，`firefox`都已支持`http/2.0`了，我们可以在自己的网站上部署`http/2.0`来尝新。

我们可以使用这个C语言的`http/2.0`的库[nghttp2](https://github.com/tatsuhiro-t/nghttp2), 安装方法:

    git clone https://github.com/tatsuhiro-t/nghttp2
    cd nghttp2
    autoreconf -ivf
    ./configure
    make
    sudo make install

安装成功后，有如下组件:

`libnghttp2-xx.so`  解析`http/2.0`的协议的库文件 

`nghttp`  一个`http/2.0`的客户端程序，类似于`wget`

`nghttpx`  一个`http/2.0`的代理程序, 类似于`squid`

`nghttpd` 一个`http/2.0`的web server程序， 类似于`nginx`

如果你的web server只有静态文件， 使用nghttpd就够了 

	nghttpd -d /path/to/docroot 443 server.key server.crt

如果不用ssl

	nghttpd -d /path/to/docroot 80 --no-tls

如果要支持php等动态内容，你可以使用`nghttpx`作为前端，后端使用`nginx`或者`apache`, 像这样

	nghttpx -f '0.0.0.0,443' -b 127.0.0.1,81 --cert server.crt --key server.key

`nghttpx`监听443端口，接收`http/2.0`请求，然后转发到`127.0.0.1:81`端口，让apache运行在此81端口处理实际的请求。

`nghttpx`黙认是启用`ssl`的，如果你的网站没有部署`ssl`，也可以使用不带`ssl`的`http/2.0`，命令如下

	nghttpx -f '0.0.0.0,80' -b 127.0.0.1,81 --frontend-notls

当然了，`nghttpx`/`nghttpd`也支持`http/1.1`，`http/1.0`协议，完全不用担心不支持`http/2.0`的浏览器。

如果你安装了[spdylay](https://github.com/tatsuhiro-t/spdylay)库, 在启用`ssl`时，`nghttpd`/`nghttpx`也支持`spdy/3.1`协议。

更多关于nghttp2的信息[点击这里](http://tatsuhiro-t.github.io/nghttp2).
