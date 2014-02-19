**目录**  *generated with [DocToc](http://doctoc.herokuapp.com/)*

- [前言](#前言)
- [一、安装虚拟机](#一、安装虚拟机)
- [二、准备安装环境](#二、准备安装环境)
	- [2.1 安装所必须的软件](#21-安装所必须的软件)
	- [2.2 基本的系统配置](#22-基本的系统配置)
		- [2.2.1 配置 SSH 无密码登陆](#221-配置-ssh-无密码登陆)
		- [2.2.2 配置主机名及静态IP地址](#222-配置主机名及静态ip地址)
			- [1) 修改本机主机名](#1-修改本机主机名)
			- [2) 配置静态 IP](#2-配置静态-ip)
			- [3) 配置主机名和 IP 对应关系](#3-配置主机名和-ip-对应关系)
			- [4) 使配置生效](#4-使配置生效)
		- [2.2.3 禁用 IPv6](#223-禁用-ipv6)
		- [3.2.1 配置 `PATH` 环境变量](#321-配置-path-环境变量)
- [三、安装 Hadoop](#三、安装-hadoop)
	- [3.1 下载 Hadoop 安装文件](#31-下载-hadoop-安装文件)
		- [3.2.2 配置 `JAVA_HOME` 环境变量](#322-配置-java_home-环境变量)
		- [3.2.3 配置 `*-site.xml`](#323-配置--sitexml)
			- [`core-site.xml`:](#core-sitexml)
	- [3.2 配置 Hadoop](#32-配置-hadoop)
			- [`hdfs-site.xml`:](#hdfs-sitexml)
			- [`mapred-site.xml`:](#mapred-sitexml)
		- [3.2.4 格式化 NameNode](#324-格式化-namenode)
		- [3.3.1 启动 Hadoop](#331-启动-hadoop)
		- [3.3.2 运行示例程序](#332-运行示例程序)
			- [下载](#下载)
			- [解压缩](#解压缩)
			- [将书籍放到 HDFS 云](#将书籍放到-hdfs-云)
			- [执行 `WordCount` 示例程序](#执行-wordcount-示例程序)
		- [3.3.3 停止 Hadoop](#333-停止-hadoop)
	- [3.3 验证 Hadoop 已安装正常](#33-验证-hadoop-已安装正常)

前言
=============

本安装指南是针对 Ubuntu Server 12.04 进行 Hadoop 1.x 单节点伪分布模式配置的。其它系统或其它版本请酌情调整配置方法。

如果不是从 [github](https://github.com/twang2218/hadoop-notes/blob/master/hadoop1-single-node-install-guide.md) 看到此文档，那么该文档很可能已经更新了。请访问 https://github.com/twang2218/hadoop-notes/blob/master/hadoop1-single-node-install-guide.md 以获取最新版本的文档。


一、安装虚拟机
================

1.1 下载 ISO 文件

我们将下载 Ubuntu 12.04 LTS 服务器版 64位版本。(对于无法使用64位版本的，可以使用32位版本代替，但是生产环境中，由于内存常常会超过4G，因此64位版本是必须的。）

访问 http://www.ubuntu.org.cn/download/server  点击“获取 Ubuntu 12.04 LTS”

1.2 添加虚拟机

我们使用 VirtualBox 来做全部的实验。这是一个免费的、开源的虚拟机，非常不错，推荐使用。

...

二、准备安装环境
================

2.1 安装所必须的软件
------------------

首先，Hadoop 需要 Java 运行环境。

| Note: |
| :---- |
| *这里，我们使用系统自带的 OpenJDK 7。关于是否应该使用 Oracle 的 JDK 的问题，在这里完全没有必要。在 Java 6 的年代，由于 Java 的开放源代码进程问题，OpenJDK 6 和 Oracle JDK 6 还是有一些不一致的，进入到 Java 7后，OpenJDK 和 Oracle JDK 已经一致了，所以可以放心的使用 OpenJDK 7。这一点 Hadoop 官方已经明确给出确认了，需要了解进一步信息的可以看这里：http://wiki.apache.org/hadoop/HadoopJavaVersions* |

其次，Hadoop 需要使用 SSH 连接各个主机。即使是伪分布模式，也需要连接本机。所以需要安装 SSH 服务器。

此外，我们编辑配置文件的时候，我将使用 `nano` 编辑器。

| Note: |
| :---- |
| *我知道许多人会选择使用 `vi`，但是从易用性上，`nano` 要比 `vi` 简单。使用 `nano` 不需要去背那些 `vi` 命令和快捷键，很适合新手。`nano` 在许多 ubuntu 系统里都是默认安装的，不需要额外的安装的。在此写进安装命令只是以防万一 （比如在使用 `vmbuilder` 构建的精简版的 Ubuntu 中，很多命令都不会被安装）。* |

> `unzip` 是可选的，但我们将来会用它对下载的 `.zip` 数据文件进行解压缩。

> `htop` 也是可选的，但是使用 `htop` 对节点负载进行分析与观察是非常有帮助的。

和 Windows 不同，在 Ubuntu 里，安装软件非常简单，只需要一条命令即可，系统会从网上找到需要的软件包进行下载并安装。而且，你不需要担心版本之间的依赖关系、是否互相有冲突、安装位置等等。安装上面的软件，只需执行：

```bash
sudo apt-get install openjdk-7-jdk openssh-server nano unzip htop
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

一路回车即可，不需要输入任何东西，就生成好了。这条命令会将生成的密钥对放入 `~/.ssh` 目录下，因此，可以执行 `ls ~/.ssh` 去确认该文件夹里面是否已经成功存在所生成的密钥。

然后，我们需要将公钥添加到目标主机的授权列表里。比如，在这里我们将先配置 Hadoop 伪分布模式，因此我们需要保证 SSH 登录本机的时候，不需要密码。如果之前没有配置过使用密钥登录的，`ssh localhost` 会提示用户输入密码。这是我们不希望出现的。于是我们用下面的这一条命令来将公钥添加到本地的授权列表。

```bash
ssh-copy-id -i localhost
```

提示输入本机密码，输入密码后回车，这样就把公钥添加到本机的授权列表了。可以执行 `ls ~/.ssh`，会发现，目录里面多了一个 `authorized_keys` 文件，这个文件就是授权公钥列表文件。这里面包含的就是我们刚刚提交的公钥。有了这个，我们登录本机就不需要密码了。

在命令行里再登录一下试试看：

```bash
ssh localhost
```

如果配置都是正确的，会发现，没有刚才的密码提示了。

| Note: |
| :---- |
| *注意到网上许多教程在配置无密码登录的时候，都是用的是命令 `cat ~/.ssh/id_rsa.pub >> ~/.ssh/authroized_keys` 这将直接手动修改授权文件 `authroized_keys`。这是比较古老的方法，可以工作，但是已不推荐使用了，因为如果命令敲错，比如少打了一个大于号，就会导致认证密钥列表被覆盖，从而导致其它的一些应用登录可能出现问题，又或者授权文件的文件名敲错，而不会有任何提示，结果还是无法无密码登录等。而且，这个方法在跨主机的时候就更复杂了，还需要把文件先复制到目标主机上。因此，除非有什么特殊的原因，推荐使用刚才所描述的 `ssh-copy-id`。*|


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

| Note: |
| :---- |
| *从这里开始，我们就可以不用在虚拟机窗口中配置了，我们可以使用 SSH 客户端（即终端），如[Putty](http://www.putty.org/)、[Bitvise SSH Client](http://www.bitvise.com/ssh-client-download)、[WinSCP](http://winscp.net/eng/docs/free_ssh_client_for_windows)、[Xshell](http://www.netsarang.com/products/xsh_overview.html)等工具来连接 `hadoop-master`。使用终端的好处是显而易见的，比如我们可以方便的复制、粘贴文字到终端窗口，也可以将里面的内容复制、粘贴出来。此外，上面的许多 SSH 客户端支持 SFTP，通过 SFTP 我们可以很轻松的将文件上传到 Linux 主机上。<br /><br /> 需要注意的是，在主机上也需要配置上述主机名和IP地址对应关系的，对于 Linux 来说，和第三步一样，配置 `/etc/hosts` 文件；对于 Windows 而言，是编辑 `C:\Windows\System32\drivers\etc\hosts` 文件，都是将第3步的内容追加到文件末尾。* |

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

可以使用命令 `ifconfig | grep inet6` 检查是否已经禁用了 IPv6。如果没有任何结果返回，则说明 IPv6 已经成功禁用了；否则，则说明禁用 IPv6 没有成功。


三、安装 Hadoop
==================

3.1 下载 Hadoop 安装文件
------------------

我们将安装当前 1.x 系列最新的版本 1.2.1，访问官网：http://hadoop.apache.org 

`Download Hadoop` ⇨ `releases` ⇨ `Download` ⇨ `Download a release now!` ⇨ 选择合适的镜像 ⇨ `hadoop-1.2.1` ⇨ 我们将下载 `hadoop-1.2.1-bin.tar.gz` 这个文件。

这个文件可以下载到本地，在传到 Ubuntu 上，但是更好的办法是直接在 Ubuntu 中下载。由于我们已经配置好了静态 IP，因此现在可以在主机上使用 SSH 连接 `hadoop-master`了。

```bash
ssh hadoop-master
```

登录进去后，我们准备 hadoop 的环境以及下载。我们将创建一个 `~/hadoop` 目录，以后所有的 hadoop 相关的软件都会放到该目录下。

```bash
mkdir ~/hadoop
cd ~/hadoop
```

| Note: |
| :---- |
| *这里需要说明的是，许多教程都最终将 hadoop 放到了 `/usr/local/hadoop` 目录下，这样做会带来很多权限问题。而很多教程或者没办法，最后使用 `root` 用户操作一切，或者使用 `chown` 来解决权限冲突，这是非常不好的，很多时候是因为写这些教程的人对 Linux 不熟悉，把 Windows 的一些习惯带到了 Linux 里来。生产环境的搭建绝不是这么简单粗暴的改变所有权，更不可能是这样子滥用`root`权限。除了需要将 hadoop 放到符合 Linux 目录结构的位置外，包括配置文件和日志，也都应该指向正确的位置，并且建立适当的用户组，以及对不同的目录使用不同的权限，这将引入很多额外的工作，因此不适合在初学时涉及。对于开发和实验环境的搭建，我们这里的做法非常简单，足以胜任后续的学习、开发。* |

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

### 3.2.1 配置 `PATH` 环境变量

hadoop 的可执行文件在 `hadoop-1.2.1/bin` 下，我们如果想执行该文件，或者使用完整路径的方式，或者将该路径加入到 `PATH` 环境变量中，以后我们直接执行文件即可。

```bash
nano ~/.profile
```

在最后一行添加：

```bash
export PATH=$PATH:$HOME/hadoop/hadoop-1.2.1/bin
```

保存退出。然后应用该配置。

```bash
source ~/.profile
```

| Note: |
| :---- |
| *这里注意到有的教程提到的是修改 `~/.bashrc` 文件。在很多情况下这是工作的，但是某些情况下则不能工作，因此[Ubuntu 官网关于环境变量的配置](https://help.ubuntu.com/community/EnvironmentVariables#A.2BAH4ALw.profile) 建议放在`~/.profile`文件中。* |


### 3.2.2 配置 `JAVA_HOME` 环境变量


Hadoop 需要在使用 `JAVA_HOME` 环境变量。由于该环境变量将在多处使用，并会在 `ssh + 主机名 + 命令` 的形式使用，因此，我们将在全局环境变量中配置：

```bash
sudo nano /etc/environment
```

在文件最后添加：

```bash
JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64/
```

`Ctrl+x`，保存退出。

> 这里虽然也有 PATH 环境变量，但是不要在这里修改它，这里是全局变量。

| Note: |
| :---- |
| *有些教程建议修改 `/etc/profile` 文件，可是修改该文件后，`JAVA_HOME` 依旧是无法出现在 `ssh` 命令模式下的环境中，就导致了还必须同时设置 `conf/hadoop-env.sh` 文件中的 `JAVA_HOME`，否则，将会出现 `Error: JAVA_HOME is not set.` 的故障。可能是写教程的人不熟悉 `/etc/profile` 和 `/etc/environment` 的加载时间所致。这里我们只需修改 `/etc/environment`文件即可。* |

然后，`sudo reboot`，重新启动虚拟机，以让全局环境变量生效。

重启后，我们可以通过显示 hadoop 的版本来判断之前的配置是否正确。

```bash
hadoop version
```

### 3.2.3 配置 `*-site.xml`

之前的配置一直是环境配置，现在才是真正的 Hadoop 相关的配置。对于单节点伪分布模式，配置非常简单，我们只需要修改三个 `*-site.xml` 文件即可。默认情况下，这三个文件都包含一个空的`<configuration>` ，在这里，我们需要为三个文件的`<configuration>`中加入对应的配置。

为方便起见，我们将当前目录换到 `conf` 目录下：

```bash
cd hadoop-1.2.1/conf
```

#### `core-site.xml`:

`nano core-site.xml`

在 `<configuration>`中加入：
```xml
    <property>
        <name>fs.default.name</name>
        <value>hdfs://hadoop-master:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>${user.home}/hadoop/hdfs/tmp</value>
    </property>
```

| Note: |
| :---- |
| *这里如果不设置 `hadoop.tmp.dir`，Hadoop 是可以正常运行的。但是其 HDFS 存储空降则使用的是 `/tmp` 目录，即内存，这显然不符合大容量存储的思想，因此我们需要将其指定到硬盘上。在这里，我们使用了 `${user.home}` 变量，该变量代表的是用户主目录 `$HOME`。<br /><br /> 这个目录需要记住，因为后续排障过程中，可能会需要到这个目录中检查 `current/VERSION` 文件的`namespaceID`以及`storageID`等。*|

#### `hdfs-site.xml`:

`nano hdfs-site.xml`

在 `<configuration>`中加入：

```xml
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
```

#### `mapred-site.xml`:

`nano mapred-site.xml`

在 `<configuration>`中加入：

```xml
    <property>
        <name>mapred.job.tracker</name>
        <value>hadoop-master:9001</value>
    </property>
```

分别保存退出。

### 3.2.4 格式化 NameNode

在配置完成后，我们需要进行一次 NameNode 的格式化。如果未格式化，NameNode 会因 HDFS 所需的数据结构错误而无法启动。

```bash
hadoop namenode -format
```

至此，Hadoop 单节点伪分布模式就配置完毕了。


3.3 验证 Hadoop 已安装正常
------------------

之前已经将 Hadoop 配置完毕，那么接下来我们如何知道 Hadoop 配置正确了呢？首先我们需要确定 Hadoop 能否运行。

### 3.3.1 启动 Hadoop

```bash
start-all.sh
```

这条命令将根据之前配置的 *-site.xml，来启动 Hadoop 云（当然，此时，我们这个云是伪分布，就一个节点）。

然后，我们需要确定节点是否正常运行了。

```bash
jps
```

这条命令可以帮助我们查看当前系统运行的 Java 程序进程。如果配置正确，我们将看到类似下面的结果：

```bash
2613 Jps
2018 NameNode
2388 JobTracker
2143 DataNode
2514 TaskTracker
2290 SecondaryNameNode
```

需要注意的是，单节点伪分布正确运行后，应该会有5个组件运行：`NameNode`、`DataNode`、`SecondaryNameNode`、`JobTracker`、以及`TaskTracker`。缺少任何一个，都说明配置上有所错误。需要回去检查各项配置。

| Note: |
| :---- |
| *在出现故障后向论坛、朋友求助的时候，需要给对方提供你的 `conf` 目录下的配置文件、以及日志，否则对方无法理解你的故障可能的问题。日志目录：`$HOME/hadoop/hadoop-1.2.1/logs`* |

### 3.3.2 运行示例程序

就如同我们搭建好其它语言的编译环境后，会编写一个 `HelloWorld` 程序来测试整个环境是否工作一样，在 Hadoop 中，我们有自己的`HelloWorld`类的程序，叫做`WordCount`。

`ＷordCount` 是对纯文本进行单词出现次数统计的，为了使用 `WordCount`，我们需要先准备一些数据，也就是一些文本文件。这里，我们使用 [古腾堡计划](http://www.gutenberg.org/)中的英文图书。

#### 下载
```bash
mkdir -p ~/hadoop/data/book/download
cd ~/hadoop/data/book/download
wget -r -l 2 -H "http://www.gutenberg.org/robot/harvest?filetypes[]=txt&langs[]=en"

```

这里`-l 2`是只取 2 级，因此并没有下载很多书，大约有200本英文书籍，解压缩后大约 90MB 左右。，如果觉得数据量不够可以酌情增加一些。根据当前的网速，下载需要一段时间。下载完成后，会出现类似于：

```
FINISHED --2014-02-18 19:44:10--
Total wall clock time: 7m 42s
Downloaded: 204 files, 34M in 6m 39s (86.0 KB/s)
```

#### 解压缩

```bash
for f in `find ~/hadoop/data/book/download -name *.zip` ; do unzip -j $f *.txt -d ~/hadoop/data/book/input; done
```

#### 将书籍放到 HDFS 云

```bash
hadoop fs -put ~/hadoop/data/book/input /data/book/input
```

上传到云后，我们可以用过命令 `hadoop fs -ls /data/book/input` 来检查是否都已进入云了。

#### 执行 `WordCount` 示例程序

```bash
hadoop jar ~/hadoop/hadoop-1.2.1/hadoop-examples-1.2.1.jar wordcount /data/book/input /data/book/output
```

| Note: |
| :---- |
| *在执行过程中，可以新开一个 SSH 终端窗口连接到 `hadoop-master`，然后运行命令 `htop` 来观察任务执行期间的节点负载情况，包括内存占用率、CPU占用率、最消耗资源的程序等等。这些信息将来可能会作为云性能优化调整的依据。*|

执行结束后，会输出下面类似的输出：

```
14/02/18 20:26:29 INFO input.FileInputFormat: Total input paths to process : 201
14/02/18 20:26:29 INFO util.NativeCodeLoader: Loaded the native-hadoop library
14/02/18 20:26:29 WARN snappy.LoadSnappy: Snappy native library not loaded
14/02/18 20:26:30 INFO mapred.JobClient: Running job: job_201402190610_0002
14/02/18 20:26:31 INFO mapred.JobClient:  map 0% reduce 0%
14/02/18 20:26:42 INFO mapred.JobClient:  map 1% reduce 0%
14/02/18 20:26:45 INFO mapred.JobClient:  map 2% reduce 0%
...
14/02/18 20:30:32 INFO mapred.JobClient:  map 99% reduce 32%
14/02/18 20:30:34 INFO mapred.JobClient:  map 100% reduce 32%
14/02/18 20:30:40 INFO mapred.JobClient:  map 100% reduce 100%
14/02/18 20:30:41 INFO mapred.JobClient: Job complete: job_201402190610_0002
14/02/18 20:30:41 INFO mapred.JobClient: Counters: 29
14/02/18 20:30:41 INFO mapred.JobClient:   Job Counters 
14/02/18 20:30:41 INFO mapred.JobClient:     Launched reduce tasks=1
14/02/18 20:30:41 INFO mapred.JobClient:     SLOTS_MILLIS_MAPS=451625
14/02/18 20:30:41 INFO mapred.JobClient:     Total time spent by all reduces waiting after reserving slots (ms)=0
14/02/18 20:30:41 INFO mapred.JobClient:     Total time spent by all maps waiting after reserving slots (ms)=0
14/02/18 20:30:41 INFO mapred.JobClient:     Launched map tasks=201
14/02/18 20:30:41 INFO mapred.JobClient:     Data-local map tasks=201
14/02/18 20:30:41 INFO mapred.JobClient:     SLOTS_MILLIS_REDUCES=228055
14/02/18 20:30:41 INFO mapred.JobClient:   File Output Format Counters 
14/02/18 20:30:41 INFO mapred.JobClient:     Bytes Written=8206570
14/02/18 20:30:41 INFO mapred.JobClient:   FileSystemCounters
14/02/18 20:30:41 INFO mapred.JobClient:     FILE_BYTES_READ=41640360
14/02/18 20:30:41 INFO mapred.JobClient:     HDFS_BYTES_READ=92553354
14/02/18 20:30:41 INFO mapred.JobClient:     FILE_BYTES_WRITTEN=90941789
14/02/18 20:30:41 INFO mapred.JobClient:     HDFS_BYTES_WRITTEN=8206570
14/02/18 20:30:41 INFO mapred.JobClient:   File Input Format Counters 
14/02/18 20:30:41 INFO mapred.JobClient:     Bytes Read=92529611
14/02/18 20:30:41 INFO mapred.JobClient:   Map-Reduce Framework
14/02/18 20:30:41 INFO mapred.JobClient:     Map output materialized bytes=37792708
14/02/18 20:30:41 INFO mapred.JobClient:     Map input records=1868098
14/02/18 20:30:41 INFO mapred.JobClient:     Reduce shuffle bytes=37792708
14/02/18 20:30:41 INFO mapred.JobClient:     Spilled Records=5323388
14/02/18 20:30:41 INFO mapred.JobClient:     Map output bytes=151082969
14/02/18 20:30:41 INFO mapred.JobClient:     Total committed heap usage (bytes)=27599167488
14/02/18 20:30:41 INFO mapred.JobClient:     CPU time spent (ms)=251850
14/02/18 20:30:41 INFO mapred.JobClient:     Combine input records=15649835
14/02/18 20:30:41 INFO mapred.JobClient:     SPLIT_RAW_BYTES=23743
14/02/18 20:30:41 INFO mapred.JobClient:     Reduce input records=2528771
14/02/18 20:30:41 INFO mapred.JobClient:     Reduce input groups=561390
14/02/18 20:30:41 INFO mapred.JobClient:     Combine output records=2636459
14/02/18 20:30:41 INFO mapred.JobClient:     Physical memory (bytes) snapshot=36463890432
14/02/18 20:30:41 INFO mapred.JobClient:     Reduce output records=561390
14/02/18 20:30:41 INFO mapred.JobClient:     Virtual memory (bytes) snapshot=218530009088
14/02/18 20:30:41 INFO mapred.JobClient:     Map output records=15542147
```

如果有错误，会有异常报错。执行下面的命令确定正常执行完毕：

```bash
hadoop fs -ls /data/book/output/
```

该命令会列出我们指定的HDFS云中的输出目录内容：

```
Found 3 items
-rw-r--r--   1 tao supergroup          0 2014-02-18 20:30 /data/book/output/_SUCCESS
drwxr-xr-x   - tao supergroup          0 2014-02-18 20:26 /data/book/output/_logs
-rw-r--r--   1 tao supergroup    8206570 2014-02-18 20:30 /data/book/output/part-r-00000
```

注意其中若存在 `_SUCCESS` 文件，即说明执行成功了。统计的结果就在 `part-r-xxxxx` 这类文件里。可以拷贝到本地查看其内容，也可以通过 `http://hadoop-master:50070` 的 Web 界面来查看其内容。

### 3.3.3 停止 Hadoop

我们知道了怎么启动 Hadoop 云；知道了怎么运行 Hadoop 程序；接下来我们需要停止 Hadoop 云：

```bash
stop-all.sh
```

可以通过 `jps` 确定一下确实都关闭了。如果需要关机，则执行 `sudo poweroff` 即可。














