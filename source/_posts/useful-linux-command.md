title: 有用的Linux命令
date: 2015-10-13 06:23:28
tags:
coverImage: http://res.cloudinary.com/dudqzxsbb/image/upload/v1444696477/linux-logo_xntobe.png
thumbnailImage: http://res.cloudinary.com/dudqzxsbb/image/upload/v1444696566/littletux_bumper.sh-600x600_ywmelk.png
---


### dd 命令
“dd”命令代表了转换和复制文件。可以用来转换和复制文件，大多数时间是用来复制iso文件(或任何其它文件)到一个usb设备(或任何其它地方)中去，所以可以用来制作USB启动器。<br />
```
$ dd if=/home/user/Downloads/debian.iso of=/dev/sdb1 bs=512M; sync
```

### uname 命令
"uname"命令就是Unix Name的简写。显示机器名，操作系统和内核的详细信息。<br />
```
$ uname -a
```

### history 命令
“history”命令就是历史记录。它显示了在终端中所执行过的所有命令的历史。 <br />
```
$ history
```

### chmod 命令
“chmod”命令就是改变文件的模式位。chmod会根据要求的模式来改变每个所给的文件，文件夹，脚本等等的文件模式（权限）。 <br />
* Read (r)=4
* Write(w)=2
* Execute(x)=1 <br />

```
$ chmod 777 abc.sh
```

### chown 命令
“chown”命令就是改变文件拥有者和所在用户组。每个文件都属于一个用户组和一个用户。
```
$ chown server:server Binary
```

### tar 命令
“tar”命令是磁带归档(Tape Archive)，对创建一些文件的的归档和它们的解压很有用。
```
$ tar -zxvf abc.tar.gz (Remember 'z' for .tar.gz)
$ tar -jxvf abc.tar.bz2 (Remember 'j' for .tar.bz2)
$ tar -cvf archieve.tar.gz(.bz2) /path/to/folder/abc
```

### cal 命令
```
$ cal 07 2145 
```

### date 命令
```
$ date --set='14 may 2013 13:57'
```

### adduser 命令
```
$ adduser <username>
$ adduser <username> sudo
```

### groups 命令
```
$ groups
$ groups <username>
```
