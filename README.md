# 商业转载请联系作者获得授权，非商业转载请注明出处。
# For commercial use, please contact the author for authorization. For non-commercial use, please indicate the source.
# 协议(License)：署名-非商业性使用-相同方式共享 4.0 国际 (CC BY-NC-SA 4.0)
# 作者(Author)：Mooncore
# 链接(URL)：https://ucantsee.me/archives/164
# 来源(Source)：月下独酌

Caddy v1 + WebDav + Filebrowser 编译过程
发布于 2020-04-07  3,562 次阅读

Caddy是个Web服务器，配置比Nginx等要简单；WebDav用于使用客户端远程管理服务器文件，Filebrowser用于网页端管理文件。这一套下来，就可以把服务器当作网盘使用了。

Caddy最近开v2版本了，上v1官网一看，filebrowser插件又下架，又要自己编译了...

所以顺便记录一下，编译最新的Caddy v1 + WebDav + Filebrowser.

安装/更新Go (>1.13):
经我测试,Go版本高于1.14.1来编译caddy就会出各种问题,filebrowser和caddy-webdev都要用旧版代码以适应旧版gloang,比如filebrowser主项目源码用V2.0.5,改golang目录的

rm /usr/local/go -rf
wget https://studygolang.com/dl/golang/go1.14.1.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.14.1.linux-amd64.tar.gz && rm go1.14.1.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
export GO111MODULE=on
echo "export PATH=\$PATH:/usr/local/go/bin" >> /etc/profile
echo "export GO111MODULE=on" >> /etc/profile
新建一个文件夹用来存放编译文件

mkdir /home/caddy-build
cd /home/caddy-build
新建main.go文件

vim main.go
输入以下代码

package main
 
import (
        "github.com/caddyserver/caddy/caddy/caddymain"
 
        // plug in plugins here, for example:
        _ "github.com/filebrowser/caddy"
        _ "github.com/hacdias/caddy-webdav"
)
 
func main() {
        // optional: disable telemetry
        caddymain.EnableTelemetry = false
        caddymain.Run()
}
保存，然后执行

go mod init caddy
go get -v github.com/caddyserver/caddy@v1.0.5
go get -v github.com/filebrowser/caddy@3d5d96320b00936c1ddde393b24475d807b7ee65
这里会有错误：

/root/go/pkg/mod/github.com/mholt/caddy@v1.0.0/caddyhttp/httpserver/server.go:34:2: module github.com/lucas-clemente/quic-go@latest found (v0.15.2), but does not contain package github.com/lucas-clemente/quic-go/h2quic
这是因为filebrowser引用的是旧版本的caddy，而且mholt/caddy迁移到caddyserver/caddy了。

注：不要这么做

本来我们只要在go.mod中加入
replace github.com/mholt/caddy => github.com/caddyserver/caddy v1.0.5
就OK的，但是现在不行，会提示
go: github.com/caddyserver/caddy used for two different module paths (github.com/caddyserver/caddy and github.com/mholt/caddy)
这是Go的bug，在1.15中会修复（这个bug修了两年）
所以我们需要自己修改Filebrowser的源码

git clone https://github.com/filebrowser/caddy.git filebrowser
修改filebrowser/go.mod

        github.com/mholt/caddy v1.0.0
改为
        github.com/caddyserver/caddy v1.0.5
修改filebrowser/caddy.go

        "github.com/mholt/caddy"
        "github.com/mholt/caddy/caddyhttp/httpserver"
改为
        "github.com/caddyserver/caddy"
        "github.com/caddyserver/caddy/caddyhttp/httpserver"
修改filebrowser/parser.go

        "github.com/mholt/caddy"
        "github.com/mholt/caddy/caddyhttp/httpserver"
改为
        "github.com/caddyserver/caddy"
        "github.com/caddyserver/caddy/caddyhttp/httpserver"
然后修改自己的go.mod，加入新的一行，编译时使用本地的源码

