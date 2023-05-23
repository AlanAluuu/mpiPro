## 第三步，Linux 多节点下 MPI 集群环境搭建

即多机配置MPI，多个节点跑程序

#### 1、虚拟机的克隆

这里需要设置三个节点（CentOS_7_6_Server、client1、client2）

前面已经实现了单机配置MPI以及nfs，所以接下来要进行集群，首先对之前的单机进行克隆

点击你想要进行克隆的虚拟机，选择虚拟机——》管理——》克隆

![Alt text](https://github.com/AlanAluuu/mpiPro/blob/main/%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA%E9%97%AE%E9%A2%98%E4%BB%A5%E5%8F%8A%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88/screenshots/kl1.png)

然后选择创建完整克隆，因为完成克隆相对于原始虚拟机时完全独立的，链接克隆相当于从副虚拟机快照创建的。

![Alt text](https://github.com/AlanAluuu/mpiPro/blob/main/%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA%E9%97%AE%E9%A2%98%E4%BB%A5%E5%8F%8A%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88/screenshots/kl2.png)

然后进行ip的修改，在虚拟机上cd到/etc/sysconfig/network-scripts中利用sudo vim ifcfg-ens33（我的这里和博主的不一样）设置Linux网络接口，在配置文件仅仅需要修改ip地址即可，网段不要变化。记得设置后要systemctl restart network重启一下网络，要不你输入ip addr显示ip地址仍然没有变化。最后ping一下，看克隆后的虚拟机和原虚拟机能不能相互连接通。也可以继续去测试一下克隆后的MPI能不能用。

#### 2、host配置

首先进行host配置，我配置的信息如下，命令行输入sudo vim /etc/hosts，输入密码，插入以下内容（这个跟你设置的三台机子的ip地址以及机器的名称有关，按照这种Ip+名称的格式设置即可）

```bash
192.168.11.1 server
192.168.11.11 client1
192.168.11.12 client2
```

然后通过ping client1来测试可否通过主机名称ping通

#### 3、SSH 登录

我们平时登录Linux服务器的时候，经常是使用用户名和密码进行登录，但是如果我们要使用它进行代码连接或者其他操作的情况下，我们需要一种更为安全的方式进行登录，就需要privateKey登录 SSH 服务器。

- RSA 非对称加密
- 在 SSH 登录时可以使用 RSA 密钥登录
- 使用工具ssh-keygen可以创建 SSH 密钥

进入 Linux 系统目录下 .ssh 目录

```bash
cd ~/.ssh/
```

此时看到报错-bash: cd: /root/.ssh/: 没有那个文件或目录

```bash
[root@localhost ~]# cd ~/.ssh/
-bash: cd: /root/.ssh/: 没有那个文件或目录
```

解决的具体方法就是执行下面代码

```bash
ssh localhost
```

然后会让你输入yes以及用户密码，输入过后继续执行cd ~/.ssh/，就可以进入到相应目录了。

接下来即**执行 ssh-keygen 创建密钥对**

```bash
ssh-keygen -t rsa
```

执行密钥生成命令，会有提示，接着连按3次回车即可。之后你可以用ls查看一下内容

```bash
[root@localhost .ssh]# ls
id_rsa  id_rsa.pub  known_hosts
```

我们发现目录下没有 authorized_keys，需要创建一个，将 id_rsa.pub 文件的内容输出到 authorized_keys

```bash
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

之后ssh 登录到 client1、client2 上面执行相同的操作，并将产生的公钥发送给service

client1执行下面的操作

```
ssh localhost 
cd ~/.ssh/
ssh-keygen -t rsa
scp ./id_rsa.pub mpiuser@server:~/.ssh/node2_id_rsa.pub
```

client2执行下面的操作

```
ssh localhost 
cd ~/.ssh/
ssh-keygen -t rsa
scp ./id_rsa.pub mpiuser@server:~/.ssh/node3_id_rsa.pub
```

可是出现了如下的报错

```
The authenticity of host 'service (192.168.11.1)' can't be established.
ECDSA key fingerprint is SHA256:kXFbQEM+3q5FD25gE8vniQic4p+9gDPVeTaYqH8rrp4.
ECDSA key fingerprint is MD5:7f:db:05:91:e6:42:ff:65:fd:da:87:8e:45:23:89:d3.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'service,192.168.11.1' (ECDSA) to the list of known hosts.
```

上网查了查，原因在于每次远程登录Linux的时候，Linux都要检查一下，当前访问的计算机公钥是不是在~/.ssh/know_hosts中，这个文件是OpenSSH记录的。当下次访问相同计算机时，OpenSSH会核对公钥。如果公钥不同，OpenSSH会发出警告，避免你受到DNS Hijack之类的攻击。SSH对主机的public_key的检查等级是根据StrictHostKeyChecking变量来配置的。默认情况下，StrictHostKeyChecking=ask。简单所下它的三种配置值：

- 1.StrictHostKeyChecking=no 最不安全的级别，当然也没有那么多烦人的提示了，相对安全的内网测试时建议使用。如果连接server的key在本地不存在，那么就自动添加到文件中（默认是known_hosts），并且给出一个警告。
- 2.StrictHostKeyChecking=ask 默认的级别，就是出现刚才的提示了。如果连接和key不匹配，给出提示，并拒绝登录。
- 3.StrictHostKeyChecking=yes 最安全的级别，如果连接与key不匹配，就拒绝连接，不会提示详细信息。

一种解决方法如下所示，即修改/etc/ssh/sshd_config配置文件

```
[root@localhost .ssh]# cd /etc/ssh/
[root@localhost ssh]# ls
moduli       ssh_host_ecdsa_key      ssh_host_ed25519_key.pub
ssh_config   ssh_host_ecdsa_key.pub  ssh_host_rsa_key
sshd_config  ssh_host_ed25519_key    ssh_host_rsa_key.pub
[root@localhost ssh]# vim /etc/ssh/sshd_config
```

在ssh_config配置文件中添加如下内容，彻底去掉SSH主机验证

```bash
StrictHostKeyChecking no
UserKnownHostsFile /dev/null
```

仍然报错，显示

```bash
[root@localhost .ssh]# scp ./id_rsa.pub mpiuser@service:~/.ssh/node2_id_rsa.pub
mpiuser@centos_7_6_service's password: 
Permission denied, please try again.
```

该怎么解决呢，vim /etc/ssh/sshd_config 找到 PermitRootLogin 取消注释，并改为 yes

```bash
[root@localhost .ssh]# vim  /etc/ssh/sshd_config
[root@localhost .ssh]# service sshd restart
Redirecting to /bin/systemctl restart sshd.service
```

继续仍然报错，那说明不是配置的问题，搜了搜，可能是公钥转发以及ssh设置到root目录下了，所以直接用户打开命令行，不要使用su进入root目录。重复上面的操作

```bash
ssh localhost 
cd ~/.ssh/
ssh-keygen -t rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

这里不用命令行的方式进行公钥整合了，查看三台主机的id_rsa.pub，直接将进行复制粘贴，如下所示

对于server

```
[lulu@server .ssh]$ cat id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDekdbHjB7m20jOfs/1W8Ilcbtt9q8CrUfnVRq61ACqGOJkzb0ebbLylCwiDS6B1GG/4iLnbY0sB0809oeRZD7n5WfVzcknoxwJdp0idKk9NDOZnLB8W0NTY8ZtdpI6mz1DuEGJD0UeG0zVyHit6NBJwcY7le/EteBaYT67O0aRYGiMRyZ3mj5gpRaunTjjvQlELQUXox80zkbiGZfacy7WnPAGttgASPUAxqEq4kJDhFhSqQU1N2njPMrg9CBB88/o7fQGwsIoQZ5p+SjWY2rke216OnNKsXr87SlpZmmZh0U8JFxSXKA+l/Nh1x2ftnuX/tLz54OSygTKbdSwQEx7 lulu@server
```

对于client1

```
[lulu@client1 .ssh]$ cat id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDEQajEtCLfMjOwCBmnhBRh8qbafaAUwVHW1fgEU1kzEtQVTUqm7RBBOgHS63kfGZoMXCm6ueSq6B2lLttIn92ERM8TTXUjwedNuL7IO28UEOdqRwrCEQWWGvdsw5y+L10qne2GsvJD2BLB9cwVHTXjBvhDiVpYZjvu9SrkiBKWs690mHe8n0D5ogrF63TW/s04Y7CQYguiHUXrKc4PGp0E7+0eznKNdFgdtm9+LsWXbk7by6YveFFyPkV+TRcvMgHMa1aXTOU/Mgr0KgWbDMBq+zlVgltV0weM//xOaefcgFnOL3a7pKW75ao86lx+dXJBogbDw9npXarp99Q8nsa7 root@client1
```

对于client2

```bash
[lulu@client2 ~]$ cd .ssh
[lulu@client2 .ssh]$ cat id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDLU4CFtr/NLOK7EqSQA8YSuiDp6am4BoELSeGmbb/v67oGJy18vvQw4O6/KadDl2wdxz1U9G/HyM6vQwMXIpwPwV+62JuRoUsTo/H2TjdWCPrtuFLva4XJj7u4oJ6Tp0U4UBRqWX7QMnL4b7+uTvMiRpNhPC4a8mzGcTIeKaZYwhy4P6WlPQQCQ5ewNtkiTr/19eGxO4I9509APPjEr0GgC0/T/wpplbXDXOL1K0yFiITjFlacPV0f/i3+H+7BMUUe8aOuy1GltW36QDiEIfGJ7GSapf8Xd0PvH6NUNnR2gAjB9O2C46WVeeEeE4VeqQ9hjmmL3qc3UybcS+KjlSuN lulu@client2
```

然后对三个虚拟机的authorized_keys文件进行修改

```bash
[lulu@client2 .ssh]$ vim authorized_keys
```

全部换成三个公钥的组合，即如下内容

```bash
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDekdbHjB7m20jOfs/1W8Ilcbtt9q8CrUfnVRq61ACqGOJkzb0ebbLylCwiDS6B1GG/4iLnbY0sB0809oeRZD7n5WfVzcknoxwJdp0idKk9NDOZnLB8W0NTY8ZtdpI6mz1DuEGJD0UeG0zVyHit6NBJwcY7le/EteBaYT67O0aRYGiMRyZ3mj5gpRaunTjjvQlELQUXox80zkbiGZfacy7WnPAGttgASPUAxqEq4kJDhFhSqQU1N2njPMrg9CBB88/o7fQGwsIoQZ5p+SjWY2rke216OnNKsXr87SlpZmmZh0U8JFxSXKA+l/Nh1x2ftnuX/tLz54OSygTKbdSwQEx7 lulu@server
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDEQajEtCLfMjOwCBmnhBRh8qbafaAUwVHW1fgEU1kzEtQVTUqm7RBBOgHS63kfGZoMXCm6ueSq6B2lLttIn92ERM8TTXUjwedNuL7IO28UEOdqRwrCEQWWGvdsw5y+L10qne2GsvJD2BLB9cwVHTXjBvhDiVpYZjvu9SrkiBKWs690mHe8n0D5ogrF63TW/s04Y7CQYguiHUXrKc4PGp0E7+0eznKNdFgdtm9+LsWXbk7by6YveFFyPkV+TRcvMgHMa1aXTOU/Mgr0KgWbDMBq+zlVgltV0weM//xOaefcgFnOL3a7pKW75ao86lx+dXJBogbDw9npXarp99Q8nsa7 root@client1
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDLU4CFtr/NLOK7EqSQA8YSuiDp6am4BoELSeGmbb/v67oGJy18vvQw4O6/KadDl2wdxz1U9G/HyM6vQwMXIpwPwV+62JuRoUsTo/H2TjdWCPrtuFLva4XJj7u4oJ6Tp0U4UBRqWX7QMnL4b7+uTvMiRpNhPC4a8mzGcTIeKaZYwhy4P6WlPQQCQ5ewNtkiTr/19eGxO4I9509APPjEr0GgC0/T/wpplbXDXOL1K0yFiITjFlacPV0f/i3+H+7BMUUe8aOuy1GltW36QDiEIfGJ7GSapf8Xd0PvH6NUNnR2gAjB9O2C46WVeeEeE4VeqQ9hjmmL3qc3UybcS+KjlSuN lulu@client2
```

#### 四、测试

下面进行一个测试，新建一个 servers 文件

```bash
[lulu@server ~]$ vim servers
```

 对servers 文件写入如下内容

```bash
server:3  #运行3个进程
client1:3  #运行3个进程
client2:3  #运行3个进程
```

 执行下面命令

```bash
[lulu@server ~]$ mpirun -n 9 -f ./servers /opt/mpich-3.3.2/examples/cpi
```

真令人惋惜，我的报错了，啊啊啊啊啊啊

毫无悬念的csdn上搜索，有个解决方法是

```bash
sudo yum install openssh-askpass
```

诶，报错变了，成下面的样子了

```bash
[proxy:0:1@client1] HYDU_sock_connect (utils/sock/sock.c:145): unable to connect from "client1" to "server" (No route to host)
[proxy:0:1@client1] main (pm/pmiserv/pmip.c:183): unable to connect to server server at port 37471 (check for firewalls!)
[proxy:0:2@client2] HYDU_sock_connect (utils/sock/sock.c:145): unable to connect from "client2" to "server" (No route to host)
[proxy:0:2@client2] main (pm/pmiserv/pmip.c:183): unable to connect to server server at port 37471 (check for firewalls!)
```

只看懂了一个检查防火墙，那就先尝试关闭一下防火墙

```bash
[lulu@server ~]$ systemctl stop firewalld
```

继续运行，发现server竟然开始分配给client任务了，不过仍然存在报错

```bash
[lulu@server ~]$ mpirun -n 9 -f ./servers /opt/mpich-3.3.2/examples/cpi
Process 0 of 9 is on server
Process 3 of 9 is on client1
Process 1 of 9 is on server
Process 6 of 9 is on client2
Process 2 of 9 is on server
Process 5 of 9 is on client1
Process 4 of 9 is on client1
Fatal error in PMPI_Reduce: Unknown error class, error stack:
PMPI_Reduce(523)................: MPI_Reduce(sbuf=0x7fff6b0961e0, rbuf=0x7fff6b0961d8, count=1, datatype=MPI_DOUBLE, op=MPI_SUM, root=0, comm=MPI_COMM_WORLD) failed
PMPI_Reduce(509)................: 
MPIR_Reduce_impl(316)...........: 
MPIR_Reduce_intra_auto(202).....: 
MPIR_Reduce_intra_smp(107)......: 
MPIR_Reduce_impl(316)...........: 
MPIR_Reduce_intra_auto(231).....: 
MPIR_Reduce_intra_binomial(125).: 
MPIDI_CH3U_Recvq_FDU_or_AEP(629): Communication error with rank 3
MPIR_Reduce_intra_binomial(125).: 
MPIDI_CH3U_Recvq_FDU_or_AEP(629): Communication error with rank 6
MPIR_Reduce_intra_smp(128)......: 
MPIR_Reduce_impl(316)...........: 
MPIR_Reduce_intra_auto(231).....: 
MPIR_Reduce_intra_binomial(188).: Failure during collective
```

那么可能是client没有设置关闭防火墙以及install openssh-askpass，那么接下来就是在client1和client2上配置sudo yum install openssh-askpass以及systemctl stop firewalld。

再次mpirun -n 9 -f ./servers /opt/mpich-3.3.2/examples/cpi，啊！仰天长啸，终于整成功啦！明天再去做下一步！

```bash
Process 5 of 9 is on client1
Process 3 of 9 is on client1
Process 4 of 9 is on client1
Process 8 of 9 is on client2
Process 2 of 9 is on server
Process 7 of 9 is on client2
Process 0 of 9 is on server
Process 6 of 9 is on client2
Process 1 of 9 is on server
pi is approximately 3.1415926544231256, Error is 0.0000000008333325
wall clock time = 0.133299
```

