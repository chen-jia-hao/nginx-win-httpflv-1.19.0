### 编译Windows版的带nginx-http-flv-module的nginx

#### 一.准备编译环境

- 基础编译环境：vs2019，安装时勾上c++桌面开发。
- Perl编译器：ActivePerl 5.26.3。这个在activestate官网上下载新的win64版本安装就行了。
- MSYS：去 http://xhmikosr.1f0.de/tools/msys/ 下一个`MSYS_MinGW-w64_GCC_710_x86-x64_Full.7z` 解压以后把bin目录加到path环境变量里

#### 二.准备nginx源码和依赖库

- ```shell
  git clone https://github.com/nginx/nginx.git
  ```

- ```shell
  git clone https://github.com/winshining/nginx-http-flv-module.git
  ```

需要nginx源码和扩展模块源码，

依赖库

- zlib
- pcre
- openssl

> 依赖库都可以在官网找到解压包

在nginx源码目录下，创建一个叫objs的目录，然后再在里面创建一个lib目录，然后将依赖解压在lib目录下，注意lib下的依赖不要有过多的目录，需点依赖包进去直接可以看见bin目录。还有`nginx-http-flv-module`也放到lib下。

目录结构截图如下

![](E:\tmp\n\nginx-win-httpflv-1.19.0\docs\pic1.png)

![](E:\tmp\n\nginx-win-httpflv-1.19.0\docs\pic2.png)

### 三.编译配置

参考官方编译指南的配置命令，并在最后加上`--add-module=objs/lib/nginx-http-flv-module`

依赖库的路径，正确指向objs/lib下的正确目录名的地方。

修改后的配置命令示例如下：

```shell
auto/configure \
    --with-cc=cl \
    --with-debug \
    --prefix= \
    --conf-path=conf/nginx.conf \
    --pid-path=logs/nginx.pid \
    --http-log-path=logs/access.log \
    --error-log-path=logs/error.log \
    --sbin-path=nginx.exe \
    --http-client-body-temp-path=temp/client_body_temp \
    --http-proxy-temp-path=temp/proxy_temp \
    --http-fastcgi-temp-path=temp/fastcgi_temp \
    --http-scgi-temp-path=temp/scgi_temp \
    --http-uwsgi-temp-path=temp/uwsgi_temp \
    --with-cc-opt=-DFD_SETSIZE=1024 \
    --with-pcre=objs/lib/pcre-8.40 \
    --with-zlib=objs/lib/zlib-1.2.11 \
    --with-openssl=objs/lib/openssl-1.1.1g \
    --with-openssl-opt=no-asm \
    --with-http_ssl_module \
    --add-module=objs/lib/nginx-http-flv-module
```

### 四.编译

在开始菜单的VS2019目录下可以找到“Developer Command Prompt for VS 2019”，启动它，并转到nginx源码目录下。

![](E:\tmp\n\nginx-win-httpflv-1.19.0\docs\pic3.png)

输入命令

```shell
nmake
```

开始编译

> 如果编译失败了，可以使用`nmake clean`清理源码目录。但是注意objs目录会被整个删掉，因此你需要重新创建objs/lib并重新往里放置依赖库

### 五.编译成功后

编译完成了之后，在objs目录下就会生成nginx.exe了

下载个win版nginx，用刚编译的nginx.exe替换原来的，再将nginx-http-flv-module目录下的stat.xsl拷贝到nginx-rtmp/html目录下。

配置conf/nginx.conf文件

```shell

worker_processes  1;
 
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;
#error_log  logs/error.log  debug;
 
#pid        logs/nginx.pid;
 
events {
    worker_connections  1024;
}

rtmp_auto_push on;
rtmp_auto_push_reconnect 1s;
rtmp_socket_dir temp;
 
# 添加RTMP服务
rtmp {
    server {
        listen 1935; # 监听端口
 
        chunk_size 4000;
        application live {
            live on;
			gop_cache on; # GOP缓存，on时延迟高，但第一帧画面加载快。off时正好相反，延迟低，第一帧加载略慢。
        }
    }
}
 
# HTTP服务
http {
    include       mime.types;
    default_type  application/octet-stream;
 
    #access_log  logs/access.log  main;
 
    server {
        listen       80; # 监听端口
		
		location / {
			add_header Access-Control-Allow-Origin *;
			add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
			add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';

			if ($request_method = 'OPTIONS') {
				return 204;
			}
			
            root html;
        }
		
		location /live {
            flv_live on; #打开HTTP播放FLV直播流功能
            chunked_transfer_encoding on; #支持'Transfer-Encoding: chunked'方式回复

            add_header 'Access-Control-Allow-Origin' '*'; #添加额外的HTTP头
            add_header 'Access-Control-Allow-Credentials' 'true'; #添加额外的HTTP头
        }
 
		location /stat.xsl {
            root html;
        }
		location /stat {
            rtmp_stat all;
            rtmp_stat_stylesheet stat.xsl;
        }
		
		location /control {
            rtmp_control all; #rtmp控制模块的配置
        }
		
    }
}

```

启动

> 双击nginx.exe即可启动

关闭

> WIN+R->cmd命令行进入nginx目录下执行 nginx.exe –s stop