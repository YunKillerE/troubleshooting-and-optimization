> 主要还是针对部署hadoop集群之前所做的一些优化与初始化

# 1，修改主机名为 FQDN

* 修改/etc/sysconfig/network 文件：

``
NETWORKING=yes
HOSTNAME=hadoop.one
``

* 修改 hosts：

``
[root@kali ~]# cat /etc/hosts
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
::1 localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.3.47 hadoop.one hadoop
192.168.3.53 dn1.one dn1
``

如果有多台则这里加入多行，比如 datanote 地

> 注意：这里127.0.0.1 localhost这一行不能删除掉，否则后面安装cdh时会无法下发parcel包

# 2，配置 SSH 免密码登录

管理节点生产密钥：

``
ssh-keygen -t rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
``

将自动应答文件传输到其他机器：

``
scp ~/.ssh/authorized_keys root@datanode1:~/.ssh/
``

# 3，关闭防火墙与selinux（所有节点）

iptables：

``
service iptables stop (临时关闭)
chkconfig iptables off (重启后生效)
``

selinux：

``
setenforce 0 (临时生效)
修改/etc/selinux/config 下的 SELINUX=disabled （重启后生效）
``

# 4，配置时间同步 ntp

> 两种情况，一种是可以访问外网的情况的话直接同步外网的时间服务器，另一种是无法访问外网的情况将管理节点作为时间服务器供其他节点题哦内部

* 第一种情况

> 比较简单，甚至直接用默认的就行了，如果公司有ntp服务器，加入一行

``
server 3.centos.pool.ntp.org
``

* 第二种情况

> 首先管理节点配置ntp服务器

``
[root@zjdw-pre0064 ~]# cat /etc/ntp.conf |grep ^[^#]
driftfile /var/lib/ntp/drift
restrict default kod nomodify notrap nopeer noquery
restrict -6 default kod nomodify notrap nopeer noquery
restrict 127.0.0.1
restrict -6 ::1
restrict 192.168.3.0 mask 255.255.255.0 nomodify notrap #允许哪些服务器来同步时间
server 127.127.1.0  #注释掉其他几行server，加入这行
fudge 127.127.1.0 stratum 10
includefile /etc/ntp/crypto/pw
keys /etc/ntp/keys
``

然后启动服务

``
service ntpd start
chkconfig ntpd on
``

配置其他服务器同步,在ntp配置文件加入一行就行，然后启动ntp服务器

``
server zjdw-pre0064
``

# 5，安装jdk

建议安装jdk8，spark2.x和hadoop3.x都建议jdk8

系统版本可以采用centos6来进行安装部署，centos7 也可以，但注意 7.0 不行，7.1 和 7.2 可以

# 6，系统open file的修改（避免Too many open files错误）

* 1，/etc/security/limits.conf  （当前shell以及由它启动的进程的资源限制）

``
* soft nofile 65535 
* hard nofile 65535
``

* 2，fs.file-max （系统所有进程一共可以打开的文件数量）

在/etc/sysctl.conf中加入

``
fs.file-max = 13122794
``

# 7，安装一些依赖包

``
yum install  -y libxslt readline readline-devel readline-static openssl openssl-devel openssl-static sqlitedevel bzip2-devel bzip2-libs ntp ntpdate lzo lzo-devel gzip zlib zlib-devel 
``

# 8，创建用户

> 根据公司需求去创建用户，一般是一个部分一个用户组，比如开发组、运维组、数据分析组、算法组等等，后续可以根据部分去分配权限

``
groupadd cloudera-dev
useradd -g cloudera-dev cloudera-dev
``


> 下面开始一些Linux系统以及内核参数的调优

# 9，SWAP内存的设置

``
gm.sh allhost "echo 1 > /proc/sys/vm/swappiness"
gm.sh allhost "echo 'vm.swappiness = 1' >> /etc/sysctl.conf"
``

* 为什么要尽可能避免使用swap? # cat /proc/sys/vm/swappiness,值默认值是60

* swappiness=0的时候表示最大限度使用物理内存，然后才是 swap空间

* swappiness＝100的时候表示积极的使用swap分区，并且把内存上的数据及时的搬运到swap空间里面

* 现在服务器的内存动不动就是上百G，或者云主机架构，机器io太慢会极大的影响机器性能，所以我们可以把这个参数值设置的低一些，让操作系统尽可能的使用物理内存，降低系统对swap的使用，从而提高系统的性能

* 在大内存服务器中我们需要设置这个值为0，尤其是在MySQL服务器上

# 10，关闭透明大页面压缩，这可能会导致重大性能问题

gm.sh allhost "echo never > /sys/kernel/mm/transparent_hugepage/defrag"
gm.sh allhost "echo never > /sys/kernel/mm/transparent_hugepage/enabled"

然后将同一命令添加到 /etc/rc.local 等初始化脚本中，以便在系统重启时予以设置

gm.sh allhost "echo 'never > /sys/kernel/mm/transparent_hugepage/enabled' >> /etc/rc.local"
gm.sh allhost "echo 'never > /sys/kernel/mm/transparent_hugepage/defrag' >> /etc/rc.local"

# 11，关闭磁盘的File Access Time

/etc/fstab

``
/dev/sdb1               /data1                  ext4    defaults,noatime        0 0
``

mount -o remount /data1

* 内核源代码 linux-2.6.33/fs/inode.c 文件里有一个 touch_atime 函数，可以看出如果 inode 的标记位是 NOATIME 的话就直接返回了，根本就走不到 NODIRATIME 那里去，所以只设置 noatime 就可以了，不必再设置 nodiratime

```
void touch_atime(struct vfsmount *mnt, struct dentry *dentry) 
{ 
        struct inode *inode = dentry->d_inode; 
        struct timespec now; 
 
        if (inode->i_flags & S_NOATIME) 
                return; 
        if (IS_NOATIME(inode)) 
                return; 
        if ((inode->i_sb->s_flags & MS_NODIRATIME) && S_ISDIR(inode->i_mode)) 
                return; 
 
        if (mnt->mnt_flags & MNT_NOATIME) 
                return; 
        if ((mnt->mnt_flags & MNT_NODIRATIME) && S_ISDIR(inode->i_mode)) 
                return; 

}
```

# 12，



























