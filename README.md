## rsync+inotify实现服务器之间同步备份



#### 一、检查系统内核是否支持 inotify

###### 1、uname -r  

###### 2、ll /proc/sys/fs/inotify

###### 出现以下三个文件表示系统默认支持 inotify

```
 -rw-r--r-- 1 root root 0 Mar 11 09:34 max_queued_events

 -rw-r--r-- 1 root root 0 Mar 11 09:34 max_user_instances

 -rw-r--r-- 1 root root 0 Mar 11 09:34 max_user_watches
```



#### 二、整体架构

这里我使用两个 Linux 服务器节点来做演示，实现两个节点间文件的实时同步，node1 为源服务器节点，就是需要同步数据的节点，部署 rsync+inotify ，node2 为同步节点，也就是接收同步数据的节点，只需要部署 rsync，如下所示。

| 主机  |    节点名    |     系统     |       ip        |  角色  |      软件      |
| :---: | :----------: | :----------: | :-------------: | :----: | :------------: |
| node1 | 源服务器节点 | CentOS7 64位 | 192.168.157.129 | Server | rsync，inotify |
| node2 |   同步节点   | CentOS7 64位 | 192.168.157.130 | Client |     rsync      |

###### 注：node2是接收服务器，所有只需要rsync



#### 三、同步节点部署（rsync）

###### 注：接收数据的服务器

同步节点，也就是node2 192.168.157.130，只需要安装配置 rsync 即可，具体步骤如下。



##### 1、安装rsync

（1）直接使用yum命令安装

```
	yum -y install rsync
```

（2）安装完成后，使用`rsync –-help`命令可查看 rsync 相关信息



##### 2、配置rsync

rsync 安装好之后，在 etc 目录下有一个 rsyncd.conf 文件，修改rsync配置文件。

```
vim /etc/rsyncd.conf
```

修改内容如下

```
uid = nobody
gid = nobody
use chroot = yes
max connections = 10
strict mode=yes
pid file = /var/run/rsyncd.pid
lock file=/var/run/rsync.lock
log file=/var/log/rsyncd.log
[backup]
        path = /home/test/ #从主服务器同步到那个目录
        comment = backup file
        ignore errrors
        read only=no
        write only=no
        hosts allow=192.168.10.2 #主服务器地址
        hosts deny=*
        list=false
        uid=root
        gid=root
        auth users=yxc #用户名
        secrets file=/etc/rsync.password #密码文件
```



其中需要用到一个密码文件，也就是上面配置的 secrets file 的值

```
 /etc/rsync.password
```



在 rsync3.1.3 版本中默认没有密码文件，需要手动创建，内容格式为：user:password，user 就是上面配置的 yxc，password 就是密码，如下所示。

```
echo "yxc:123456" > /etc/rsync.password
```



然后需要给密码文件600权限

```
chmod 600 /etc/rsync.password
```


启动 rsync 守护进程

```
/usr/local/bin/rsync --daemon
```



启动之后可查看 rsync 进程，如下

```
ps -ef | grep rsync
```



如有需要可加入系统自启动文件

```
echo "/usr/local/bin/rsync --daemon" >> /etc/rc.local
```


rsync 默认端口为873，所以开放873端口

```
firewall-cmd --add-port=873/tcp --permanent --zone=public
```



重启防火墙(修改配置后要重启防火墙)

```
firewall-cmd --reload
```



#### 四、源服务器节点部署（rsync+inotify）

###### 注：发送数据的服务器和跑脚本的服务器

源服务器节点，也就是 node1 192.168.157.129，需要部署 rsync 和 inotify。



#### 1、安装inotify-tools

```
yum install inotify-tools -y
```



##### 2、安装rsync

```
yum -y install rsync
```



##### 3、配置rsync

源服务器节点中只需要配置认证密码文件，首先在 etc 文件夹下创建文件 rsync.password，只需要密码，不需要用户，密码需要和同步节点（也就是接收服务器） node2 中的一致，我这里也就是123456。

```
vim /etc/rsync.password
```



#需要给密码文件600权限

```
chmod 600 /etc/rsync.password
```


启动 rsync 守护进程

```
/usr/local/bin/rsync --daemon
```


如有需要可加入系统自启动文件

```
echo "/usr/local/bin/rsync --daemon" >> /etc/rc.local
```


同样开放873端口，如下所示

```
firewall-cmd --add-port=873/tcp --permanent --zone=public
```



#重启防火墙(修改配置后要重启防火墙)

```
firewall-cmd --reload
```

 

##### 3、手动同步测试

先在源服务器节点的 /root/data/backuptest/ 目录下新建一个 test 文件夹

```
mkdir data/backuptest/test
```


然后使用如下命令进行同步测试，其中一些参数要和同步节点配置文件中相对应，比如下面的认证模块名 backup、用户名 yxc 等。

```php
rsync -avH --port 873 --delete /root/data/backuptest/ yxc@192.168.157.130::backup --password-file=/etc/rsync.password
```

##### 4、部署inotify

###### （2）创建rsync同步的shell脚本

发送数据服务器：node1-192.168.157.129

在需要同步文件的目录建立

```
sheel.sh
```

sheel.sh：内容如下

