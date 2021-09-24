xl2tpd的2022.09.12到期


yum install https://dl.fedoraproject.org/pub/epel/8/Everything/x86_64/Packages/e/epel-release-8-13.el8.noarch.rpm -y

yum install -y net-tools bridge-utils iptables

yum install libreswan xl2tpd  NetworkManager-l2tp  NetworkManager-ppp -y

yum remove firewalld
环境
Java代码 
ios ip:  
223.103.3.xxx  
  
tx:  
docker0 172.17.0.1 没用到  
eth0 10.0.4.3 通外网  netmask  255.255.252.0  
  
will虚拟出：  
ppp0 连 aws  
ppp1 连 ios  
  
aws：  
18.183.169.35  
eth0 通外网172.31.38.118  
  
will虚拟出：  
ppp0 连 tx  


网络通是关键
要达到
ios-->tx--->aws-->自由世界

需要设置的iptables和iproute 为

Java代码 
tx：  
iptables -t nat -A POSTROUTING -s 192.168.2.0/24  -o ppp0  -j MASQUERADE  
iptables -t nat -A POSTROUTING -s 10.0.4.0/22  -o eth0  -j MASQUERADE  
route add -net 18.183.169.0 netmask 255.255.255.0 dev eth0  
aws：  
iptables -t nat -A POSTROUTING -s 10.0.4.0/22  -o eth0  -j MASQUERADE  
route add -net  101.35.118.0 netmask 255.255.255.0 dev eth0  


内核
Java代码 
modprobe l2tp_ppp  
modprobe ppp-compress-18  

如果这个报错，重启服务器
另外安装完xl2tpd有可能也要重启服务器
配合配置
/etc/xl2tpd/xl2tpd.conf
Java代码 
[global]  
ipsec saref = yes  
force userspace = yes  
  
/etc/ipsec.conf  
config setup  
    plutodebug=none  
    protostack=netkey  
    dumpdir=/var/run/pluto/  

网络： 关注路由和 iptables
使用NetworkManager 禁用firewalld

ios-tx-aws

通用的设置是把 mtu 都设置成1400 ，并设置promisc混杂模式 ，所有机器的所有网卡 都要设置，
Java代码 
这个tx和aws相同  
  
ifconfig eth0 mtu 1400  
ifconfig docker0 mtu 1400  
ifconfig lo mtu 1400  
  
ifconfig eth0 promisc  
ifconfig docker0 promisc  

有了虚拟网卡后
ifconfig ppp0 promisc
ifconfig ppp1 promisc

/etc/xl2tpd/xl2tpd.conf中的 /etc/ppp/options.xl2tpd
也要设置
mtu 1400
mru 1400
这样会使新建的ppp0和ppp1也是1400

还要设置系统的转发

