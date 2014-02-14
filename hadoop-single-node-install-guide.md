一、安装虚拟机
================

二、准备安装环境
================

2.1 安装所必须的软件
------------------

首先，Hadoop 需要 Java 运行环境。

> 这里，我们使用系统自带的 OpenJDK 7。关于是否应该使用 Oracle 的 JDK 的问题，在这里完全没有必要。在 Java 6 的年代，由于  Java 的开放源代码进程问题，OpenJDK 6 和 Oracle JDK 6 还是有一些不一致的，进入到 Java 7后，OpenJDK 和 Oracle JDK 已经一致了，所以可以放心的使用 OpenJDK 7。这一点 Hadoop 官方已经明确给出确认了，需要了解进一步信息的可以看这里：http://wiki.apache.org/hadoop/HadoopJavaVersions

其次，Hadoop 需要使用 SSH 连接各个主机。即使是伪分布模式，也需要连接本机。所以需要安装 SSH 服务器。

此外，我们编辑配置文件的时候，我将使用 `nano` 编辑器。

> 我知道许多人会使用 `vi`，但是从易用性上，`nano` 要比 `vi` 简单。使用 `nano` 不需要去背那些 `vi` 命令和快捷键，很适合新手。`nano` 在许多 ubuntu 系统里都是默认安装的，不需要额外的安装的。在此写进安装命令只是以防万一 （比如在使用 `vmbuilder` 构建的精简版的 Ubuntu 中，很多命令都不会被安装）。

和 Windows 不同，在 Ubuntu 里，安装软件非常简单，只需要一条命令即可，系统会从网上找到需要的软件包进行下载并安装。而且，你不需要担心版本之间的依赖关系、是否互相有冲突、安装位置等等。安装上面的软件，只需执行：

```bash
sudo apt-get install openjdk-7-jdk openssh-server nano
```

当询问是否安装的时候，按 `y` 回车

2.2 基本的系统配置
------------------

### 2.2.1 配置 SSH 无密码登陆

Hadoop 的 master 在控制 slaves 的时候，比如 NameNode 启动、停止 DataNode 等，需要使用 SSH 连接相应的节点。如果不配置无密码登陆，那么在启动停止的时候，都需要人工手动输入各个节点的密码。这是不现实的。

那么既保证安全，又能够避免麻烦的办法有两种，一种是使用 `ssh-agent`，在生产环境中可以考虑使用，配置较复杂；另一种则是使用 SSH 密钥登陆，这个操作非常简单，很适合实验环境配置，也是绝大多数网上教程使用的方法。

其原理很简单，只是简单的非对称加密，大学学过应用密码学课程的人都应该很熟悉，记不清的可以搜索关键字，非对称加密、RSA 等。在这里不赘述了。

在 Ubuntu 上配置无密码登陆非常容易。只需要两条命令。

首先，我们需要生成一对儿密钥：公钥和私钥。只需要使用命令：

```bash
ssh-keygen
```

一路回车即可，不需要输入任何东西，就生成好了。这条命令会将生成的密钥对放入 `~/.ssh` 目录下，因此，可以执行下面的命令去确认该文件夹里面是否已经成功存在所生成的密钥：

```bash
ls ~/.ssh
```

然后，我们需要将公钥添加到目标主机的授权列表里。

比如，在这里我们将先配置 Hadoop 伪分布模式，因此我们需要保证 SSH 登录本机的时候，不需要密码。我们可以先登录本机确认一下：

```bash
ssh localhost
```

