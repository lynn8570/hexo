---
title: "gitbook知识服务共享平台搭建"
date: "2018-03-15"
categories: 
    - "SERVER"
---



# git 服务器搭建 #

主要参考[廖雪峰搭建git服务器](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/00137583770360579bc4b458f044ce7afed3df579123eca000)一文主要注意的几个点：

1. 创建的 home/git/.ssh/ 目录的权限和 home/git/.ssh/authorized_keys 的权限问题
   ``` -rw-r--r-- 1 git git 1234 2018-03-10 17:43 authorized_keys
   root@guoyzh-ubuntu:/home/git/.ssh# ls -all
   ......
   -rw-r--r-- 1 git git 1234 2018-03-10 17:43 authorized_keys

    drwxr-xr-x  2 git  git  4096 2018-03-10 17:43 .ssh
   ```


2. /etc/ssh/sshd_config 文件可用于配置验证方式

   ```
   # Package generated configuration file
   # See the sshd_config(5) manpage for details

   # What ports, IPs and protocols we listen for
   Port 22
   # Use these options to restrict which interfaces/protocols sshd will bind to
   #ListenAddress ::
   #ListenAddress 0.0.0.0
   Protocol 2
   # HostKeys for protocol version 2
   HostKey /etc/ssh/ssh_host_rsa_key
   HostKey /etc/ssh/ssh_host_dsa_key
   #Privilege Separation is turned on for security
   UsePrivilegeSeparation yes

   # Lifetime and size of ephemeral version 1 server key
   KeyRegenerationInterval 3600
   ServerKeyBits 768

   # Logging
   SyslogFacility AUTH
   LogLevel DEBUG3

   # Authentication:
   LoginGraceTime 120
   PermitRootLogin yes
   StrictModes yes

   RSAAuthentication no
   PubkeyAuthentication yes   #使用公钥验证
   AuthorizedKeysFile	/home/git/.ssh/authorized_keys #公钥文件地址，使我们创建的，并且需要把各个用户的公钥收集起来，然后黏贴到这个文件中
   # Don't read the user's ~/.rhosts and ~/.shosts files
   IgnoreRhosts yes
   # For this to work you will also need host keys in /etc/ssh_known_hosts
   RhostsRSAAuthentication no
   # similar for protocol version 2
   HostbasedAuthentication no
   # Uncomment if you don't trust ~/.ssh/known_hosts for RhostsRSAAuthentication
   #IgnoreUserKnownHosts yes

   # To enable empty passwords, change to yes (NOT RECOMMENDED)
   PermitEmptyPasswords no

   # Change to yes to enable challenge-response passwords (beware issues with
   # some PAM modules and threads)
   ChallengeResponseAuthentication no

   # Change to no to disable tunnelled clear text passwords
   #PasswordAuthentication no  #允许使用密码验证

   # Kerberos options
   #KerberosAuthentication no
   #KerberosGetAFSToken no
   #KerberosOrLocalPasswd yes
   #KerberosTicketCleanup yes

   # GSSAPI options
   #GSSAPIAuthentication no
   #GSSAPICleanupCredentials yes

   X11Forwarding yes
   X11DisplayOffset 10
   PrintMotd no
   PrintLastLog yes
   TCPKeepAlive yes
   #UseLogin no

   #MaxStartups 10:30:60
   #Banner /etc/issue.net

   # Allow client to pass locale environment variables
   AcceptEnv LANG LC_*

   Subsystem sftp /usr/lib/openssh/sftp-server

   # Set this to 'yes' to enable PAM authentication, account processing,
   # and session processing. If this is enabled, PAM authentication will
   # be allowed through the ChallengeResponseAuthentication and
   # PasswordAuthentication.  Depending on your PAM configuration,
   # PAM authentication via ChallengeResponseAuthentication may bypass
   # the setting of "PermitRootLogin without-password".
   # If you just want the PAM account and session checks to run without
   # PAM authentication, then enable this but set PasswordAuthentication
   # and ChallengeResponseAuthentication to 'no'.
   UsePAM yes
   ```

3. 设置不允许通过shell登录git用户账号

   ```
   guoyzh:x:1000:1000:guoyzh,,,:/home/guoyzh:/bin/bash
   sshd:x:115:65534::/var/run/sshd:/usr/sbin/nologin
   huangyc:x:1001:0:huangyc,,,,:/home/huangyc:/bin/bash
   git:x:1002:1002:,,,:/home/git:/usr/bin/git-shell  #改成/usr/bin/git-shell
   ```

