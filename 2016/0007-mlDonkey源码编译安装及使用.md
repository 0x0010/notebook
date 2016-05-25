网络上的文章包括官方的文档里都没有对mlDonkey的快速安装及使用进行详细描述，或许是我自己没看仔细。我这里会尽量详细地描述整个安装过程和如何快速上手使用。本文使用的Linux发行版本是CentOS 6.7，不同发行版本的差异应该不大。如果按照此教程还无法安装使用mlDonkey，可以在文章评论里描述你遇到的问题，我们一起讨论。

[mlDonkey官方网址](http://mldonkey.sourceforge.net/Main_Page)，我使用的版本是2014年3月份发布的3.1.5版。[点击下载mlDonkey-3.1.5]( https://sourceforge.net/projects/mldonkey/files/mldonkey/3.1.5/mldonkey-3.1.5.tar.bz2/download)

### 编译环境准备
下载的package是tar.bz2的压缩格式。解压这种格式的压缩包依赖bzip2，可以通过<code>yum install bzip2</code>安装bzip2。
进入刚才下载的源码包目录下，使用<code>tar -xvjf mldonkey-3.1.5</code>将源码释放到当前</code>./mldonkey-3.1.5</code>。
<code>cd mldonkey-3.1.5</code>进入源码目录。使用命令<code>./configure</code>进行编译环境检查。根据提示安装缺少的包，在我的安装过程中没有非常特殊并且难处理的依赖。
唯一需要特别说明的是，当检查OCaml编译器的时候不要敲y继续，这个时候要断下来，手动安装更高版本的编译器。
使用<code>yum install ocaml-camlp4-devel</code>安装ocaml-camlp4-devel，这个是踩过坑得出的结论，同时yum会安装两个依赖：ocaml和rpm-build。安装好之后再次使用<code>./configure</code>命令，不出意外，我们可以看到configure成功之后的提示Configuring MLDonkey 3.1.5 completed.
### 编译安装
编译安装比较简单，使用<code>make</code>命令编译，使用<code>make install</code>命令安装。默认情况，二进制文件会放在/usr/local/bin目录。
````
lrwxrwxrwx. 1 root root    5 May 25 09:41 mlbt -> mlnet
lrwxrwxrwx. 1 root root    5 May 25 09:41 mldc -> mlnet
lrwxrwxrwx. 1 root root    5 May 25 09:41 mldonkey -> mlnet
lrwxrwxrwx. 1 root root    5 May 25 09:41 mlgnut -> mlnet
-rwxr-xr-x. 1 root root 8.4M May 25 09:41 mlnet
lrwxrwxrwx. 1 root root    5 May 25 09:41 mlslsk -> mlnet
````
至此，我们编译安装就结束了。接下来将会介绍如何使用这个工具进行下载。

### mlDonkey的使用