ipv4.sh
Java代码 
#!/bin/sh  
iptables -t nat -A POSTROUTING -s 10.0.4.0/22  -o eth0  -j MASQUERADE  
#iptables -t nat -A POSTROUTING -s 192.168.1.0/24  -o eth0  -j MASQUERADE  
#route add -net  101.35.118.0 netmask 255.255.255.0 dev eth0  
ifconfig eth0 mtu 1400  
ifconfig lo mtu 1400  
ifconfig eth0 promisc  
ifconfig lo promisc  
#ifconfig ppp0 promisc  
for each in /proc/sys/net/ipv4/conf/*  
do  
    echo 0 > $each/accept_redirects  
    echo 0 > $each/send_redirects  
done  
  
#./add_ext.sh 101.35.118.0  
ip route  


vim /etc/sysctl.conf
Java代码 
net.ipv4.ip_forward = 1  
net.ipv4.conf.all.accept_redirects = 0  
net.ipv4.conf.all.rp_filter = 0  
net.ipv4.conf.all.send_redirects = 0  
net.ipv4.conf.default.accept_redirects = 0  
net.ipv4.conf.default.rp_filter = 0  
net.ipv4.conf.default.send_redirects = 0  
net.ipv4.conf.eth0.accept_redirects = 0  
net.ipv4.conf.eth0.rp_filter = 0  
net.ipv4.conf.eth0.send_redirects = 0  
net.ipv4.conf.lo.accept_redirects = 0  
net.ipv4.conf.lo.rp_filter = 0  
net.ipv4.conf.lo.send_redirects = 0  
net.ipv4.conf.docker0.rp_filter = 0  
net.ipv4.conf.ip_vti0.rp_filter = 0  

sysctl -p /etc/sysctl.conf




然后为了网络通，需要设置 ip route 和 iptables

开启NetworkManager，关闭firewalld
Java代码 
yum install NetworkManager-l2tp NetworkManager-ppp -y  
  
systemctl restart NetworkManager  
systemctl enable NetworkManager  
systemctl disable firewalld  

启动ipsec和xl2tpd后在外网测试端口是否通了，
Java代码 
nc -vuz 18.183.169.35 4500  
nc -vuz 18.183.169.35 1701  
nc -vuz 18.183.169.35 5000  

为了让ios和tx通
add_ext.sh
Java代码 
#!/bin/sh  
 [ ! $1 ] && echo "a is null,please use ./add_e.sh ip..." && exit  
  
echo "route will add $1,please check with  ip route "  
#route add -net  61.148.245.0 netmask 255.255.255.0 dev eth0  
route add -net  $1 netmask 255.255.255.0 dev eth0  

./add_ext.sh ios的ip
例如：
./add_ext.sh 223.104.3.0
或者直接使用：
Java代码 
route add -net  223.104.3.0 netmask 255.255.255.0 dev eth0  

为了让 checkppp0.sh中的echo 'c testvpn' > /var/run/xl2tpd/l2tp-control 生效
在tx上
./add_ext.sh 3.112.1.0

把aws的ip3.112.1.0 通过eth0访问，这样，default route异常的情况下也可以建立ppp0-ppp0连通



在tx中，
Java代码 
iptables -t nat -A POSTROUTING -s 192.168.2.0/24  -o ppp0  -j MASQUERADE  

因为/etc/xl2tpd/xl2tpd.conf设置的，给客户端建立的都是192.168.2段
而ppp0是由 tx到 aws的通路， 这样的iptables设置，是为了， 客户端 的网络直接连到 ppp0上，再连到 aws，实现ios到aws网络的联通，这步骤也要设置ppp0的promisc和mtu 1400

ppp0是通过
命令
echo 'c testvpn' > /var/run/xl2tpd/l2tp-control
和配置
/etc/xl2tpd/xl2tpd.conf中配置的
Java代码 
[lac testvpn]  
lns =  18.183.169.35  
;pppoptfile = /etc/ppp/peers/testvpn.l2tpd  
pppoptfile = /etc/ppp/peers/aws.l2tpd  
ppp debug = yes  

其中
/etc/ppp/peers/aws.l2tpd
配置的aws的用户和密码
Java代码 
remotename testvpn  
user "root"  
password "Haohao123!"  
unit 0  
nodeflate  
nobsdcomp  
noauth  
persist  
nopcomp  
noaccomp  
maxfail 5  
debug  



在tx上
Java代码 
iptables -t nat -A POSTROUTING -s 10.0.4.0/22  -o eth0  -j MASQUERADE  

这个是否需要，忘记了



然后设置
aws
Java代码 
iptables -t nat -A POSTROUTING -s 10.0.4.0/22  -o eth0  -j MASQUERADE  

10.0.4.是tx的网段，这样设置是让，tx连过来的网络多都可以通过eth0 出去，eth0是aws连外网通往自由世界的网络呀


所以原理就是
aws 通过xl2tpd建立了aws的ppp0 ，并且eth0能访问自由世界

tx 通过xl2tdp 的 lac testvpn 客户端，建立了 tx的 ppp0 ，链接到了 aws的ppp0上，在通过aws上的iptables，把网络发到aws的eth0上去
当ios 发起网络链接时，ios获取ip 192.168.2.x ，这时 tx 通过xl2tpd 的 lns default 服务端，建立了 tx的ppp1，  iptables又把ios的流量全通过tx的ppp0 转发到aws的ppp0


所以网络两路就是

ios ----> ppp1 ----> iptables ----> tx的 ppp0 ----> aws的ppp0 ----> iptables ---> aws的eth0 ---> 自由的飞翔

上面所有配置

tx上：
ipsec服务需要的两个配置
/etc/ipsec.conf
和
/etc/ipsec.d/l2tp-ipsec.conf
xl2tpd需要的3个配置：
/etc/xl2tpd/xl2tpd.conf 总体配置
/etc/ppp/peers/aws.l2tpd 客户端密码
/etc/ppp/options.xl2tpd 建立ppp1的配置




cat /etc/ipsec.conf
config setup下面新增
Java代码 
protostack=netkey  
dumpdir=/var/run/pluto/  

这个配置是要配合内核使用的
modprobe l2tp_ppp
modprobe ppp-compress-18


新增配置文件/etc/ipsec.d/l2tp-ipsec.conf
Java代码 
conn L2TP-PSK-NAT  
    rightsubnet=0.0.0.0/0  
    dpddelay=10  
    dpdtimeout=20  
    dpdaction=clear  
    encapsulation=yes  
    also=L2TP-PSK-noNAT  
conn L2TP-PSK-noNAT  
    authby=secret  
    pfs=no  
    auto=add  
    keyingtries=3  
    rekey=no  
    ikelifetime=8h  
    keylife=1h  
    type=transport  
    left=10.0.4.3  
    leftprotoport=17/1701  
    right=%any  
    rightprotoport=17/%any  

    其中left是 tx的内网能访问外网的ip地址，比如eth0的

/etc/xl2tpd/xl2tpd.conf
Java代码 
[lac testvpn]  
lns =  18.183.169.35  
pppoptfile = /etc/ppp/peers/aws.l2tpd  
ppp debug = yes  
[global]  
ipsec saref = yes  
force userspace = yes  
  
[lns default]  
ip range = 192.168.2.128-192.168.2.254  
local ip = 192.168.2.99  
require chap = yes  
refuse pap = yes  
require authentication = yes  
name = LinuxVPNserver  
ppp debug = yes  
pppoptfile = /etc/ppp/options.xl2tpd  
length bit = yes  

其中[lac testvpn]是客户端配置 ，使用 echo 'c testvpn' > /var/run/xl2tpd/l2tp-control  的时候触发，会链接 aws，在tx上建立ppp0 和在aws上建立ppp0的点对点链接

[global]需要开启内核的配置
配合
modprobe l2tp_ppp
modprobe ppp-compress-18
使用

[lns default] 的ip 改成192.168.2.99 段， 和 aws的xl2tpd.conf的配置区分开，不同网段就可以了，aws可以保留192.168.1.99的网段


/etc/ppp/options.xl2tpd
Java代码 
ipcp-accept-local  
ipcp-accept-remote  
ms-dns  8.8.8.8  
ms-dns  1.1.1.1  
noccp  
auth  
idle 1800  
mtu 1400  
mru 1400  
nodefaultroute  
debug  
proxyarp  
connect-delay 5000  
refuse-pap  
refuse-mschap  
require-mschap-v2  
persist  
logfile /var/log/xl2tpd.log  


在tx上的脚本：
checkppp0.sh
Java代码 
#!/bin/sh  
ppp0=`ifconfig |grep ppp0`  
if [ ! -n "$ppp0" ] ;then  
    a="will start vpn"  
    echo 'c testvpn' > /var/run/xl2tpd/l2tp-control  
    sleep 5  
    route del default  
    ip link set ppp0 up  
    /usr/sbin/route add default dev ppp0  
    touch /opt/c  
else  
    ip link set ppp0 up  
    /usr/sbin/route add default dev ppp0  
    a="noting to do"  
fi  
/usr/sbin/ifconfig ppp0  
  
echo $a  
#route add -net  117.136.38.0 netmask 255.255.255.0 dev eth0  
#ip route get 172.17.0.13  


这个脚本是建立整体链接用的
echo 'c testvpn' > /var/run/xl2tpd/l2tp-control 触发建立tx到aws的ppp0 ---ppp0的通路
tx默认的网关是eth0
route del default删掉网关
设置ppp0为网关：
Java代码 
/usr/sbin/route add default dev ppp0  

这样从ios建立的链接ppp1的网络也会从ppp0 出去，ppp0再去找aws的ppp0，从而实现 ios的网络通过 tx 到了aws

add_ext.sh
Java代码 
#!/bin/sh  
 [ ! $1 ] && echo "a is null,please use ./add_e.sh ip..." && exit  
  
echo "route will add $1,please check with  ip route "  
#route add -net  6223.103.3.0 netmask 255.255.255.0 dev eth0  
route add -net  $1 netmask 255.255.255.0 dev eth  

这个是设置ios客户端的网路

以为上一个脚本checkppp0.sh执行后，网关变成ppp0，外面的网络都进不来了

通过这个 让ios的网络 223.103.3.176 能通过路由 直接从eth0进入 tx网络，



配置aws的配置文件
ipsec的
/etc/ipsec.conf
/etc/ipsec.d/l2tp-ipsec.conf
和xl2tpd的
/etc/xl2tpd/xl2tpd.conf
/etc/ppp/options.xl2tpd


ipsec的
/etc/ipsec.conf
Java代码 
config setup  
    plutodebug=none  
    protostack=netkey  
    dumpdir=/var/run/pluto/  

同样需要这样的配置

/etc/ipsec.d/l2tp-ipsec.conf
Java代码 
conn L2TP-PSK-NAT  
    rightsubnet=0.0.0.0/0  
    dpddelay=10  
    dpdtimeout=20  
    dpdaction=clear  
    also=L2TP-PSK-noNAT  
conn L2TP-PSK-noNAT  
    authby=secret  
    pfs=no  
    auto=add  
    keyingtries=3  
    rekey=no  
    ikelifetime=8h  
    keylife=1h  
    type=transport  
    left=172.31.37.174  
    leftprotoport=17/1701  
    right=%any  
    rightprotoport=17/%any  

其中left是可以方位外网的内网地址，比如eth0的地址

/etc/xl2tpd/xl2tpd.conf
Java代码 
[global]  
ipsec saref = yes  
force userspace = yes  
  
[lns default]  
ip range = 192.168.1.128-192.168.1.254  
local ip = 192.168.1.99  
require chap = yes  
refuse pap = yes  
require authentication = yes  
name = LinuxVPNserver  
ppp debug = yes  
pppoptfile = /etc/ppp/options.xl2tpd  
length bit = yes  

其中
[global]中的两条是新加的

/etc/ppp/options.xl2tpd
Java代码 
ipcp-accept-local  
ipcp-accept-remote  
ms-dns  8.8.8.8  
ms-dns  1.1.1.1  
# ms-dns  192.168.1.1  
# ms-dns  192.168.1.3  
# ms-wins 192.168.1.2  
# ms-wins 192.168.1.4  
noccp  
auth  
#obsolete: crtscts  
idle 1800  
mtu 1400  
mru 1400  
nodefaultroute  
debug  
#obsolete: lock  
proxyarp  
connect-delay 5000  
refuse-pap  
refuse-mschap  
require-mschap-v2  
persist  
logfile /var/log/xl2tpd.log  



两台机器都要配置
Java代码 
/etc/ipsec.d/default.secrets  
:   PSK "Haohao123!"  
  
cat /etc/ppp/chap-secrets    
# Secrets for authentication using CHAP    
# client    server  secret          IP addresses    
root    *   Haohao123!  *  



配置完之后，顺序是
两台机器设置
yum  install NetworkManager-l2tp NetworkManager-ppp -y

systemctl restart NetworkManager
检查ip route
检查 iptables -t nat -L
检查ipv4.sh
Java代码 
for each in /proc/sys/net/ipv4/conf/*  
do  
    echo 0 > $each/accept_redirects  
    echo 0 > $each/send_redirects  
done  

检查内核配置
modinfo ppp-compress-18
modinfo  l2tp_ppp

systemctl start ipsec
systemctl restart xl2tpd

然后在tx上
使用checkppp0.sh

检查 ppp0 的mtu和promisc设置

ios链接 后 ，检查tx上的ppp1 的mtu和promisc的设置