4. 测试ssh 登录，查找问题的时候可以使用如下命令

   ```
    ssh -vT git@10.24.9.244
    OpenSSH_7.5p1, OpenSSL 1.0.2k  26 Jan 2017
    debug1: Reading configuration data /etc/ssh/ssh_config
    debug1: Connecting to 10.24.9.244 [10.24.9.244] port 22.
    debug1: Connection established.
   ```

5. 其他用户使用 `git clone git@10.24.9.244:/home/gitrepo/zoweedoc.git`即可clone仓库


# 虚拟机

为了不破坏服务器上的环境，安装了个虚拟机来做下面的事情

虚拟机安装：

1.安装VMware workstations

2.下载Ubuntu镜像

## 安装npm,nodejs

Node.js 是一个基于 Chrome V8 引擎的 JavaScript 运行环境，其使用了一个事件驱动、非阻塞式 I/O 的模型，使其轻量又高效。 
Node.js 的包管理器 npm，是全球最大的开源库生态系统，功能及其强大。 

1. 安装nvm

    sudo git clone https://github.com/cnpm/nvm.git

    `vim ~/.bashrc`

   添加 source ~/git/nvm/nvm.sh  配置终端启动时自动执行 `source ~/git/nvm/nvm.sh`

2. 通过nvm安装node

   `nvm ls-remote` 查看远程版本

   最新的release版本

   ```
    lynn@ubuntu:~$ nvm --version
    0.26.1
   ```

   先安裝一個

   nvm install v9.8.0

   ```
   2018-03-13 20:23:26 (1.91 MB/s) - ‘/home/lynn/git/nvm/bin/node-v9.8.0-linux-x64/node-v9.8.0-linux-x64.tar.gz’ saved [18071050/18071050]

   WARNING: checksums are currently disabled for node.js v4.0 and later
   Now using node v9.8.0 (npm v5.6.0)
   ```

3. Node.js 的包管理器 npm，但是据说比较慢，可以用cnpm来加速

   `npm install koa --registry=http://registry.npm.taobao.org`

4. 关闭终端后，输入npm无效了

   ```
   lynn@ubuntu:~/git/nvm$ nvm use v9.8.0
   Now using node v9.8.0 (npm v5.6.0)
   ```

## gitbook

### 安装gitbook

利用npm安装gitbook

```
 npm install gitbook-cli -g
/home/lynn/git/nvm/versions/node/v9.8.0/bin/gitbook -> /home/lynn/git/nvm/versions/node/v9.8.0/lib/node_modules/gitbook-cli/bin/gitbook.js
+ gitbook-cli@2.3.2
added 578 packages in 116.801s
```

### 添加demo gitbook

```
mkdir demo
cd demo
gitbook init
.......
warn: no summary file in this book 
info: create README.md 
info: create SUMMARY.md 
info: initialization is finished 
```

自动生成了readme 和summary

### 生成html静态网页

```
gitbook build
```

多了一个`_book`的目录

打开index.html就可以查看书籍内容

### 启动服务gitbook serve

```
lynn@ubuntu:~/gitbook/zoweedoc$ gitbook serve
Live reload server started on port: 35729
Press CTRL+C to quit ...

info: 7 plugins are installed 
info: loading plugin "livereload"... OK 
info: loading plugin "highlight"... OK 
info: loading plugin "search"... OK 
info: loading plugin "lunr"... OK 
info: loading plugin "sharing"... OK 
info: loading plugin "fontsettings"... OK 
info: loading plugin "theme-default"... OK 
info: found 1 pages 
info: found 11 asset files 
info: >> generation finished with success in 1.9s ! 

Starting server ...
Serving book on http://localhost:4000
```

可通过浏览器访问gitbook了

# push触发

现在我们可以生成gitbook了，如何在zoweedoc.git有push工作后，立即同步所有的更新，再出发gitbook的build，生成最新的网页代码呢？



## Git 钩子

和其它版本控制系统一样，Git 能在特定的重要动作发生时触发自定义脚本。 有两组这样的钩子：客户端的和服务器端的。 客户端钩子由诸如提交和合并这样的操作所调用，而服务器端钩子作用于诸如接收被推送的提交这样的联网操作。 你可以随心所欲地运用这些钩子。

1. 在服务器上建一个普通的Git仓库，用于存放网站
2. 在源git的post-receive中执行git pull的操作，实现文档更新，然后可以重新build 

但是现在我们的git服务器环境太老了，很多东西没办法安装，上述用hook的方式进行更新，无法实现

采用在另外一台机子上，采用定时更新

## 定时更新

- 查看当前定时任务

  crontab -l

- 创建编辑定时任务

  crontab -e

