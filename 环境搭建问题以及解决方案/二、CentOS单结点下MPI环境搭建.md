## 第二步，CentOS单结点下MPI环境搭建

#### 一、配置网络环境

首先配置网络环境，到官网上下载，我下载的源码包为mpich3.2.2.tar.gz(网址：http://www.mpich.org/static/downloads/3.2/）

注意这里的网络环境配置可以参考CSDN上的这篇文章https://blog.csdn.net/weixin_46144832/article/details/123762429?app_version=5.15.5&code=app_1562916241&csdn_share_tail=%7B%22type%22%3A%22blog%22%2C%22rType%22%3A%22article%22%2C%22rId%22%3A%22123762429%22%2C%22source%22%3A%22weixin_52099311%22%7D&uLinkId=usr1mkqgl919blen&utm_source=app

我产生的主要原因是没有关注网络适配器

首先打开对应的虚拟机的虚拟机设置，点击“网络适配器” ，选择 NAT 模式，然后点确定。

重要的是下一步，在VM的编辑选项中选择虚拟网格编辑器。选中有 “NAT模式” 的那行记录，我的电脑上也是VNnet8是NAT模式，然后确保图中标记的两个勾必须打上，如果没有默认勾选，应该手动勾选 。然后**点击 “NAT设置”**，这一步很重要，因为你需要**记录子网IP 、子网掩码以及网关**。

 然后在虚拟机上cd到/etc/sysconfig/network-scripts中利用vim ifcfg-ens33（我的这里和博主的不一样）设置Linux网络接口，在配置文件需要修改如下内容，可以不需要双引号

- BOOTPROTO=static（指定网络协议类型为静态 IP 地址，而不是动态主机配置协议 (DHCP)。这意味着你手动分配了一个 IP 地址，而不是从 DHCP 服务器自动获取一个 IP 地址）

- ONBOOT=yes（指定系统在启动时自动激活该网络接口。如果该值设置为 no，则需要手动运行 ifup 命令来启用该接口）

- IPADDR（指定分配给网络接口的 IPv4 地址，需要和刚才记录的ip在同一个网段）

- NETMASK（指定用于定义子网边界的掩码，使用刚刚记录的子网掩码）

- GATEWAY（定系统的默认网关 IP 地址，它代表了该网络的出口点，使用刚刚记录的网关地址）

- DNS1=8.8.8.8（指定首选 DNS 解析器的 IP 地址）

- DNS2=8.8.4.4 （指定备用 DNS 解析器的 IP 地址）

- ZONE=public （指定防火墙的区域，例如公共、专用等。这有助于管理网络流量和安全性）

  

#### 二、下载解压安装

在下载到的文件处打开终端，将文件移动到/opt下（我自己设置的文件路径，也可以换）

```bash
[root@localhost 下载]#mv mpich-3.3.2.tar.gz /opt
```

cd到/opt下并解压文件到当前文件夹：

```bash
[root@localhost 下载]#cd /opt
[root@localhost opt]#ls
mpich-3.3.2.tar.gz  rh  share
[root@localhost opt]#tar -zxvf mpich-3.2.2.tar.gz
```

查看解压的文件有

```bash
[root@localhost opt]#ls
mpich-3.3.2  mpich-3.3.2.tar.gz  rh  share
```

进入解压的文件 mpich-3.3.2并查看内容

```bash
[root@localhost opt]# cd mpich-3.3.2
[root@localhost mpich-3.3.2]# ls
aclocal.m4  configure.ac  examples     mpich.def         RELEASE_NOTES
autogen.sh  contrib       maint        mpich-doxygen.in  src
CHANGES     CONTRIBUTING  Makefile.am  mpi.def           subsys_include.m4
confdb      COPYRIGHT     Makefile.in  README            test
configure   doc           man          README.envvar     www
```

进行编译环境配置，--prefix表示安装路径

```bash
[root@localhost mpich-3.3.2]# ./configure --prefix=/opt/mpich
```

这里出现了一个小问题

```bash
[root@localhost mpich-3.3.2]# ./configure --prefix=/opt/mpich
Configuring MPICH version 3.3.2 with  '--prefix=/opt/mpich'
Running on system: Linux localhost.localdomain 3.10.0-1160.90.1.el7.x86_64 #1 SMP Thu May 4 15:21:22 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
checking build system type... x86_64-unknown-linux-gnu
checking host system type... x86_64-unknown-linux-gnu
checking target system type... x86_64-unknown-linux-gnu
checking for icc... no
checking for pgcc... no
checking for xlc... no
checking for xlC... no
checking for pathcc... no
checking for gcc... no
checking for clang... no
checking for cc... no
configure: error: in /opt/mpich-3.3.2':
configure: error: no acceptable C compiler found in $PATH
See `config.log' for more details
```

这是因为没有安装gcc环境，解决办法

```bash
[root@localhost]# sudo yum install gcc
```

继续

```bash
[root@localhost mpich-3.3.2]# ./configure --prefix=/opt/mpich
```

还是产生错误

```bash
configure: error: No Fortran 77 compiler found. If you don't need to
        build any Fortran programs, you can disable Fortran support using
        --disable-fortran. If you do want to build Fortran
        programs, you need to install a Fortran compiler such as gfortran
        or ifort before you can proceed.
```

修改,

```bash
[root@localhost mpich-3.3.2]# ./configure --prefix=/opt/mpich --disable-fortran
```

产生如下报错

```bash
configure: error: Aborting because C++ compiler does not work.  If you do not need a C++ compiler, configure with --disable-cxx
```

还是因为没有下载好编译的环境，解决方案如下，就是一下子下载全部的

```bash
[root@localhost mpich-3.3.2]#yum install cpp gcc gcc-c++ gcc-gfortran gcc-objc++ gcc-objc gcc44 libgcc libgfortran libobjc libstdc++ libstdc++-devel
```

解决完毕后进行编译安装

```bash
[root@localhost mpich-3.3.2]# make install
```

上面这个过程很慢，耐心等待，下载完毕后进入到bin目录并复制一下路径，为后面的添加环境变量做准备

```
[root@localhost mpich-3.3.2]# ls
[root@localhost mpich-3.3.2]# cd /opt/mpich/bin/
[root@localhost bin]# pwd
/opt/mpich/bin
```

得到的结果进行复制，比如我得到的就是/opt/mpich/bin

#### 三、添加环境变量

```bash
[root@localhost bin]# vim /etc/profile
```

按下i进行编辑，在最后一行添加如下，$PATH:后是你前面自己复制的路径

```bash
##mpich##
export PATH=$PATH:/opt/mpich/bin
```

最后esc退出编辑模式，shift+：输入wq保存退出

下面是更新（激活）环境变量

```bash
[root@localhost bin]# source /etc/profile
```



#### 四、测试安装

进入刚才解压的mpi-3.2.1目录，然后ls，发现里面有个examples文件夹，进入examples文件夹，ls一下

```bash
[root@localhost bin]# cd /opt
[root@localhost opt]# cd mpich-3.3.2/
[root@localhost mpich-3.3.2]# ls

[root@localhost mpich-3.3.2]# cd examples/
[root@localhost examples]# ls
```

可以看到里面有一个hellow.c的c源文件，我们通过mpi接口对其进行编译

```'
[root@localhost examples]#mpicc  hellow.c -o hellow
```

-o 是objective的缩写，hellow是文件名，意思是把hellow.c 源文件编译成名字为hellow的目标（可执行）文件。编译完成后ls发现examples目录下会多出一个hellow文件。

```bash
[root@localhost examples]#mpirun -np N ./hellow
```

-np 表示number of processors, 即进程数，N 自己取值。比如我的命令和结果为：

```bash
[root@localhost examples]#mpirun -np 4 ./hellow
```

结果类似如下则说明单机配置mpi成功

```
Hello world from process 0 of 4
Hello world from process 1 of 4
Hello world from process 2 of 4
Hello world from process 3 of 4
```