如果之前没有配置过密钥登录的，这里会提示用户输入密码。这是我们不希望出现的。于是我们用下面的这一条命令来将公钥添加到本地的授权列表。

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub localhost
```

提示输入本机密码，输入密码后回车，这样就把公钥添加到本机的授权列表了。大家可以执行：

```bash
ls ~/.ssh
```

来看一眼，会发现，里面多了一个 `authorized_keys` 文件，这个文件就是授权公钥列表文件。这里面包含的就是我们刚刚提交的公钥。有了这个，我们登录本机就不需要密码了。

在命令行里再登录一下试试看：

```bash
ssh localhost
```

如果配置都是正确的，会发现，没有刚才的密码提示了。

> 注意到网上许多教程在配置无密码登录的时候，都是用的是命令：
> 
> ```bash
> cat ~/.ssh/id_rsa.pub >> ~/.ssh/authroized_keys
> ```
> 
> 这将直接手动修改授权文件`authroized_keys`。这是比较古老的方法，可以工作，但是已不推荐使用了，因为如果命令敲错，比如少打了一个大于号，就会导致认证密钥列表被覆盖，从而导致其它的一些应用登录可能出现问题，又或者授权文件的文件名敲错，而不会有任何提示，结果还是无法无密码登录等。而且，这个方法在跨主机的时候就更复杂了，还需要把文件先复制到目标主机上。因此，除非有什么特殊的原因，推荐使用刚才所描述的 `ssh-copy-id`。


### 2.2.2 配置主机名及静态IP地址

对于单节点的伪分布模式，本步骤可以忽略，只要在 Hadoop 配置文件中使用 `localhost` 就可以启动。完全不会有问题。但是，在多节点全分布模式下，这一步就是必须的了。在这里配置的原因是考虑到将来还会用这个节点做多节点实验。

此外，还有一个原因是，如果你的 Ubuntu 是运行于虚拟机中的，那么配置好了静态IP和主机名，就会很方便外部主机通过浏览器访问 Hadoop 的界面，比如查看 Hadoop 状态、浏览 HDFS 之类的。如果没有配置那些，也可以访问，但是会有一些麻烦。

在配置之前，我们先定义一下将要配置网络环境。

我们的 Ubuntu 是安装在 VirtualBox 虚拟机中的，网卡连接方式的是 VirtualBox 虚拟机的桥接网卡，桥接表明虚拟机和实际的物理机处于同级的位置，也就是说从外部看来，虚拟机和主机是局域网中两台机器，因此同属局域网的网段。

该网络的网段是： `10.0.1.0/24`

网关是： `10.0.1.1`

该网关同时也有 DNS 转发的功能。

我们今天配置的伪分布的单节点，以及将来配置的全分布的各个节点将都在这个网络下，为了方便起见，我们先定义一下各个节点的 IP 和主机名：

```bash
10.0.1.110 hadoop-master
10.0.1.111 hadoop-secondary
10.0.1.121 hadoop-data1
10.0.1.122 hadoop-data2
10.0.1.123 hadoop-data3
10.0.1.124 hadoop-data4
10.0.1.125 hadoop-data5
10.0.1.126 hadoop-data6
```

前面是该节点应该使用的IP，后面是该节点的主机名。今天我们将在 `hadoop-master` 上面配置单节点伪分布模式。

下面开始配置。



#### 1) 修改本机主机名

```bash
sudo nano /etc/hostname
```


这是一个一行内容的文本文件，将其修改为 `hadoop-master`。

然后按 `Ctrl+x`，会提示已修改，是否保存，输入 `y` 回车，还会再确认一下文件名，直接回车即可，文件就保存了。



#### 2) 配置静态 IP

```bash
sudo nano /etc/network/interfaces
```

将其中的

```bash
iface eth0 inet dhcp
```

替换为

```bash
iface eth0 inet static
    address 10.0.1.110
    netmask 255.255.255.0
    gateway 10.0.1.1
    dns-nameservers 10.0.1.1 8.8.8.8