replace github.com/filebrowser/caddy => /home/caddy-build/filebrowser
保存，然后执行下载WebDav插件

go get -v github.com/hacdias/caddy-webdav@42ae1f7bd1061e60322735362b63e1973c38f48b
又报错

# github.com/caddyserver/caddy/caddyhttp/httpserver
/root/go/pkg/mod/github.com/caddyserver/caddy@v1.0.5/caddyhttp/httpserver/tplcontext.go:281:14: undefined: blackfriday.HtmlRenderer
/root/go/pkg/mod/github.com/caddyserver/caddy@v1.0.5/caddyhttp/httpserver/tplcontext.go:283:11: undefined: blackfriday.EXTENSION_TABLES
/root/go/pkg/mod/github.com/caddyserver/caddy@v1.0.5/caddyhttp/httpserver/tplcontext.go:284:11: undefined: blackfriday.EXTENSION_FENCED_CODE
/root/go/pkg/mod/github.com/caddyserver/caddy@v1.0.5/caddyhttp/httpserver/tplcontext.go:285:11: undefined: blackfriday.EXTENSION_STRIKETHROUGH
/root/go/pkg/mod/github.com/caddyserver/caddy@v1.0.5/caddyhttp/httpserver/tplcontext.go:286:11: undefined: blackfriday.EXTENSION_DEFINITION_LISTS
/root/go/pkg/mod/github.com/caddyserver/caddy@v1.0.5/caddyhttp/httpserver/tplcontext.go:287:34: too many arguments to conversion to blackfriday.Markdown: blackfriday.Markdown([]byte(body), renderer, extns)
这是因为caddy自动引用的blackfriday包版本有问题。修改go.mod，加入新的一行

replace github.com/russross/blackfriday => github.com/russross/blackfriday v1.5.2
保存，然后执行

go mod why
查找一下依赖，输出

root@cloud1:/home/caddy-build# go mod why
# caddy
caddy
ok，现在可以开始编译

go build -o caddy
生成的可执行文件在当前目录(/home/caddy-build/caddy)

编译出来的文件下载(amd64):caddy.tar.gz

经测试成功编译可用的版本
批量编译bash脚本

#/bin/bash
for sysarch in `go tool dist list`
do
	systemtype=`echo ${sysarch} | cut -d \; -f 1`
	systemarch=`echo ${sysarch} | cut -d \; -f 2`
	CGO_ENABLED=0 GOOS=${systemtype} GOARCH=${systemarch} go build -ldflags='-w -s' -o caddy_${systemtype}_${systemarch}
done
echo 'All Jobs Done!'

====================================================================
caddy使用教程
https://dengxiaolong.com/caddy/zh/cli.html
第三方汉化了官方文档
================================================================================
自动ssl证书申请
dav.yourdomain.com {
 #申请免费SSL证书
 tls 你的邮箱
 #根目录
 root /usr/local/caddy/www/file/dav
 #给网站目录添加简单的登录机制（账号admin密码password自行修改）
 basicauth / admin password
 timeouts none
 gzip
 webdav / {
  scope /usr/local/caddy/www/file/dav #webdav的真实目录,在windows里这个不能设置为任何一个分区的根目录
  allow /
 }
}
================================================================================================
filemanager的插件配置

