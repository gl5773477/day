# go get代理工具

>注：一切的前提是你有个代理，通常为socks5协议的代理服务器，有些包仅仅支持http或https，需要转换一下。

常用的四种工具： ShadowSockets、polipo、proxychains-ng、cow

## **0、最简单的方式：不使用任何工具**
```sh
http_proxy=x.x.x.x:port https_proxy=x.x.x.x:port go get
```
> 注：对于自己或公司的私有仓库，使用代理就取不到了，除非使用镜像。（特殊情况：私有仓库需要使用代理获取三方包，此时可以使用dep依赖奥管理工具或者手动git clone维护）

## **1、ShadowSockets**
和proxychains-ng一样出名，但配置要比一下三种麻烦，暂不介绍，参考[Mac+shadowsocks+polipo快捷实现](https://segmentfault.com/a/1190000008449046)。

## **2、proxychains-ng**

- 安装
```sh
# mac下
brew install proxychains-ng
```
- 配置代理：/usr/local/etc/proxychains.conf添加
```sh
# 服务地址和端口自己设置
socks5 x.x.x.x 1080
```
- 使用测试
```sh
# proxychains4 程序 参数
proxychains4 curl https://www.google.com
```
- 可能出现的问题
>有些程序，比如一些三方包，可能不支持socks5协议访问，此时需要做代理转换，将socks5请求转为http或https的请求，常用的方式就是polipo工具，既然如此，还是直接使用polipo吧。

虽然配置好了，但是使用时需要指定proxychains4，稍显麻烦，有人说通过`proxychains4 /bin/bash`开一个内终端使用，但我觉得还是麻烦，推荐使用下面两种方式之一。

## **3、polipo**

通过设置大部分程序都识别的的环境变量：http_proxy和https_proxy，将请求通过代理服务器以http或https的方式发出去。

- 安装
```sh
# mac下
brew install polipo
# linux
sudo apt-get install polipo
```

- 配置：
1. polipo软件配置：
mac下：/usr/local/opt/polipo/homebrew.mxcl.polipo.plist
```xml
...
<plist version="1.0">
    ...
    <array>
        <string>/usr/local/opt/polipo/bin/polipo</string>
        <!-- 添加的部分:填入自己拥有的socks5代理服务器地址和端口 -->
        <string>socksParentProxy=x.x.x.x:1080</string>
    </array>
    ...
</plist>
...
```

linux下：/etc/polipo/config
```sh
socksParentProxy = "x.x.x.x:1080"  
socksProxyType = socks5
```

2. 环境变量配置
```sh
export http_proxy="http://127.0.0.1:8123/"
export https_proxy="http://127.0.0.1:8123/"
```

- 启动
```sh
# 后台方式启动，linux下service polipo restart
polipo & # 监听在8123端口
# 测试
curl https://www.google.com
```

## **4、cow**

[[中文文档]](https://github.com/cyfdecyf/cow)内部貌似使用了shadowsocks机制。

- 安装
```sh
curl -L git.io/cow | bash
# 或者 mac下
brew install cow
```

- 配置
1. cow软件配置：$HOME/.cow/rc

```sh
#默认已添加
listen = http://127.0.0.1:7777 
# 填入自己拥有的socks5代理服务器地址和端口
proxy = socks5://x.x.x.x:1080
```
2. mac环境变量配置：~/.bash_profile
```sh
export http_proxy=http://127.0.0.1:7777
export https_proxy=http://127.0.0.1:7777
```

- 启动

```sh
# 后台方式启动
cow & # 监听在7777端口
# 测试
curl https://www.google.com
```
参考[mac 代理](https://www.jianshu.com/p/0bfe70cbd49c)。