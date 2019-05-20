---
title: svn-http环境搭建及配置
date: 2019-03-01 10:12:41
tags:
 - svn
 - svn-http
 - helm
 - kubernetes
 - docker
categories:
 - 工具
 - svn
---
本文介绍svn-http环境搭建及如何配置权限。
<!--more-->在项目文档管理领域svn一直是带头大哥，接下来我们一起看看怎么用云服务器部署svn并且进行权限配置

# 编译svn镜像
svn有两种访问协议svn协议和http协议。这里我们选用http协议，能够在web端显示文档库。
这里我自己写的dockerfile，基于alpine-base轻量级的linux内核，安装mod_dav_svn（http协议需要）和apache。
具体的dockerfile参考[这里](https://github.com/scutjason/tools/blob/master/svn-docker/Dockerfile)。

# 部署svn
我这里采用kubernetes阿里云部署，利用helm工具。主要是下面几个参数说明一下，在service.yaml中
```
spec:
  type: LoadBalancer
  ports:
    -  port: 80
      targetPort: 80    # 这个端口是访问http协议的端口
      protocol: TCP
      name: http
    -  port: 3960        # 这个端口是svn协议的端口 -- 对应 dockerfile中svnserve 的listen-port
      targetPort: 3960
      protocol: TCP
      name: svn
```

执行命令
```
  helm install ./svn_helm --namespace=myspace -n svn
```
查看是否部署成功
```
PS C:\Users\scutjason> helm ls
svn                     1               Mon Mar  4 16:46:21 2019        DEPLOYED        svn-1.0.3               1.0             myspace

PS C:\Users\scutjason> kubectl get pods --namespace=myspace
svn-8584bc7df9-7zw26                 1/1       Running       0

PS C:\Users\scutjason> kubectl get services --namespace=finmind
svn                 LoadBalancer   172.41.16.160    112.168.165.225   80:30855/TCP,3960:30166/TCP   1d

112.168.165.225 这个ip就是我们svn的外网http地址。打开浏览器输入http://112.168.165.225/svn 就能看到页面了，虽然里面啥都没有，因为我们还没有创建仓库。
```
ok部署成功。
# 创建svn仓库
在cmd中执行下面的命令进入svn容器。
```
kubectl exec -it svn-8584bc7df9-7zw26  /bin/sh --namespace=myspace
svnadmin create /home/svn/repo    # svn仓库默认地址在/home/svn目录，我们web能访问到的路径也在/home/svn这一目录。
```
ok，重新打开浏览器输入http://112.168.165.225/svn 就应该能看到有个repo文件夹了。

这里可能需要改变一下仓库目录的权限。
```
chown -R apache:apache /home/svn/
```

# 配置svn用户名和密码
接下来配置svn的用户密码
```
htpasswd -b /etc/subversion/passwd scutjason 123456
```
配置一个scutjason的用户，其密码是123456，如果想添加其他用户和密码，也是执行这条命令。
加下来在浏览器输入http://112.168.165.225/svn/repo 或者用TortoiseSVN checkout repo仓库的时候会提示输入用户名和密码。
然后我们输入上面配置的用户名和密码后就能访问或者checkout了。

# 配置svn仓库的访问权限
具体的权限控制目录在/etc/subversion/subversion-access-control文件。用vi打开编辑即可。
```
user = scutjason, scutsylvia # 配置user组里面有两个人scutjason 和 scutsylvia
[repo:/]    # 对应repo仓库目录下的所有目录
@user = rw  # user这个组的人都有rw权限

[repo:/code]    # 对于repo仓库下的code目录
scutjason = rw  # scutjason 有读写权限
scutsylvia = r  # scutsylvia 有读的权限
* =             # 除了scutjason 和 scutsylvia外的所有人没有任何权限
```

# svn的后台服务
```
/ # top
Mem: 7665112K used, 345084K free, 26764K shrd, 249712K buff, 1479368K cached
CPU:   2% usr   0% sys   0% nic  97% idle   0% io   0% irq   0% sirq
Load average: 0.01 0.09 0.12 2/1224 409
  PID  PPID USER     STAT   VSZ %VSZ CPU %CPU COMMAND
  400   234 apache   S    98324   1%   3   0% /usr/sbin/httpd -DFOREGROUND
  401   234 apache   S    98324   1%   1   0% /usr/sbin/httpd -DFOREGROUND
  399   234 apache   S    98324   1%   3   0% /usr/sbin/httpd -DFOREGROUND
  397   234 apache   S    98324   1%   3   0% /usr/sbin/httpd -DFOREGROUND
  398   234 apache   S    98324   1%   1   0% /usr/sbin/httpd -DFOREGROUND
  234   231 root     S    98312   1%   2   0% /usr/sbin/httpd -DFOREGROUND
  224   216 go-dnsma S     9512   0%   1   0% go-dnsmasq --default-resolver --ndots 1 --fwd-ndots 0 --hostsfile=/etc/hosts
  271     0 root     S     1540   0%   0   0% /bin/sh
  402     0 root     S     1540   0%   2   0% /bin/sh
  264     0 root     S     1540   0%   0   0% /bin/sh
  231    35 root     S     1536   0%   3   0% {apachectl} /bin/sh /usr/sbin/apachectl -DFOREGROUND
  216   213 root     S     1532   0%   0   0% sh ./run
  409   402 root     R     1524   0%   3   0% top
    1     0 root     S      184   0%   0   0% s6-svscan -t0 /var/run/s6/services
  213     1 root     S      184   0%   1   0% s6-supervise resolver
   30     1 root     S      184   0%   1   0% s6-supervise s6-fdholderd
   29     1 root     S      172   0%   0   0% foreground  if   /etc/s6/init/init-stage2-redirfd   foreground    if     if      s6-echo      -n      --      [s6-init] making user provided files available at /var/
   35    29 root     S      168   0%   2   0% foreground  s6-setsid  -gq  --  with-contenv  /usr/sbin/apachectl  -DFOREGROUND  importas -u ? ? if  s6-echo  --  /usr/sbin/apachectl exited ${?}  foreground  redirf
/ #
```


```
/ # ps
PID   USER     TIME   COMMAND
    1 root       0:00 s6-svscan -t0 /var/run/s6/services
   29 root       0:00 foreground  if   /etc/s6/init/init-stage2-redirfd   foreground    if     if      s6-echo      -n      --      [s6-init] making user provided files available at /var/run/s6/etc...
   30 root       0:00 s6-supervise s6-fdholderd
   35 root       0:00 foreground  s6-setsid  -gq  --  with-contenv  /usr/sbin/apachectl  -DFOREGROUND  importas -u ? ? if  s6-echo  --  /usr/sbin/apachectl exited ${?}  foreground  redirfd  -w  1  /var/run/s6/e
  213 root       0:00 s6-supervise resolver
  216 root       0:00 sh ./run
  224 go-dnsma   0:00 go-dnsmasq --default-resolver --ndots 1 --fwd-ndots 0 --hostsfile=/etc/hosts
  231 root       0:00 {apachectl} /bin/sh /usr/sbin/apachectl -DFOREGROUND
  234 root       0:00 /usr/sbin/httpd -DFOREGROUND
  264 root       0:00 /bin/sh
  271 root       0:00 /bin/sh
  397 apache     0:00 /usr/sbin/httpd -DFOREGROUND
  398 apache     0:00 /usr/sbin/httpd -DFOREGROUND
  399 apache     0:00 /usr/sbin/httpd -DFOREGROUND
  400 apache     0:00 /usr/sbin/httpd -DFOREGROUND
  401 apache     0:00 /usr/sbin/httpd -DFOREGROUND
  402 root       0:00 /bin/sh
  410 root       0:00 ps
```
