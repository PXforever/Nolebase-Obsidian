---
share: "true"
---

# Service

> service是由

# 编写一个简单服务

1. 首先在`/home/linaro`下创建一个脚本`demo.sh`，如下：

```shell
#!/bin/bash

function do_something()
{
	echo "Hello World!"
}

function do_exit()
{
	echo "exit service!"
}

function do_restart()
{
	echo "restart service"
	do_something
}

case $1 in
	run|RUN)
		do_something
	;;
	stop|STOP)
		do_exit
	;;
	restart|RESTART)
		do_restart
	;;
	*)
		do_something
esac
exit 0

```

2. 为赋予权限:

```shell
chmod a+x demo.sh
```

3. 在`lib/system/system`或`/etc/systemd/system`编写一个服务`demo.service`:

```shell
[Unit]
Description=Demo service
#Requires=
#After=

[Service]
Type=oneshot
RuntimeDirectory=/home/linaro
#EnvironmentFile=
User=linaro
Group=linaro
PermissionsStartOnly=true
#ExecStartPre=
ExecStart=/home/linaro/demo.sh start
ExecStop=/home/linaro/demo.sh stop
Restart=0

[Install]
WantedBy=multi-user.target
```

4. 赋予权限

```shell
sudo chmod a+x demo.service
```

5. 在目录创建软链接

```shell
sudo ln -s demo.service  /etc/systemd/system/multi-user.target.wants/demo.service
```

6. 使能

```shell
systemctl enable demo
```

7. 使用命令：

```shell
# 因为改过服务文件，所以需要重新加载
systemctl daemon-reload
```

8. 执行服务

```shell
# 查看其状态
systemctl staatus demo

# 执行
systemctl start demo

# 重启
systemctl restart demo

# 停止
systemctl stop demo
```

