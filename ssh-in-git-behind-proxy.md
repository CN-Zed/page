# 如何在使用SSH协议的git操作的时候使用代理服务器

由于我日常开发过程中经常使用Github，由于大家都懂的原因，不管是解析使用光棍DNS解析到美国，或者使用国内DNS解析到新加坡，国内直连Github的速度都不尽如人意。

当 `git clone` 一些大的库的时候非常慢，尤其是我日常使用zsh的时候，每次`cd`到git repo的目录时git prompt会非常卡导致体验相当差。

网上有很多教程都是针对https协议代理，但是我比较倾向于使用ssh协议，稍作了解以后得到了本文将要描述的解决方案。

## 1. Requirement

### `connect` 工具。

####  使用Debian系的发行版

直接通过apt安装

```bash
$ sudo apt install connect-proxy
```

#### 其他Linux发行版

查询各发行版的预编译包 [https://pkgs.org/download/connect-proxy](https://pkgs.org/download/connect-proxy)

#### 其他平台

请自行搜索`connect-proxy`或者其他替代工具。


## 2. 修改ssh config文件

使用你最爱的编辑器打开ssh config文件，通常这个文件都在用户根目录（*nix: ~ ; Windows: %UserProfile%）的.ssh文件夹下，有一个名为config的文件。

如果没有请先创建这个文件/文件夹。

```bash
$ vim ~/.ssh/config
```

然后添加如下内容到文件中：

```config
Host github.com
     Hostname github.com
     User git
     ProxyCommand connect -H 127.0.0.1:1080 %h %p
```

配置文件中的几个内容简单的介绍一下：
+ `Host`一个自定义的主机名
+ `Hostname`目标主机的地址，这里我们输入github.com
+ `User`ssh连接时使用的用户名，git over ssh使用的用户名通常都是git
+ `ProxyCommand`ssh的代理设置
    + `connect -H 127.0.0.1:1080 %h %p` 使用上面安装的connect-proxy工具，把%h:%p的TCP链接通过代理服务器转发。
    + `-H 127.0.0.1:1080`是指定一个代理，这里是说指定一个http代理，代理服务器地址是127.0.0.1，端口号是1080。（你懂的软件本地默认代理）
    + 或者可以使用`-S`链接sock代理服务


## 3. 测试

我这里使用的代理软件有详细的日志，可以通过该日志确定git over ssh是否正常通过代理连接。如果你使用别的代理软件，可以通过在远程代理服务器日志来查看是否有相关连接。

```log
[2018-09-11 16:40:22] connect to github.com:22
[2018-09-11 16:40:22] Socket connected to ss server: ***
```

最后让我们clone一下B神做的**宇宙第一编码/console字体**`更纱黑体`的库来测试一下速度。

```log
 zed  ~  time git clone git@github.com:be5invis/Sarasa-Gothic.git   

Cloning into 'Sarasa-Gothic'...
remote: Counting objects: 350, done.
remote: Total 350 (delta 0), reused 0 (delta 0), pack-reused 350
Receiving objects: 100% (350/350), 30.94 MiB | 2.54 MiB/s, done.
Resolving deltas: 100% (185/185), done.
git clone git@github.com:be5invis/Sarasa-Gothic.git  2.05s user 2.34s system 21% cpu 19.999 total

```