```sh
#!/bin/bash
src=/www/text/ #需要同步的目录
des=backup
rsync_passwd_file=/etc/rsync.password #密码文件
ip1=192.168.10.3 #需要同步的服务器1
ip2=192.168.10.4 #需要同步的服务器2
user=yxc #用户名需要同步数据服务器中的[backup]中的用户名
cd ${src}
/usr/local/bin/inotifywait -mrq --format  '%Xe %w%f' -e modify,create,delete,attrib,close_write,move ./ | while read file
do
        INO_EVENT=$(echo $file | awk '{print $1}')
        INO_FILE=$(echo $file | awk '{print $2}')
        echo "-------------------------------$(date)------------------------------------"
        echo $file

        if [[ $INO_EVENT =~ 'CREATE' ]] || [[ $INO_EVENT =~ 'MODIFY' ]] || [[ $INO_EVENT =~ 'CLOSE_WRITE' ]] || [[ $INO_EVENT =~ 'MOVED_TO' ]]
        then
                echo 'CREATE or MODIFY or CLOSE_WRITE or MOVED_TO'
                rsync -avzcR --password-file=${rsync_passwd_file} $(dirname ${INO_FILE}) ${user}@${ip1}::${des} &&
                rsync -avzcR --password-file=${rsync_passwd_file} $(dirname ${INO_FILE}) ${user}@${ip2}::${des}
        fi
        if [[ $INO_EVENT =~ 'DELETE' ]] || [[ $INO_EVENT =~ 'MOVED_FROM' ]]
        then
                echo 'DELETE or MOVED_FROM'
                rsync -avzR --delete --password-file=${rsync_passwd_file} $(dirname ${INO_FILE}) ${user}@${ip1}::${des} &&
                rsync -avzR --delete --password-file=${rsync_passwd_file} $(dirname ${INO_FILE}) ${user}@${ip2}::${des}
        fi
        if [[ $INO_EVENT =~ 'ATTRIB' ]]
        then
                echo 'ATTRIB'
                if [ ! -d "$INO_FILE" ]
                then
                        rsync -avzcR --password-file=${rsync_passwd_file} $(dirname ${INO_FILE}) ${user}@${ip1}::${des} &&
                        rsync -avzcR --password-file=${rsync_passwd_file} $(dirname ${INO_FILE}) ${user}@${ip2}::${des}
                fi
        fi

done
```

###### 其中 host 是 client 的 ip，src 是 server 端要实时监控的目录，des 是认证的模块名，需要与 client 一致，user 是建立密码文件里的认证用户。

然后给这个脚本赋予权限

```shell
chmod 755 /root/data/backuptest/inotifyrsync.sh
```



后台运行这个脚本

```shell
/root/data/backuptest/inotifyrsync.sh &
```



有需要可以将脚本加入系统自启动文件中

```shell
echo "/root/data/backuptest/inotifyrsync.sh &" >> /etc/rc.local
```



##### 五、总结

[原文](https://blog.csdn.net/xch_yang/article/details/104941081 '原文')中rsync 脚本 每次都是全量的同步(这就坑爹了)，而且 file列表是循环形式触发rsync ，等于有10个文件发生更改，就触发10次rsync全量同步(简直就是噩梦)，那还不如直接写个死循环的rsync全量同步得了。

\#有很多人会说 日志输出那里明明只有差异文件的同步记录。其实这是rsync的功能，他本来就只会输出有差异需要同步的文件信息。不信你直接拿这句rsync来跑试试。

\#这种在需要同步的源目录文件量很大的情况下，简直是不堪重负。不仅耗CPU还耗时，根本不可以做到实时同步。

要做到实时，就必须要减少rsync对目录的递归扫描判断，尽可能的做到只同步inotify监控到已发生更改的文件。结合rsync的特性，所以这里要分开判断来实现一个目录的增删改查对应的操作。



##### 每两小时做1次全量同步

因为inotify只在启动时会监控目录，他没有启动期间的文件发生更改，他是不知道的，所以这里每2个小时做1次全量同步，防止各种意外遗漏，保证目录一致。

 

```shell
crontab -e

\* */2 * * * rsync -avz --password-file=/etc/rsync-client.pass /data/ root@192.168.0.18::data && rsync -avz --password-file=/etc/rsync-client.pass /data/ root@192.168.0.19::data
```



##### 优化 **Inotify**

\# 在/proc/sys/fs/inotify目录下有三个文件，对inotify机制有一定的限制

```shell
 ll /proc/sys/fs/inotify/

-rw-r--r--1 root root 09月923:36 max_queued_events

-rw-r--r--1 root root 09月923:36 max_user_instances

-rw-r--r--1 root root 09月923:36 max_user_watches
```

```
max_user_watches #设置inotifywait或inotifywatch命令可以监视的文件数量(单进程)

max_user_instances #设置每个用户可以运行的inotifywait或inotifywatch命令的进程数

max_queued_events #设置inotify实例事件(event)队列可容纳的事件数量
```



```shell
[root@web ~]# echo 50000000>/proc/sys/fs/inotify/max_user_watches -- 把他加入/etc/rc.local就可以实现每次重启都生效

[root@web ~]# echo 50000000>/proc/sys/fs/inotify/max_queued_events
```

