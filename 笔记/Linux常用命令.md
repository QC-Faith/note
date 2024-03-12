# vim（文本编辑器）

退出`vim`的方式：

- 保存退出：

  > 先按`ESC`，再输入冒号，再输入`WQ(写入并退出)`
  >
  > `ESC`后，直接按`shift+zz`

- 正常退出(前提是打开的文本文件在内容上没有被改动过)

  > `ESC`后再输入冒号，再输入`q`

- 不保存退出

  > 先按`ESC`，再输入冒号，再输入`q!`

#压缩

```shell
ar -x fileName.deb		//.deb解压
tar -zxvf fileName.tar.gz		//.tar.gz解压
rpm2cpio fileName.rpm | cpio -div		//.rpm解压
tar -xvJf fileName.tar.xz		//.tar.xz解压
```

# 权限

> chmod 777 /路径/文件名	//chmod 是修改文件的执行属性(所属组,所属者以及其他人所有的权限,比如 读,写,执行)
>     chmod 644 取消只读权限（文件可写）
>     chmod 444 设置文件只读（文件不可写）
> chown -R qiyuesuo:qiyuesuo /opt/qiyuesuo/config/	//chown 是修改文件的所有者(owner),和所属组(group)  需要 root 权限才能执行此命令
>
> **`chmod` 命令**
>
> > `Linux/Unix` 的文件调用权限分为三级 : 文件拥有者、群组、其他。利用 `chmod` 可以控制文件如何被他人所调用。
> >
> > 用于改变 `linux` 系统文件或目录的访问权限。用它控制文件或目录的访问权限。该命令有两种用法。一种是包含字母和操作符表达式的文字设定法；另一种是包含数字的数字设定法。
> >
> > 每一文件或目录的访问权限都有三组，每组用三位表示，分别为文件属主的读、写和执行权限；与属主同组的用户的读、写和执行权限；系统中其他用户的读、写和执行权限。可使用` ls -l test.txt `查找。
>
> **`chown` 命令**
>
> > `chown` 将指定文件的拥有者改为指定的用户或组，用户可以是用户名或者用户 `ID`；组可以是组名或者组 `ID`；文件是以空格分开的要改变权限的文件列表，支持通配符。
> >
> > `-c ` 显示更改的部分的信息
> >
> > `-R` 处理指定目录及子目录 

### chown

> 把`/var/run/httpd.pid`的所有者设置`root`： `chown root /var/run/httpd.pid`
>
> 将文件 `file1.txt`的拥有者设为 `runoob`，群体的使用者 `runoobgroup`: `chown runoob:runoobgroup file1.txt`
>
> 将当前前目录下的所有文件与子目录的拥有者皆设为 `runoob`，群体的使用者 `runoobgroup`: `chown -R runoob:runoobgroup *`
>
> 把 `/home/runoob` 的关联组设置为 `512`（关联组ID），不改变所有者： `chown :512 /home/runoob`

# 模糊匹配

```java
grep -n "COMPATABLE_DBMS" /opt/ShenTong/admin/OSRDB.conf 	// grep -n "查找的字符" 查找的字符所在的文件名   返回对应字符所在行数
grep -i "V\$TEMP_SPACE_HEADER"  *.sql 		// -i 搜索时忽略大小写
```

> ll |grep emb 查找带有emb的
> rm -rf tomcat-embed-* 删除带有tomcat-embed-的

###find

> 将当前目录及其子目录下<font color='Peach'>所有文件后缀为 **.c**</font>的文件列出来: `find . -name "*.c"`
>
> 将当前目录及其子目录中的<font color='Peach'>所有文件</font>列出： `find . -type f`
>
> 将当前目录及其子目录下所有<font color='Peach'>最近 20 天内更新过</font>的文件列出 `find . -ctime -20`
>
> 查找 /var/log 目录中更改时间在 7 日以前的普通文件，并在删除之前询问它们：`find /var/log -type f -mtime +7 -ok rm {} \;`
>
> 查找当前目录中文件属主具有读、写权限，并且文件所属组的用户和其他用户具有读权限的文件：`find . -type f -perm 644 -exec ls -l {} \;`
>
> 查找系统中所有文件长度为 0 的普通文件，并列出它们的完整路径：`find / -type f -size 0 -exec ls -l {} \;`

