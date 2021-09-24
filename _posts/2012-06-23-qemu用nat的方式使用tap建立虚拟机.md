qemu用nat的方式使用tap建立虚拟机
qemu用nat的方式使用tap建立虚拟机 恢复 删除
2016-06-23
博客分类： tap网络qemu
虚拟机ubuntulinuxtap 
普通桥接参考 http://haoningabc.iteye.com/blog/2306736

需求:用nat的方式，利用qemu建立一个虚拟机，使虚拟机可以访问外网
目前主机的ip为192.168.139.85
想设置虚拟机的ip段为192.168.122.0段


需要技术：qemu,tap,brctl
环境:Ubuntu 14.04.4 LTS

一.建立一个120M的简版linux的镜像hda.img,要求支持ip,ifconfig,dhclient等网络命令
建立方法,这个debootstrap命令国内可能不通，去aws上操作吧,如果已经有了可用的img文件，直接看第二步
Java代码 
cd /opt/  
debootstrap        --variant=minbase         --include=dhcp-client,ssh,vim,make,psmisc,mini-httpd,net-tools,iproute,iputils-ping,procps,telnet,iptables,wget,tcpdump,curl,gdb,binutils,gcc,libc6-dev,lsof,strace         --exclude=locales,aptitude,gnupg,cron,udev,tasksel,rsyslog,groff-base,manpages,gpgv,man-db,apt,debian-archive-keyring,sysv-rc,sysvinit,insserv,python2.6         --arch i386         etch etch         'http://archive.debian.org/debian'  

注意需要dhcp-client
下载ubuntu的etch 版本
使用https://github.com/killinux/jslinux_reversed
Java代码 
cd /var/www/html  
git clone https://github.com/killinux/jslinux_reversed  
mv /opt/etch /var/www/html/jslinux_reversed/contrib/squeeze  
./createimage.sh  

生成hda.img
这个是给jslinux用的，需要修改一下/sbin/init文件
mount -o loop hda.img hda
vim hda/sbin/init
Java代码 
#!/bin/sh  
show_boot_time 2>/dev/null  
echo "JSLinux started, initializing..."  
export PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin  
export HOME=/root  
export TERM=vt100  
mount -n -t proc /proc /proc  
mount -n -t sysfs /sys /sys  
mount -n -t devpts devpts /dev/pts  
mount -n -t tmpfs /tmp /tmp  
mkdir -p "/tmp/root"  
ip link set up dev lo  
main() {  
    echo >/dev/clipboard  
    while :; do  
        setsid sh -c "exec bash 0<>/dev/ttyS0 1>&0 2>&0"  
    done  
}  
#. /dev/clipboard  
main "$@"  

注释掉#. /dev/clipboard

二.使用qemu建立虚拟机
Java代码 
qemu-system-i386 -kernel /root/jslinux/obj/linux-x86-basic/arch/i386/boot/bzImage -drive file=hda.img,if=ide,cache=none -append "console=ttyS0 root=/dev/sda rw rdinit=/sbin/init notsc=1"  -nographic -boot order=dc,menu=on -net nic,vlan=0,macaddr=52:54:00:12:34:22,model=e1000,addr=08 -net tap,ifname=tap1,script=no,downscript=no  

bzImage 内核需要自己编译，参考http://haoningabc.iteye.com/blog/2237569
这里注意使用-net tap,ifname=tap1
建立vm后，
主机ip link show
发现多了个tap1设备
把tap1绑定到桥上，这个桥可以自己随便建
Java代码 
ip link set tap1 up  
brctl addbr virbr0  
brctl addif virbr0 tap1  
#设置nat的iptables  
iptables -t nat -A POSTROUTING -s "192.168.122.0/255.255.255.0" ! -d "192.168.122.0/255.255.255.0" -j MASQUERADE  
#设置转发  
echo 1 >/proc/sys/net/ipv4/ip_forward  
ifconfig eth0 promisc  


三.主机上启动dnsmasq服务，提供dhcp的server功能,
注意参数指向刚建的virbr0桥上
Java代码 
dnsmasq --strict-order --except-interface=lo --interface=virbr0 --listen-address=192.168.122.1 --bind-interfaces  --dhcp-range=192.168.122.2,192.168.122.254 --conf-file=""  --pid-file=/var/run/qemu-dhcp-virbr0.pid  --dhcp-leasefile=/var/run/qemu-dhcp-virbr0.leases --dhcp-no-override   

关键点是--interface=virbr0

四.在虚拟机中，dhcp配置网络
Java代码 
ip link set eth0 up  
dhclient  
ping www.baidu.com  
ifconfig eth0  
#成功设置ip 192.168.122.37  
cat /etc/resolv.conf  
#发现dns也自动建立了  