filemanager [url] [scope] {
 database path
}
url 是要设置的网站URL。默认是 / (比如 /doubi 那么访问入口就是 http://ip/doubi )。
scope 是要浏览的服务器文件目录路径，可以使相对或绝对路径。默认是 ./ (服务器上面文件的绝对或相对路径)。
database path 是 filemanager 的数据库路径（如果不写这个参数，则默认就是 /usr/local/caddy/filemanager.db）。
=============================================================================================================

设置自启
适用于systemd，如debian8以上,ubuntu16以上
filebrowser的自启
编辑vi /etc/systemd/system/filebrowser.service

[Unit]
Description=filebrowser
Requires=network.target
 
[Service]
Type=forking
ExecStart=/usr/local/bin/filebrowser -r 文件路径 -p 8080
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
执行

systemctl start filebrowser && systemctl enable filebrowser
=============================================================================================
可以使用 forwardproxy ,一个http/https的代理插件,可以直接开https代理,但是会被秒封
forwardproxy {
       basicauth 你的用户名 你的密码
    }
=============================================================================================
webdav插件官方指南:

webdav [url] {
    scope       path
    modify      [true|false]
    allow       path
    allow_r     regex
    block       path
    block_r     regex
}
所有选项都是可选的。

url 是你可以访问WebDAV接口的地方。默认为/。
scope 是指示WebDAV范围的一个绝对路径或相对路径(与Caddy当前工作目录相对)。默认为..
modify 指示用户是否有权编辑/修改文件。默认值为true。
allow和block 用于允许或拒绝使用特定文件或目录到作用域的相对路径访问它们。您可以使用魔术单词dotfiles来允许或拒绝访问以点开始的每个文件。
allow_r和block_r 是前面选项的变体，你可以对它们使用正则表达式。
强烈推荐将这个指令和basicauth一起来保护WebDAV接口。

webdav {
    # 这里放置全局配置
    # 所有用户都会继承他们
    user1:
    # 你可以在这里为`user1`放置特殊的设置
    # 它们将覆盖这个特定用户的全局变量。
}
基本

webdav
通过/访问的当前工作目录的WebDAV。

自定义范围

webdav {
    scope /admin
}
通过/admin访问的整个文件系统的WebDAV。

拒绝规则

webdav {
    scope /
    block /etc
    block /dev
}
通过/访问的整个文件系统的WebDAV，不能访问/etc和/dev目录。

用户权限

basicauth / sam pass
webdav {
    scope /
    
    sam:
    block /var/www
}
通过/访问的整个文件系统的WebDAV。用户sam不能访问/var/www，但其他用户可以。
=================================================================================================

http.basicauth
basicauth实现HTTP基本身份验证。基本身份验证可用于通过用户名和密码保护目录和文件。注意，基本身份校验在纯HTTP协议访问时也是不安全的。请谨慎考虑决定使用HTTP基本身份验证来保护什么。

当用户请求受保护的资源时，如果用户尚未提供用户名和密码，浏览器将提示用户输入用户名和密码。如果Authorization头中存在正确的凭证，服务器将授予对资源的访问权，并将{user}占位符设置为用户名的值。如果头信息丢失或凭证不正确，服务器将返回"HTTP 401 Unauthorized"。

此指令允许使用.htpasswd文件，方法是在密码参数前面加上htpasswd=和要使用的.htpasswd文件的路径。

对.htpasswd的支持只针对遗留站点，将来可能会被删除;不要在新站点上使用.htpasswd。

语法
basicauth path username password
path 要保护的文件或目录
username 用户名
password 密码
这种语法使用默认范围(realm)"Restricted"，对于保护单个文件或基本路径/目录非常方便。为了保护多个资源或指定一个范围，使用下面的变体：

basicauth username password {
    realm name
    resources
}
username 用户名
password 密码
realm 标识保护范围；它是可选的，不能重复。realm用于指定应用保护的范围。对于配置为记住身份验证细节(大多数浏览器都是这样)的用户代理来说非常方便。
resources 要保护的文件/目录列表，每行一个。
例子
保护"/secret"下所有文件，只有Bob使用密码"hiccup"才能访问：

basicauth /secret Bob hiccup
使用范围"Mary Lou's documents"保护多个文件和目录，用户"Mary Lou"可以使用她的密码"milkshakes"访问：

basicauth "Mary Lou" milkshakes {
    realm "Mary Lou's documents"
    /notes-for-mary-lou.txt
    /marylou-files
    /another-file.txt
}
===============================================================================================