# 常管理命令

**`cat` 命令**

> 用于连接文件并打印到标准输出设备上。

主要有三大功能：

1. 一次显示整个文件:

```shell
cat filename
```

2. 从键盘创建一个文件:

```shell
cat > filename	///只能创建新文件，不能编辑已有文件。
```

3. 将几个文件`合并`为一个文件，会覆盖源文件:

```shell
cat file1 file2 > file	
    ///  -b 对非空输出行号
	///  -n 输出所有行号
```

4. 多个文件`追加`到一个文件，末尾追加不会覆盖

~~~shell
cat file1 file2 >> file 
cat test1.log >> test2.log
~~~

5. 追加输入内容到file，直到`EOF`为止，`EOF`为结束标记不会填充到`file`中

~~~shell
cat >> file << EOF  
~~~

**`cp` 命令**

> 将源文件复制至目标文件，或将多个源文件复制至目标目录。
>
> 注意：命令行复制，如果目标文件已经存在会提示是否覆盖，而在 `shell` 脚本中，如果不加` -i `参数，则不会提示，而是直接覆盖！

```
-i 提示
-r 复制目录及目录内所有项目
-a 复制的文件与原文件时间一样
```

**`find` 命令**

> 用于在文件树中查找文件，并作出相应的处理。

命令格式：

```java
find pathname -options [-print -exec -ok ...]
```

命令参数：

```java
pathname: find命令所查找的目录路径。例如用.来表示当前目录，用/来表示系统根目录。
-print： find命令将匹配的文件输出到标准输出。
-exec： find命令对匹配的文件执行该参数所给出的shell命令。相应命令的形式为'command' {  } \;，注意{   }和\；之间的空格。
-ok： 和-exec的作用相同，只不过以一种更为安全的模式来执行该参数所给出的shell命令，在执行每一个命令之前，都会给出提示，让用户来确定是否执行。
```

命令选项：

```java
-name 按照文件名查找文件
-perm 按文件权限查找文件
-user 按文件属主查找文件
-group  按照文件所属的组来查找文件。
-type  查找某一类型的文件，诸如：
   b - 块设备文件
   d - 目录
   c - 字符设备文件
   l - 符号链接文件
   p - 管道文件
   f - 普通文件
```

**`head` 命令**

> `head` 用来显示档案的开头至标准输出中，默认 `head` 命令打印其相应文件的开头 `10` 行。

**常用参数**：

```
-n<行数> 显示的行数（行数为复数表示从最后向前数）
```

**`less` 命令**

> `less` 与 `more` 类似，但使用 `less` 可以随意浏览文件，而 `more` 仅能向前移动，却不能向后移动，而且 less 在查看之前不会加载整个文件。

