# Server端配置 #

##  第一步，cobbler环境准备
*   **cobbler服务器和客户机IP**
Cobbler服务器系统：Redhat 6.4
IP地址：172.16.10.1

    需要安装不熟的Linux系统的客户机IP地址段：172.16.10.10-172.16.10.100
子网掩码：255.255.255.0
所有服务器均支持PXE网络启动
实现目的：通过配置Cobbler服务器，全自动批量安装部署Linux系统
*   **关闭防火墙与Selinux**
[root@localhost /]# service iptables stop && chkconfig iptables off
[root@localhost /]# setenforce 0 && sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
*   **配置yum源及epel源**
[root@localhost /]# wget https://mirrors.ustc.edu.cn/fedora/epel/6/x86_64/epel-release-6-8.noarch.rpm  
[root@localhost /]# rpm -ivh epel-release-6-8.noarch.rpm
[root@localhost /]# cat << EOF  >> /etc/yum.repos.d/rhel-source.repo
> [centos6.8]
> name=centos6.8
> basurl=http://mirrors.163.com/centos6.8/os/x86_64/
> gpgchenk=0
> enabled=1
>EOF
*   **同步系统时间**
[root@localhost /]# tzclect                   #设置时区
[root@localhost /]# ntpdate  x.x.x.x         #同步时钟   x.x.x.x代表时间服务器
地址
[root@localhost /]# hwclock -w              #写硬件时钟
*   **安装cobbler**
    [root@localhost /]# yum install dhcp tftp-server httpd syslinux cobbler
    cobbler-web pykickstart debmirroy -y

## 第二步，配置cobbler
*1、 设置httpd服务*
*2、 配置tftp与rsync*

    [root@localhost /]# chkconfig tftp on
    [root@localhost /]# chkconfig rsync on
    [root@localhost /]# service xinetd restart

*3、 配置cobbler主配置文件*
    在启动Cobbler服务之前，你需要修改一些配置文件。在修改每一个文件之前最好先备份下当前的文件。
    Cobblerd的配置文件为/etc/cobbler/settings ,这个文件是YAML信息的格式文件。
    根据需要修改 cobbler主配置文件: /etc/cobbler/settings
    ①Server 和 Next_Server
    server 选项用于为cobbler 服务器指定ip地址，请不要使用0.0.0.0，设置一个你希
    望和cobbler服务器通过http和tftp等协议链接的IP。
    next_server选项是DHCP/PXE网络引导文件被下载的TFTP服务器的IP，在本例中它将和
    server设置为同一个IP

    [root@localhost /]# sed -i 's/server: xxx.xxx.xxx.xxx/server:xxx.xxx.xxx.xxx/' /etc/cobbler/settings
②防止误重装系统
pxe安装只允许一次，防止误操作 ( 在正式环境有用。实际测试来看，这个功能可以屏蔽掉 )

    [root@localhost /]# sed -i 's/pxe_just_once: 0/pxe_just_once: 1/g' /etc/cobbler/settings

③Cobbler管理rsync
    默认为0,不对rsync进行管理,可以修改为1 进行管理

    [root@localhost /]# sed -i 's/manage_rsync: 0/manage_rsync: 1/g' /etc/cobbler/settings
⑤. Cobbler管理dhcp
    为了pxe的启动,需要一个DHCP服务器地址，并直接引导系统，它可以在网络中下载引
    导文件到TFTP的服务器，cobbler可以通过manage_dhcp的设置来进行管理
    配置dhcp服务,首先修改cobbler配置，让cobbler来管理dhcp服务，在做自定义配置时，需要修改
    dhcp相关配置，以配合PXE启动用，编辑文件/etc/cobbler/settings
    manage_dhcp: 1 (注：默认为0 ,表示不进行管理dhcp服务,可以修改为1,对其进行管 理。此为使cobbler管理dhcp也就是后面用于同步更新配置信息[cobbler    sync])
    也可用命令修改：

    [root@localhost /]# sed -i 's/manage_dhcp: 0/manage_dhcp: 1/g' /etc/cobbler/settings
    [root@localhost /]# grep '^manage_dhcp' /etc/cobbler/settings
*4、修改cobbler管理DHCP的模板*
/etc/cobbler/dhcp.template,此文件是cobbler管理dhcp的模板，确保DHCP分配的地址和
cobbler在同一网段，对于此文件本例中只需修改如下部分
    #需要修改192.168.122.0为自己的网段
