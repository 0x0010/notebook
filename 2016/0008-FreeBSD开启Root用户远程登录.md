新安装的FreeBSD系统默认关闭了Root用户的远程登录（ssh）。

可以通过修改SSH的配置文件，打开这个限制。

编辑文件<code>/etc/ssh/sshd_config</code>，找到Authentication
````
# Authentication:

#LoginGraceTime 2m
PermitRootLogin yes
#StrictModes yes
#MaxAuthTries 6
#MaxSessions 10
````
将PermitRootLogin修改为yes，别忘了将这行最前边的＃号去掉。
