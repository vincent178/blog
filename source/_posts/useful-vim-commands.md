title: 好用的 vim 命令
date: 2015-10-16 09:59:54
tags:
thumbnailImage: http://res.cloudinary.com/dudqzxsbb/image/upload/v1444960834/Vim-editor_logo_avcvg7.png
---

打开的文本中插入一个1到100的序列 <br />
`:r!seq 100`

当前的每一行文字前面增加「序号. 」 <br />
`:let i=1 | g /^/ s//\=i.". "/ | let i+=1`

当前目录下（包括子文件夹）所有后缀为 java 的文件中的 apache 替换成 eclipse <br />
```
vim
:n **/*.java
:argdo %s/apache/eclipse/ge | update 
```

