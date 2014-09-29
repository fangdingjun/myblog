##反向代理twitter.com的nginx配置文件

折腾nginx反向代理，终于配置成功了twitter.com的反向代理，可以登陆成功。

首先,需要安装 HttpSubsModule 模块，参见 [http://wiki.nginx.org/HttpSubsModule](http://wiki.nginx.org/HttpSubsModule)

在http部分加入如下指令，使`HttpSubsModule`替换javascript中域名，否则登陆时无法输码

	subs_filter_types  application/x-javascript application/javascript;


####反代`twitter.com`的配置

    server {
        server_name t.example.org;

        #proxy_cache cache1;

        listen 443 ssl spdy;

        add_header Strict-Transport-Security max-age=3153600;
        #add_header X-Frame-Options DENY;
        
        # 证书
        ssl_certificate      /home/user/a.example.org.crt;
        ssl_certificate_key  /home/user/a.example.org.key;

        #root t.example.org;

		# 禁止搜索引擎
        if ($http_user_agent ~ (google|bot|baidu) ){
            return 403;
        }

        location / {
            proxy_set_header Accept-Encoding "";
            proxy_set_header Accept-Langauge "zh-CN";
            proxy_set_header Host $proxy_host;

            proxy_pass https://twitter.com/;

            # cookie domain replace
            proxy_cookie_domain twitter.com t.example.org;

            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
			
			# 域名替换
            subs_filter twitter.com t.example.org;
            subs_filter abs.twimg.com abs.example.org;
			#subs_filter t.co co.example.org;
        }

    }

####反代`abs.twimg.com`的配置

    server {
        server_name abs.example.org;

        #proxy_cache cache1;

        listen 443 ssl spdy;

        add_header Strict-Transport-Security max-age=3153600;
        add_header X-Frame-Options DENY;

        # 证书
        ssl_certificate      /home/user/a.example.org.crt;
        ssl_certificate_key  /home/user/a.example.org.key;

        #root abs.example.org;
		
		# 禁止搜索引擎
        if ($http_user_agent ~ (google|bot|baidu) ){
            return 403;
        }

        location / {
            proxy_set_header Accept-Encoding "";
            proxy_set_header Accept-Langauge "zh-CN";
            proxy_set_header Host $proxy_host;

            proxy_pass https://abs.twimg.com/;

            # cookie domain replace
            proxy_cookie_domain twimg.com abs.example.org;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
			
			# 域名替换
            subs_filter twitter.com t.example.org;
            subs_filter abs.twimg.com abs.example.org;
			#subs_filter t.co co.example.org;
        }
    }


我是使用子域名反代twitter域名的方式， 理论上可以使用子目录反代域名的方式，但是这种配置会比较麻烦，很是考验耐心，我成功配置过用子目录反代google。

`abs.twimg.com` 我这里可以直接访问，但是有一个js文件控制着登陆框的密码输入，所以反代后替换掉域名，就可以输入密码了。

`pbs.twimg.com`是图片的域名，我这里可以直接访问，就不用反代了。

2014-09-29 18:24 更新：

>删除了`t.co`配置部分，因为如果替换域名`t.co`, 会连带替换到脚本中的一些其它东西，脚本会执行出错。

