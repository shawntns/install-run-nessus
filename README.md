# 假设运维已创建了账户privuser，该账户可以使用su+sudo，方法参考步骤0为例
## STEP0. Add the user 'privuser' using root account
```
[root@localhost ~]# useradd -r privuser
[root@localhost ~]# passwd username
[root@localhost ~]# usermod -aG wheel privuser
```
## STEP1. SSH login with 'privuser'
```
Shawns-MacBook-Pro:~ xxu$ ssh privuser@172.16.221.154
privuser@172.16.221.154's password:
Last failed login: Mon Mar  4 21:10:06 CST 2019 from 172.16.221.1 on ssh:notty
There was 1 failed login attempt since the last successful login.
Last login: Mon Mar  4 21:07:47 2019 from 172.16.221.1
-bash: warning: setlocale: LC_CTYPE: cannot change locale (UTF-8): No such file or directory
```
## STEP2. Install Nessus by using su + sudo
```
[privuser@localhost ~]$ su - privuser
Password:
Last login: Mon Mar  4 21:10:09 CST 2019 from 172.16.221.1 on pts/0
[privuser@localhost ~]$
[privuser@localhost ~]$ sudo rpm -ivh /tmp/Nessus-8.2.2-es7.x86_64.rpm --nosignature
/etc/host.conf: line 2: bad command `Nospoof on'
Preparing...                          ################################# [100%]
Updating / installing...
   1:Nessus-8.2.2-es7                 ################################# [100%]
Unpacking Nessus Core Components...
/etc/host.conf: line 2: bad command `Nospoof on'
 - You can start Nessus by typing /bin/systemctl start nessusd.service
 - Then go to https://localhost.localdomain:8834/ to configure your scanner
```
## STEP3. add the user 'nonprivuser' using 'privser'
```
[privuser@localhost ~]$ sudo useradd -r nonprivuser
```
## STEP4. Remove 'world' permissions on Nessus binaries in the /sbin directory.
```
[privuser@localhost ~]$ sudo chmod 750 /opt/nessus/sbin/*
```
## STEP5. Change ownership of /opt/nessus to the non-root user.
```
[privuser@localhost ~]$ sudo chown nonprivuser:nonprivuser -R /opt/nessus
```
## STEP6. Set capabilities on nessusd and nessus-service.
```
[privuser@localhost ~]$ sudo setcap "cap_net_admin,cap_net_raw,cap_sys_resource+eip" /opt/nessus/sbin/nessusd
[privuser@localhost ~]$ sudo setcap "cap_net_admin,cap_net_raw,cap_sys_resource+eip" /opt/nessus/sbin/nessus-service
```
## STEP7. Remove and add the following lines in **/usr/lib/systemd/system/nessusd.service**
* Remove:ExecStart=/opt/nessus/sbin/nessus-service-q
* Add:ExecStart=/opt/nessus/sbin/nessus-service -q --no-root
* Add:User=nonprivuser
```
[privuser@localhost ~]$ sudo vi /usr/lib/systemd/system/nessusd.service
[privuser@localhost ~]$ cat /usr/lib/systemd/system/nessusd.service
# ---------------------------------------------------- #
#                                                      #
# WARNING: DO NOT EDIT                                 #
#                                                      #
# This file has been autogenerated, edits to this file #
# directly may be overwriten during a build            #
#                                                      #
# ---------------------------------------------------- #


[Unit]
Description=The Nessus Vulnerability Scanner
After=network.target

[Service]
Type=simple
PIDFile=/opt/nessus/var/nessus/nessus-service.pid
ExecStart=/opt/nessus/sbin/nessus-service -q --no-root
Restart=on-abort
ExecReload=/usr/bin/pkill nessusd
EnvironmentFile=-/etc/sysconfig/nessusd
User=nonprivuser

[Install]
WantedBy=multi-user.target
Alias=nessusd.service
```
## STEP8. Reload nessusd.
```
[privuser@localhost ~]$ sudo systemctl daemon-reload
[privuser@localhost ~]$ sudo systemctl start nessusd
```
## 命令行安装完成！接下来请访问https://<ip>:8834做初始化与激活。