>    subnet 192.168.122.0 netmask 255.255.255.0 {
>            option routers             192.168.122.10;
>            option domain-name-servers 192.168.122.10;
>            option subnet-mask         255.255.255.0;
>            range dynamic-bootp        192.168.122.100 192.168.122.200;
>            default-lease-time         21600;
>            max-lease-time             43200;
>            next-server                $next_server;}

如果是多网卡需要指定DHCP服务的网络接口

    [root@localhost /]# vim /etc/sysconfig/dhcpd
修改内容如下：
    #Command line options here
__DHCPDARGS=eth0__

#测试dhcp服务器配置是否正确，需要在执行cobbler sync之后测试。因为没有sync
之前/etc/dhcp/dhcpd.conf还没有被同步。
#cobbler sync同步配置前需启动httpd服务，否则会报错。

*5、检查cobbler配置根据提示解决报错*

    [root@localhost /]# cobbler check
>1: some network boot-loaders are missing from /var/lib/cobbler/loaders, you may
>run 'cobbler get-loaders' to download them, or, if you only want to handle
>x86/x86_64 netbooting, you may ensure that you have installed a *recent*
>version of the syslinux package installed and can ignore this message entirely.
>Files in this directory, should you want to support all architectures, should
>include pxelinux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders'
>command is the easiest way to resolve these requirements.
>2 : file /etc/xinetd.d/rsync does not exist
>3 : since iptables may be running, ensure 69, 80/443, and 25151 are unblocked
>4 : comment out 'dists' on /etc/debmirror.conf for proper debian support
>5 : comment out 'arches' on /etc/debmirror.conf for proper debian support
>6 : The default password used by the sample templates for newly installed
>machines (default_password_crypted in /etc/cobbler/settings) is still set to
>'cobbler' and should be changed, try: "openssl passwd -1 -salt
>'random-phrase-here' 'your-password-here'" to generate new one
>7 : fencing tools were not found, and are required to use the (optional) power
>management features. install cman or fence-agents to use them

   #报错解释
>1:运行cobbler get-loaders 下载缺少的启动文件，需要联网
>2:安装rsync    装好了依然显示没有装，可能是bug
>3:关闭防火墙
>4:注释掉/etc/debmirror.conf 中@dists开头的行
>5:注释掉/etc/debmirror.conf 中@arches开头的行
>6:修改cobbler配置文件中的默认密码，可以使用openssl passwd -1 -salt ‘任意字符’ ‘你的密码’ 生成密码
>7:电源管理相关错误，安装fence-agents可解决

   所有问题修改以后需要重启cobbler服务，然后重新执行cobbler sync，之后再次运行
cobbler check查看。如果紧剩rsync一项则不用理会。

    [root@localhost /]# service cobblerd restart
    [root@localhost /]# cobbler sync
    [root@localhost /]#  cobbler check
#检查开机启动项

    [root@localhost /]#  chkconfig cobblerd on
    [root@localhost /]#  chkconfig dhcpd on
    [root@localhost /]#  httpd on
    [root@localhost /]#  xinetd on
    [root@localhost /]#  tftp on
    [root@localhost /]#  rsync on
## 第三步，导入镜像，配置ks文件

1、导入要安装的镜像文件
①、挂载镜像

    [root@localhost /]# mount /dev/sr0   /mnt
②、使用cobbler import命令导入镜像
主要有以下几个参数--path 指定导入镜像的路径，--name 指定导入镜像的名称，--arch
指定导入镜像的架构(32位还是64位)。还有需要说明的是这里导入的时间相对较长大概在
5-10分钟左右，请大家耐心等待。

    [root@localhost /]# cobbler import --path=/mnt --name=rhel6.5 --arch=x86_64
*注意1： 这个安装源的唯一标示就是根据--name和--arch这两个参数来定义
本例导入成功后，安装源的唯一标示就是：rhel6.5-x86_64，如果重复，系统会提示导入
失败，其它命令可通过cobbler –help来进行查看。如果需要更多的参数定制，也可以查看
官方文档: man cobbler ,然后查找 import 的配置，可以使用另外一个命令: cobbler distro。
从上面显示信息所知：cobbler会将镜像中的所有安装文件拷贝到本地一份，放在/var/www/cobbler/ks_mirrors
下的rhel6.5-x86_64目录下。同时会创建一个名字为rhel6.5-x86_64的一个(distro)发布
版本，以及一个名字为rhel6.5-x86_64的profile文件。*
*注意2：/var/www/cobbler 目录必须具有足够容纳 Linux安装文件的空间。如果空间不够
，可以对/var/www/cobbler目录进行移动，建软链接来修改文件存储位置。
例如：*

    [root@localhost /]# ln -s /home/cobbler /var/www
