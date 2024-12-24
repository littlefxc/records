---
status: Done
---


## 开机自启动

1. 在wsl中创建一个启动配置文件，里面填写上要启动的服务。

```bash
sudo vi /etc/init.wsl
```

```bash
#! /bin/sh
service docker start
service mysql start
```

2. 赋予文件可执行权限

```bash
chmod +x /ect/init.wsl
```

    1. 在windows系统上创建一个启动脚本startup.vbs。（我个人还是喜欢手动启动）

```
Set ws = WScript.CreateObject("WScript.Shell")        
ws.run "wsl -d Ubuntu-22.04 -u root /etc/init.wsl",0
```

4. 将启动脚本添加到启动目录。(通过win+r，输入 shell:startup进入 ）。

PS：每次在wsl启动的时候都会加载.bashrc文件，因此我们也可以把服务启动放到.bashrc文件里面，这样打开wsl的时候，服务也会自动启动。

## 启用 systemd

目前需要我们手动配置开启 systemd，即在 WSL 启动时执行 `wsl-systemd` 即可：

```
sudo vim /etc/wsl.conf
```

在这份配置文件中加上下面的配置：

```
# /etc/wsl.conf
[boot]
command="/usr/libexec/wsl-systemd"
```

如果上面的配置最终无法成功工作，可以尝试下面另一种配置方法

```
# /etc/wsl.conf
[boot]
systemd=true
```

## 启用 snap

现在 Ubuntu 的战略似乎都转向用 snapcraft 打包应用，很多软件在 apt 官方源内的版本都不是最新，甚至有的都不再提供 apt 安装方式

但是，snap 的使用需要依赖 systemd，而这个在 WSL2 上没有正常开启，导致我们在使用 snap 时会报错：

>error: cannot communicate with server: Post http://localhost/v2/snaps/xxx: dial unix /run/snapd.socket: connect: no such file or directory

目前需要我们手动配置开启 systemd，即在 WSL 启动时执行 `wsl-systemd` 即可：

```
sudo vim /etc/wsl.conf
```

在这份配置文件中加上下面的配置：

```
# /etc/wsl.conf
[boot]
command="/usr/libexec/wsl-systemd"
```

如果上面的配置最终无法成功工作，可以尝试下面另一种配置方法

```
# /etc/wsl.conf
[boot]
systemd=true
```

最后重启 WSL2，即可正常使用 snap：
![[snaplist.png]]

但到这里其实还只是能使用 snap 的命令，并不能正常使用 snap 安装的软件包，还需要把 snap 安装的软件二进制可执行文件的文件夹放到环境变量中，然后重启终端：

```
# 注意使用单引号不是双引号，或者直接用编辑器修改对应 bash 或 zsh 的配置文件也可以
echo 'export PATH=$PATH:/snap/bin\n' >> ~/.zshrc
```

## 安装 redis

```shell
# 安装 redis
snap install redis
# 启动 redis
snap start redis
# 关闭 reids
snap stop redis
# 移除 redis
snap remove redis
```

## 参考资源

[# WSL2 Ubuntu 22.04 安装踩坑记录](https://blog.csdn.net/qq_32114645/article/details/124548058)