**``more` 命令**

> 功能类似于` cat, more` 会以一页一页的显示方便使用者逐页阅读，而最基本的指令就是按空白键`（space）`就往下一页显示，按 `b` 键就会往回`（back）`一页显示。

**`ln` 命令**

> 功能是为文件在另外一个位置建立一个同步的链接，当在不同目录需要该问题时，就不需要为每一个目录创建同样的文件，通过 `ln` 创建的链接`（link）`减少磁盘占用量。

分类：软链接、硬链接

软链接：

1. 软链接，以路径的形式存在。类似于Windows操作系统中的快捷方式

2. 软链接可以跨文件系统 ，硬链接不可以

3. 软链接可以对一个不存在的文件名进行链接

4. 软链接可以对目录进行链接

硬链接:

1. 硬链接，以文件副本的形式存在。但不占用实际空间。

2. 不允许给目录创建硬链接

3. 硬链接只有在同一个文件系统中才能创建

**注意：**

1. `ln`命令会保持每一处链接文件的同步性，也就是说，不论你改动了哪一处，其它的文件都会发生相同的变化；

2. `ln`的链接又分软链接和硬链接两种，软链接就是`ln –s `源文件 目标文件，它只会在你选定的位置上生成一个文件的镜像，不会占用磁盘空间，硬链接 `ln` 源文件 目标文件，没有参数`-s`， 它会在你选定的位置上生成一个和源文件大小相同的文件，无论是软链接还是硬链接，文件都保持同步变化。

3. `ln`指令用在链接文件或目录，如同时指定两个以上的文件或目录，且最后的目的地是一个已经存在的目录，则会把前面指定的所有文件或目录复制到该目录中。若同时指定多个文件或目录，且最后的目的地并非是一个已存在的目录，则会出现错误信息。

**`locate` 命令**

> `locate` 通过搜寻系统内建文档数据库达到快速找到档案，数据库由 `updatedb` 程序来更新，`updatedb` 是由 `cron daemon` 周期性调用的。默认情况下 `locate` 命令在搜寻数据库时比由整个由硬盘资料来搜寻资料来得快，但较差劲的是 `locate` 所找到的档案若是最近才建立或 刚更名的，可能会找不到，在内定值中，`updatedb` 每天会跑一次，可以由修改 `crontab` 来更新设定值` (etc/crontab)`。

`locate` 与 `find` 命令相似，可以使用如` *、? `等进行正则匹配查找

**`mv`命令**

> 移动文件或修改文件名，根据第二参数类型（如目录，则移动文件；如为文件则重命令该文件）。
>
> 当第二个参数为目录时，第一个参数可以是多个以空格分隔的文件或目录，然后移动第一个参数指定的多个文件到第二个参数指定的目录中。

**`rm`命令**

> 删除一个目录中的一个或多个文件或目录，如果没有使用` -r `选项，则` rm `不会删除目录。如果使用` rm `来删除文件，通常仍可以将该文件恢复原状。

**`tail`命令**

> 用于显示指定文件末尾内容，不指定文件时，作为输入信息进行处理。常用查看日志文件。

**`vim`命令**

> `Vim`是从 `vi` 发展出来的一个文本编辑器。代码补完、编译及错误跳转等方便编程的功能特别丰富，在程序员中被广泛使用。

打开文件并跳到第 `10` 行：`vim +10 filename.txt` 。
打开文件跳到第一个匹配的行：`vim +/search-term filename.txt` 。
以只读模式打开文件：`vim -R /etc/passwd `。
基本上 `vi/vim` 共分为三种模式，分别是命令模式`（Command mode）`，输入模式`（Insert mode）`和底线命令模式`（Last line mode）`。



> `which`     	查看可执行文件的位置。 
>
> `whereis`  	查看文件的位置。 
>
> `locate`        配合数据库查看文件位置。 
>
> `find`           实际搜寻硬盘查询文件名称。

## 文档编辑命令

**`grep` 命令**

> 全局正则表达式搜索。

`grep` 的工作方式:  它在一个或多个文件中搜索字符串模板。如果模板包括空格，则必须被引用，模板后的所有字符串被看作文件名。搜索的结果被送到标准输出，不影响原文件内容。

**`wc`命令**

`wc(word count)`功能为统计指定的文件中字节数、字数、行数，并将统计结果输出

## 磁盘管理命令

**`cd`命令**

> 切换当前目录

**`df`命令**

> 显示磁盘空间使用情况。获取硬盘被占用了多少空间，目前还剩下多少空间等信息
>
> 如果没有文件名被指定，则所有当前被挂载的文件系统的可用空间将被显示。默认情况下，磁盘空间将以 `1KB` 为单位进行显示，除非环境变量` POSIXLY_CORRECT `被指定，那样将以`512`字节为单位进行显示

**`du`命令**

> `du` 命令也是查看使用空间的，但是与` df `命令不同的是` Linux du `命令是对文件和目录磁盘使用的空间的查看

**`ls`命令**

>  `list` 的缩写，通过 `ls` 命令不仅可以查看 `linux` 文件夹包含的文件，而且可以查看文件权限(包括目录、文件夹、文件权限)查看目录信息等等。

**`mkdir`命令**

> `mkdir` 命令用于创建文件夹。

**`pwd`命令**

> 查看当前工作目录路径

**`rmdir`命令**

> 从一个目录中删除一个或多个子目录项，删除某目录时也必须具有对其父目录的写权限。
>
> **注意**：不能删除非空目录

## 网络通讯命令

**`ifconfig` 命令**

> 用于查看和配置 `Linux` 系统的网络接口。
> 查看所有网络接口及其状态：`ifconfig -a `。
> 使用 `up` 和 `down` 命令启动或停止某个接口：`ifconfig eth0 up 和 ifconfig eth0 down` 。

**`iptables` 命令**

> 配置 `Linux` 内核防火墙的命令行工具。
>
> 使用 `iptables save `命令，进行保存。否则，服务器重启后，配置的规则将丢失。

**`netstat `命令**

> 命令用于显示网络状态。

**`ping`命令**

> 用于检测主机。
>
> 执行`ping`指令会使用`ICMP`传输协议，发出要求回应的信息，若远端主机的网络功能没有问题，就会回应该信息，因而得知该主机运作正常。

## 系统管理命令

**`date` 命令**

> 显示或设定系统的日期与时间。

**`free`命令**

> 显示系统内存使用情况，包括物理内存、交互区内存`(swap)`和内核缓冲区内存。

**`kill` 命令**

> 发送指定的信号到相应进程。不指定型号将发送`SIGTERM（15）`终止指定进程。如果任无法终止该程序可用`"-KILL"` 参数，其发送的信号为`SIGKILL(9) `，将强制结束进程，使用`ps`命令或者`jobs` 命令可以查看进程号。`root`用户将影响用户的进程，非`root`用户只能影响自己的进程。

# 防火墙与端口

[(34条消息) Linux防火墙firewall只允许特定ip访问_Pert-的博客-CSDN博客_linux防火墙只允许特定ip访问](https://blog.csdn.net/s_frozen/article/details/120636667?spm=1001.2101.3001.6650.5&utm_medium=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromBaidu~Rate-5-120636667-blog-116776227.pc_relevant_multi_platform_whitelistv4&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromBaidu~Rate-5-120636667-blog-116776227.pc_relevant_multi_platform_whitelistv4)

~~~markdown
firewall-cmd --query-port=6379/tcp		
# 正常情况应该提示yes或者no，意为防火墙是否放行此端口，若提示firewall is not running则是firewall防火墙被关闭，此命令打开systemctl start firewalld.service

firewall-cmd --add-port=15563/tcp
# 放行6379端口

firewall-cmd --state
# 查看防火墙状态

# 查看所有开放的端口
firewall-cmd --zone=public --list-ports
~~~

# Redis

~~~markdowm
15563
Scf*1997#0525@

配置文件位置：/usr/local/redis/etc

redis系统服务文件：/etc/systemd/system/redis.service
修改后使生效：systemctl daemon-reload

启动查看与停止

systemctl status redis
systemctl stop redis

开机自启
systemctl enable redis

查看是否开机自启动
systemctl list-unit-files | grep redis
~~~



# 直接执行方法

~~~markdown
去除包名（package com.xxx.xxx）
# 不依赖外部jar
javac Test.java
java Test

# 依赖外部jar（例如依赖Test.java当前目录下的postgresql-42.4.2.jar包）
javac Test.java
java -cp $CLASSPATH:postgresql-42.4.2.jar:Test Test
~~~



#  安全防护

## 定时任务

[centos7设置服务为开机自启动（以crond.serivce为例）_Mr.Xiong`s 运维日志的技术博客_51CTO博客](https://blog.51cto.com/mrxiong2017/2084790)

~~~shell
# 1：查看crond.serivce服务的自启动状态
[root@localhost ~]# systemctl is-enabled crond.service
disabled  #此时crond.serivce的自启动状态为disabled

# 2：开启crond.serivce服务自启动

[root@localhost ~]# systemctl enable crond.service

[root@localhost ~]# systemctl is-enabled crond.service
enabled

# 列出所有的启动文件：
systemctl list-unit-files

# 列出所有状态为enable的启动文件
systemctl list-unit-files | grep enable

# 关闭crond.serivce的自启动状态
systemctl disable crond.service

# 开启和关闭crond.service服务
# 1：查看crond.service的启动状态
systemctl status crond.service

# 开启crond.service服务命令
systemctl start crond.service

# 停止crond.service服务命令
systemctl stop crond.service
~~~

## 允许登录

[ssh访问控制，多次失败登录即封掉IP，防止暴力破解_AlvesWeiDong的博客-CSDN博客](https://blog.csdn.net/weixin_38628533/article/details/80572531)

读取/var/log/secure，查找关键字 Failed，**（#cat /var/log/secure | grep Failed）**

```shell
# 1.先把始终允许的IP填入 /etc/hosts.allow ，这很重要！比如：
sshd:19.16.18.1:allow
sshd:19.16.18.2:allow
# 2./etc/ssh/sshd_config配置文件中设置AllowUsers选项
然后重启SSH
service sshd restart
```

~~~shell
 vim /usr/local/bin/secure_ssh.sh
~~~

~~~shell
#! /bin/bash
cat /var/log/secure|awk '/Failed/{print $(NF-3)}'|sort|uniq -c|awk '{print $2"="$1;}' > /usr/local/bin/black.list
for i in `cat  /usr/local/bin/black.list`
do
  IP=`echo $i |awk -F= '{print $1}'`
  NUM=`echo $i|awk -F= '{print $2}'`
  if [ $NUM -gt 1 ]; then
    grep $IP /etc/hosts.deny > /dev/null
    if [ $? -gt 0 ];then
      echo "sshd:$IP:deny" >> /etc/hosts.deny
    fi
  fi
done
~~~

将`secure_ssh.sh`脚本放入`cron`计划任务，每1分钟执行一次。**添加定时任务**`vim /etc/crontab`

~~~shell
*/1 * * * *  sh /usr/local/bin/secure_ssh.sh
~~~

```shell
# 查看自定义黑名单
cat /usr/local/bin/black.list
# 查看服务器黑名单
cat /etc/hosts.deny
```

# 限定登录IP

~~~markdown
/etc/hosts.allow
/etc/hosts.deny

/etc/ssh/sshd_config配置文件中设置AllowUsers选项
然后重启SSH
service sshd restart
~~~

# JDK安装

1. 上传安装包至`opt`目录（也可以是别的目录）

2. 执行以下命令将`jdk`安装在`/usr/local/java`目录下

   ```bash
   cd /usr/local && mkdir java    
   cp /opt/jdk-8u191-linux-x64.tar.gz ./java
   cd java/
   tar -zxvf jdk-8u191-linux-x64.tar.gz
   rm -rf jdk-8u191-linux-x64.tar.gz
   ```

### 配置环境变量

1. 编辑环境变量配置文件：`vim /etc/profile`
2. 在文件末尾添加如下内容(JAVA_HOME为安装jdk的路径)

```bash
export JAVA_HOME=/usr/local/java/jdk1.8.0_191
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$JAVA_HOME/bin:$PATH
```

保存并退出后使用`source /etc/profile`使配置生效

<font color='Apricot'>**如果需要切换用户，使用 `su - 用户名` 将环境带过去即可**</font>

```shell
修改当前用户目录下的 .bash_profile 文件
(1) 首先，通过指令 cd ~ 进入到当前用户所在的文件夹下。
(2) 然后，通过指令 vi .bash_profile 用vim编辑器打开 .bash_profile文件，进入后，按键盘上的【A】键或【i】键进入vim编辑器的编辑状态，在文件最尾部加入JDK环境变量的配置，如下所示：
export JAVA_HOME=/home/openam_jxdoe/jdk1.7.0_80
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JRE_HOME=$JAVA_HOME/jre
其中，PATH和CLASSPATH后面的值不需要改变，只需要修改JAVA_HOME后面的值即可，用你的Java JDK解压安装的位置（就是你上面放软件的位置）代替 /home/openam_jxdoe/jdk1.7.0_80 即可。
(3) 修改完后，按键盘上的【Esc】键退出vim编辑器的编辑状态，然后键盘输入指令 ：wq 保存并退出vim编辑器。
(4) 最后，通过指令 source .bash_profile 使Linux应用你刚配置好的 .bash_profile 文件，Java JDK 在Linux上便配置好了。
(5) 通过命令 java -version 查看你目前的版本
更新之后的版本需要重开窗口才可以是新的版本
```