*导入时间较长, 请耐心等待!!!在正常导完之后会给出如下提示:*
\*\*\* TASK COMPLETE \*\*\*
*有时可能会出现卡住的现象,如果导入时间过长,可通过比对文件大小来确定是否已经正常导入*
#比对文件大小的方法

    [root@localhost /]# du -sh /var/www/cobbler/ks_mirror/rhel6.5x86_64/
    [root@localhost /]# du -sh /mnt
*如果上述两个命令执行过显示的结果出入较大, 则可能文件没有正常导入
在重新导入之前最好先把之前的内容删除再导入
cobbler [distro] remove –name=[CentOS-6.7] 方括号中的内容根据自己的情况来填写 ,
        更多命令通过cobbler –help 来查看
        剩下其它系统导入方法类似，只是名字和路径更改下即可。重复上面的操作，把
        其他的系统镜像文件导入到Cobbler导入完成之后，可通过 cobbler list 来查看
        导入的结果。*

③. 创建kickstarts自动安装脚本（For Centos/RHEL)
        注意：这是关键步骤之一
        由于需要安装的操作系统发行厂商不同，因此KS文件的写法要求，也不一而足。
        本文只讨论 CentOs/RHEL 系列的 KS配置
        另外：操作系统 版本不同，KS也存在一定的差异，比如CentOS5 ,和CentOS6下就
        有不同，切记！
        默认kickstart文件是/var/lib/cobbler/kickstarts/sample_end.ks，需要手动
        为每个发行版单独指定，或单独修改。
        自定义ks文件:手动的修改已有的Kickstart文件用system-config-kickstart工具生成Kickstart文件

注意：kickstarts自动安装脚本中不允许有中文（注释有中文也不行），否则会
报错,可以使用ksvalidator程序验证kickstart文件中是否有语法错误。

    [root@localhost /]# ksvalidator  ks.cfg 
④、修改profile指定新的KS启动文件
按照操作系统版本分别关联系统镜像文件和kickstart自动安装文件
       在第一次导入系统镜像时，cobbler会给安装镜像指定一个默认的kickstart自动安装文件  例如：rhel6.5-x86_64版本的kickstart自动安装文件为：/var/lib/cobbler/kickstarts/sample_end.ks

#cobbler ks相关命令

    cobbler profile report --name rhel6.5-x86_64 #查看profile设置
    cobbler distro report --name rhel6.5-x86_64  #查看安装镜像文件信息
    cobbler profile remove --name=rhel6.5-x86_64  #移除profile
    cobbler profile add --name=rhel6.5-x86_64 --distro=rhel6.5-x86_64
         --kickstart=/var/lib/cobbler/kickstarts/rhel6.5-x86_64.ks  #添加
          
    cobbler profile edit --name=rhel6.5-x86_64  --distro=rhel6.5-x86_64
         --kickstart=/var/lib/cobbler/kickstarts/rhel6.5-x86_64.ks  #编辑
          
    命令：cobbler profile add|edit|remove --name=安装引导名 --distro=系统
    镜像名 --kickstart=kickstart自动安装文件路径
    参数说明：
       --name：自定义的安装引导名，注意不能重复
       --distro：系统安装镜像名，用cobbler distro list可以查看
       --kickstart：与系统镜像文件相关联的kickstart自动安装文件（此文件必须预
         先准备好）
    更多命令参数可执行cobbler --help查看

 Cobbler最主要的setting file就是/etc/cobbler/settings。Cobbler2.4.0开始引入动态修改模式（Dynamic Settings），我们只需启动这一模式，便不用再手  动修改这个文件了。该文件是YAML格式的，如果直接修改setting文件，则必须重启cobbler服务才会生效，但如果是通过CLI命令或者是Web GUI进行修改的话，改动会立即生效，无需重启服务。       修改allow_dynamic_settings的值为1

    [root@localhost /]# cd /etc/cobbler/
    [root@localhost /]# cp settings settings.save
    [root@localhost /]# sed -i 's/^[[:space:]]\+/ /' /etc/cobbler/settings
    [root@localhost /]# sed -i 's/allow_dynamic_settings: 0/allow_dynamic_settings: 1/g' /etc/cobbler/settings
 修改该配置后重启cobbler服务

    [root@localhost /]# /etc/init.d/cobblerd restart 
 这个时候，你就可以通过命令行来动态修改cobbler配置