- 编辑定时任务

  `* */1 * * * /home/lynn/gitbook/syn.sh /home/lynn/gitbook/zoweedoc`

  脚本

  ```
  #!/bin/sh
  source ~/git/nvm/nvm.sh
  cd $1
  unset GIT_DIR
  git pull origin master

  nvm use v9.8.0
  gitbook serve
  ```

- 查看定時任務執行情況

  看 /var/log/cron这个文件就可以，可以用tail -f /var/log/cron观察

- 定時任務有沒有啟動排查的

  - 确认服务器是否开机定时任务计划服务

    ```
    lynn@ubuntu:/var/log$ service crond status
    ● crond.service
       Loaded: not-found (Reason: No such file or directory)
       Active: inactive (dead)
    ```

  - 开启服务

    ```
    lynn@ubuntu:~$ service crond start
    Failed to start crond.service: Unit crond.service not found.
    ```

  - 新的服务名字变了

    ```
    sudo service cron  start #启动服务
    lynn@ubuntu:~$ service cron status
    ● cron.service - Regular background program processing daemon
       Loaded: loaded (/lib/systemd/system/cron.service; enabled; vendor preset: ena
       Active: active (running) since Wed 2018-03-14 00:49:42 PDT; 1h 47min ago
         Docs: man:cron(8)
     Main PID: 793 (cron)
       CGroup: /system.slice/cron.service
               └─793 /usr/sbin/cron -f

    Mar 14 02:15:01 ubuntu CRON[3456]: (CRON) info (No MTA installed, discarding out
    Mar 14 02:15:01 ubuntu CRON[3456]: pam_unix(cron:session): session closed for us
    Mar 14 02:17:01 ubuntu CRON[3474]: pam_unix(cron:session): session opened for us
    Mar 14 02:17:01 ubuntu CRON[3475]: (root) CMD (   cd / && run-parts --report /et
    Mar 14 02:17:01 ubuntu CRON[3474]: pam_unix(cron:session): session closed for us
    Mar 14 02:30:01 ubuntu CRON[3647]: pam_unix(cron:session): session opened for us
    Mar 14 02:30:01 ubuntu CRON[3648]: (lynn) CMD (/home/lynn/gitbook/syn.sh /home/l
    Mar 14 02:30:01 ubuntu CRON[3647]: (CRON) info (No MTA installed, discarding out
    Mar 14 02:30:01 ubuntu CRON[3647]: pam_unix(cron:session): session closed for us
    Mar 14 02:36:28 ubuntu systemd[1]: Started Regular background program processing
    lines 1-18/18 (END)

    ```

  - ​

定时更新http://blog.csdn.net/csdn_yasin/article/details/70332796

查看执行情况

http://blog.csdn.net/u013850277/article/details/54344805

## 访问虚拟机服务器IP

### 本机访问虚拟机服务器

虚拟机通过ifconfig查询虚拟机IP

```
lynn@ubuntu:~$ ifconfig
ens33     Link encap:Ethernet  HWaddr 00:0c:29:5d:64:1f  
          inet addr:192.168.32.128  Bcast:192.168.32.255  Mask:255.255.255.0
          inet6 addr: fe80::aae1:44b2:cc44:dbba/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:101881 errors:0 dropped:0 overruns:0 frame:0
          TX packets:33601 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:121150102 (121.1 MB)  TX bytes:2044329 (2.0 MB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:234 errors:0 dropped:0 overruns:0 frame:0
          TX packets:234 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:18513 (18.5 KB)  TX bytes:18513 (18.5 KB)
```

本机可直接通过，访问虚拟机服务器，查看gitbook

```
http://192.168.32.128:4000/
```



### 其他机子如何访问

1. 将虚拟机设置成NAT模式
2. 启动虚拟网络编辑器，选择VMnet8，选择NAT模式，选择NAT设置
3. 点击添加，主机端口8888，即希望别的机子访问的端口号`http://主机IP:8888/`，虚拟机地址，虚拟机实际IP地址，ifconfig查看，`192.168.32.128`，虚拟机端口，因为gitbook serve端口是4000，所以填4000
4. 完成确定

这样在其他主机就可以通过`http://主机IP:8888/`访问主机上虚拟机的4000端口服务了。

参考设置[如何配置其他主机访问虚拟机](https://jingyan.baidu.com/article/8065f87fe9773b23312498a9.html)



# md文件编辑器

推荐使用typora，window版本的软件安装包地址 `\\10.24.9.247\360Downloads\.01应用组共享资料\09软件工具\typora-setup-x64.exe` 