```

和刚才一样，`Ctrl+x`，保存退出。



#### 3) 配置主机名和 IP 对应关系

```bash
sudo nano /etc/hosts
```


在文件尾部添加：

```bash
10.0.1.110 hadoop-master
10.0.1.111 hadoop-secondary
10.0.1.121 hadoop-data1
10.0.1.122 hadoop-data2
10.0.1.123 hadoop-data3
10.0.1.124 hadoop-data4
10.0.1.125 hadoop-data5
10.0.1.126 hadoop-data6
```
 
保存退出。



#### 4) 使配置生效

```bash
sudo ifdown eth0 && sudo ifup eth0
```

可以用 `ifconfig` 来检查自己的网络配置，或者直接 `ssh hadoop-master` 试一下。如果碰到错误，检查一下配置文件。



### 2.2.3 禁用 IPv6

由于 Hadoop 只在 IPv4 开发，因此对 IPv6 的兼容性目前还存在问题。因此，最好禁用系统的 IPv6，防止不必要的麻烦。

```bash
sudo nano /etc/sysctl.d/10-disable-ipv6.conf
```

在文件中添加如下内容：

```bash
net.ipv6.conf.all.disable_ipv6=1
```

使配置文件生效：

```bash
sudo sysctl -p /etc/sysctl.d/10-disable-ipv6.conf
```

三、安装 Hadoop
==================

3.1 下载 Hadoop 安装文件
------------------

我们将安装当前 1.x 系列最新的版本 1.2.1，访问官网：http://hadoop.apache.org 

`Download Hadoop` ⇨ `releases` ⇨ `Download` ⇨ `Download a release now!` ⇨ 选择合适的镜像 ⇨ `hadoop-1.2.1` ⇨ 我们将下载 `hadoop-1.2.1-bin.tar.gz` 这个文件。

这个文件可以下载到本地，在传到 Ubuntu 上，但是更好的办法是直接在 Ubuntu 中下载。由于我们已经配置好了静态 IP，因此现在可以在主机上使用 SSH 连接 `hadoop-master`了。

```bash
ssh wombat@hadoop-master
```
登录进去后，我们准备 hadoop 的环境以及下载。我们将创建一个 `~/hadoop` 目录，以后所有的 hadoop 相关的软件都会放到该目录下。

```bash
mkdir ~/hadoop
cd ~/hadoop
```

> 这里需要说明的是，许多教程都最终将 hadoop 放到了 `/usr/local/hadoop` 目录下，这样做会带来很多权限问题。而很多教程使用 `chown` 来解决权限冲突，这是非常错误的，很多时候是因为写这些教程的人对 Linux 不熟悉，把 Windows 的一些习惯带到了 Linux 里来。生产环境的搭建绝不是这么简单粗暴的改变所有权。除了需要将 hadoop 放到符合 Linux 目录结构的位置外，包括配置文件和日志，也都应该指向正确的位置，并且建立适当的用户组，以及对不同的目录使用不同的权限，这将引入很多额外的工作，因此不适合在初学时涉及。对于开发和实验环境的搭建，我们这里的做法非常简单，足以胜任后续的学习、开发。

```bash
wget http://apache.dataguru.cn/hadoop/common/hadoop-1.2.1/hadoop-1.2.1-bin.tar.gz
```

如果网络环境较差，则需要检查下载的文件是否正确。执行 

```bash
sha1sum hadoop-1.2.1-bin.tar.gz
```

如果输出和下面的内容一样，则说明下载正确了。

```
267748f61c9e27c3e1895453863b435458255939  hadoop-1.2.1-bin.tar.gz
```

如果下载正确，则对压缩文件进行解压缩：

```bash
tar -zxvf hadoop-1.2.1-bin.tar.gz
```

执行上述命令后，会生成 `hadoop-1.2.1` 目录，至此，我们可以开始正式的配置了。



3.2 配置 Hadoop
------------------

### 3.2.1 配置 `.bashrc` 文件

hadoop 的可执行文件在 `hadoop-1.2.1/bin` 下，我们如果想执行该文件，或者使用完整路径的方式，或者将该路径加入到 `PATH` 环境变量中，以后我们直接执行文件即可。

```bash
sudo nano ~/.bashrc
```

在最后一行添加：

```bash
export PATH=$PATH:$HOME/hadoop/hadoop-1.2.1/bin
```

保存退出。然后应用该配置。

```bash
source ~/.bashrc
```

### 3.2.2 配置 `conf/hadoop-env.sh` 文件

为方便起见，我们将当前目录换到 conf 目录下：

```bash
cd hadoop-1.2.1/conf
```

Hadoop 需要在配置文件中指定 `JAVA_HOME` 环境变量。

```bash
nano hadoop-env.sh
```

在最后一行添加：

```bash
JAVA_HOME=/usr/lib/jvm/java-1.7.0-openjdk-amd64/
```

保存退出。

此时，我们可以通过显示 hadoop 的版本来判断之前的配置是否正确。

```bash
hadoop version
```

### 3.2.3 配置 `*-site.xml`

之前的配置一直是环境配置，现在才是真正的 Hadoop 相关的配置。对于单节点伪分布模式，配置非常简单，我们只需要修改三个 `*-site.xml` 文件即可。默认情况下，这三个文件都包含一个空的`<configuration>` ，在这里，我们需要为三个文件的`<configuration>`中加入对应的配置。

#### `core-site.xml`:

```xml
<property>
    <name>fs.default.name</name>
    <value>hdfs://h1-master:9000</value>
</property>
```


#### `hdfs-site.xml`:

```xml
<property>
    <name>dfs.replication</name>
    <value>1</value>
</property>
```


#### `mapred-site.xml`:

```xml
<property>
    <name>mapred.job.tracker</name>
    <value>h1-master:9001</value>
</property>
```

分别保存退出。至此，Hadoop 单节点伪分布模式就配置完毕了。





































