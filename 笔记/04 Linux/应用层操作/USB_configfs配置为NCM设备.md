---
share: "true"
---

# NCM 设备配置例程

+ `kernel`开启如下配置,使用`make menuconfig`：

```shell
-> Device Drivers
  │       -> USB support (USB_SUPPORT [=y])
  │         -> USB Gadget Support (USB_GADGET [=y])
  │ (1)       -> USB Gadget functions configurable through configfs (USB_CONFIGFS [=y])
  				USB_CONFIGFS_NCM [=y]
```

开启NCM的`configfs`功能

```shell
# 上面开启实际上就可以启动了，但是有些可能没有该选项可以打开下面
USB_F_NCM=y #这个会编译f_nucm.c文件，也是就是USB-NCM 的功能会被注册到USB configfs列表中
USB_G_NCM=y
```

+ 开机后，配置`configfs`

```shell
USB_ATTRIBUTE=0x409
USB_GROUP=rockchip

CONFIGFS_DIR=/sys/kernel/config
USB_CONFIGFS_DIR=${CONFIGFS_DIR}/usb_gadget/${USB_GROUP}
USB_STRINGS_DIR=${USB_CONFIGFS_DIR}/strings/${USB_ATTRIBUTE}
USB_FUNCTIONS_DIR=${USB_CONFIGFS_DIR}/functions
USB_CONFIGS_DIR=${USB_CONFIGFS_DIR}/configs/${USB_SKELETON}

echo "Debug: configfs_init"
mkdir /dev/usb-ffs

mount -t configfs none ${CONFIGFS_DIR}
mkdir ${USB_CONFIGFS_DIR} -m 0770
echo 0x2207 > ${USB_CONFIGFS_DIR}/idVendor
echo 0x0310 > ${USB_CONFIGFS_DIR}/bcdDevice
echo 0x0200 > ${USB_CONFIGFS_DIR}/bcdUSB
mkdir ${USB_STRINGS_DIR}   -m 0770
SERIAL=`cat /proc/cpuinfo | grep Serial | awk '{print $3}'`
if [ -z $SERIAL ];then
SERIAL=0123456789ABCDEF
fi
echo $SERIAL > ${USB_STRINGS_DIR}/serialnumber
echo "rockchip"  > ${USB_STRINGS_DIR}/manufacturer
echo "rk3xxx"  > ${USB_STRINGS_DIR}/product
```

+ 生成function

```shell

cd ${USB_FUNCTIONS_DIR}
mkdir ncm.gs0 #输入ncm.*驱动就回去func list寻找是否有注册，没有启动功能则会返回无该文件(No such file or directory)
```

+ 创建USB实例

```shell
USB_SKELETON=b.1
mkdir ${USB_CONFIGS_DIR}  -m 0770
mkdir ${USB_CONFIGS_DIR}/strings/${USB_ATTRIBUTE}  -m 0770
```

+ 其它

```shell
echo 0x1 > ${USB_CONFIGFS_DIR}/os_desc/b_vendor_code
echo "MSFT100" > ${USB_CONFIGFS_DIR}/os_desc/qw_sign
echo 500 > ${USB_CONFIGS_DIR}/MaxPower
ln -s ${USB_CONFIGS_DIR} ${USB_CONFIGFS_DIR}/os_desc/b.1
```

+ 绑定`USB function`到`USB`实例(`config`目录下的`b.1`)

```shell
ln -s ${USB_FUNCTIONS_DIR}/ncm.gs0 ${USB_CONFIGS_DIR}/f1 #f*,usb实例可以挂载多个功能实现
```

+ `USB CDC subClass`的专有配置

```shell
# 这部分是NCM特有的配置，如果HID，会有报告描述符

```

+ 启动USB

```shell
ls /sys/class/udc/| awk '{print $1}'
# ffd00000.dwc3
echo ffd00000.dwc3 > ${USB_CONFIGFS_DIR}/UDC	
```

# 实际操作

> 在正点原子的`rv1126`的源码中（或者说可能是rk的源码就有），有实现`USB gadget configfs`。文件位置在：

```shell
/etc/init.d/S50usbdevice
```

**有`S50usbdevice-注释版`在目录下**

修改部分位置即可实现`CDC NCM`，修改如下：

```shell
#!/bin/sh
# .................
HID_EN=off
NCM_EN=off # add here

# .................

function_init()
{
	# .................
	mkdir ${USB_FUNCTIONS_DIR}/hid.usb0
	#add by PX
	mkdir ${USB_FUNCTIONS_DIR}/ncm.gs0
}

# .................

parameter_init()
{
			# .................
			
			usb_hid_en)
				HID_EN=on
				make_config_string hid
				;;
			usb_ncm_en) # add here
				NCM_EN=on
				make_config_string ncm
				;;
			*)
				parse_parameter ${line}
				;;
		esac
	done < $USB_CONFIG_FILE

	case "$CONFIG_STRING" in
		ums)
			PID=0x0000
			;;
			
		# .................
		
		ncm) # add here
			PID=0x0005
			;;
		*)
			PID=0x0019
	esac
}

pre_run_binary()
{
	# .................

	# Add uvc app here with start-stop-daemon
	
	#add NCM usb0 ip setting # add here
	#if [ $NCM_EN = on ];then
		#sleep 10 &&
		#ifconfig usb0 192.168.0.201 up &&
		#echo "USB(NCM) Ehtener usccessful UP"
	#fi
}

# .................

syslink_function()
{
	echo "[USB configfs]link $1 to ${USB_CONFIGS_DIR}/f${USB_FUNCTIONS_CNT}"
	ln -s ${USB_FUNCTIONS_DIR}/$1 ${USB_CONFIGS_DIR}/f${USB_FUNCTIONS_CNT}
	let USB_FUNCTIONS_CNT=USB_FUNCTIONS_CNT+1
}

bind_functions()
{
	# .................

	if [ $UMS_EN = on ];then
		echo ${UMS_RO} > ${USB_FUNCTIONS_DIR}/mass_storage.0/lun.0/ro
		# .................
	fi
	
	# add here
	if [ $NCM_EN = on ];then
		echo 0xEF > ${USB_CONFIGFS_DIR}/bDeviceClass
		echo 0x02 > ${USB_CONFIGFS_DIR}/bDeviceSubClass
		echo 0x01 > ${USB_CONFIGFS_DIR}/bDeviceProtocol
		syslink_function ncm.gs0
	fi

	echo ${CONFIG_STRING} > ${USB_CONFIGS_DIR}/strings/${USB_ATTRIBUTE}/configuration
}
```

接着利用该脚本读取配置文件`/etc/init.d/.usb_config`来配置，那么可以修改为:

```shell
usb_ncm_en
#之前是usb_hid_en，但考虑该脚本不会对注释处理，并且暂时不需要ADB，那么就只需要留上面一行即可。
```

配置完成后开机即可，在开机后，命令行输入:

```shell
ifconfig -a
#usb0      Link encap:Ethernet  HWaddr 4E:07:E0:F8:EE:55
#          BROADCAST MULTICAST  MTU:1500  Metric:1
#          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
#          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
#          collisions:0 txqueuelen:1000
#          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
ifconfig usb0 192.168.0.201 netmask 255.255.255.0 up
# 手动设置USB网卡的ip地址

```

在windows端一般会产生一个网卡，并且可以看到`CDC NCM`，有些电脑没有，比如我是用的`windows 7`就无法弹出网卡信息。这边让该`USB`接入到虚拟机，在ubuntu可以看到：

```shell
ifconfig -a
#enxaa9a8479749b: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
#        ether aa:9a:84:79:74:9b  txqueuelen 1000  (以太网)
#        RX packets 0  bytes 0 (0.0 B)
#        RX errors 0  dropped 0  overruns 0  frame 0
#        TX packets 0  bytes 0 (0.0 B)
#        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# 手动或者命令行配置USB网卡
# 手动，在图形界面网络设置可以看到一个USB网卡，手动设置
# 代码配置
ifconfig enxaa9a8479749b 192.168.0.202 netmask 255.255.255.0
```

上面手动配置会报错，所以还是使用界面配置。

```shell
alientek@ubuntu:~$ ifconfig enxaa9a8479749b 192.168.0.202 netmask 255.255.255.0
SIOCSIFADDR: 不允许的操作
SIOCSIFFLAGS: 不允许的操作
SIOCSIFNETMASK: 不允许的操作
```

## 测试

+ `Host`向`Rv1126`ping

```shell
# 主机端
ping -I enxaa9a8479749b 192.168.0.201
# 测试需要拔掉rv1126与主机连接的网络接口可ping通，猜测可能是rv1126的数据路由默认从rj45的网卡进出数据
```

修改Rv1126下的路由表：

```shell

# 使得所有数据即从RJ45端的网卡出，也从USB CDC_NCM出。所有RJ45接收的数据都会发到USB网卡
# 也就是将两个网卡桥接
```

或者

```shell
# rv1126端
ifconfig usb0 192.168.5.201 netmask 255.255.255.0 up

# PC端配置如下：
ipv4 = 192.168.0.202
netmask = 255.255.255.0
gateway = 192.168.5.1
```

这样就不需要拔线即可ping通。

## 网络压力测试(网速极限)

> 使用`iperf3`指令可以实现网络压力测试，这里在虚拟机和rv1126都使用该指令来搭建服务端和客户端来测试。
>
> `iperf3`默认服务端口是`5201`

### 服务端简单搭建

```shell
# 简单搭建,在PC端运行
iperf3 -s
# 指定端口
iperf3 -s -p 8088
# 指定网卡
iperf3 -s -B enxaa9a8479749b
```

### 客户端简单执行

```shell
# 简单测试,后面的ip地址是服务器的
iperf3 -c 192.168.1.1
# 多个并行数据,这里设置3个并行数据流
iperf3 -c <server ip> -P 3
# 反向测试，服务器发送，或者说测试上行速度
iperf3 -c <server ip> -R
# 双向测试
iperf3 -c <server ip> --bidir
```

### 实际使用示例

#### 示例1

```shell
# 配置：Host-PC：192.168.5.202，Device-Rv1126:192.168.5.201

# 服务端
iperf3 -s

# 客户端
iperf3 -c 192.168.5.202 -b 1000M -t 60 -d
# -B 绑定指定网卡来测试
# -c 指定为客户端
# -b 表示使用的测试带宽
# -t 表示以时间为测试结束条件，默认10s，这里是60s
# -d 打印更详细的debug信息
```

**<font color=yellow>结果</font>**

```shell
# 服务端
ccepted connection from 192.168.5.201, port 55182
[  5] local 192.168.5.202 port 5201 connected to 192.168.5.201 port 55184
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  40.9 MBytes   343 Mbits/sec                  
# ....................................................              
[  5]  60.00-60.04  sec  1.74 MBytes   360 Mbits/sec                  
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-60.04  sec  2.53 GBytes   362 Mbits/sec                  receiver

# 客户端
send_parameters:
{
	"tcp":	true,
	"omit":	0,
	"time":	60,
	"parallel":	1,
	"len":	131072,
	"bandwidth":	1000000000,
	"pacing_timer":	1000,
	"client_version":	"3.6"
}
Connecting to host 192.168.5.202, port 5201
SNDBUF is 16384, expecting 0
RCVBUF is 131072, expecting 0
Setting application pacing to 125000000
Congestion algorithm is cubic
[  5] local 192.168.5.201 port 55184 connected to 192.168.5.202 port 5201
sent 46336 bytes of 131072, total 46336
# ....................................................   
sent 131072 bytes of 131072, total 2714459072
tcpi_snd_cwnd 437 tcpi_snd_mss 1448 tcpi_rtt 6802
send_results
{
	"cpu_util_total":	14.186830199361872,
	"cpu_util_user":	1.3845652033007685,
	"cpu_util_system":	12.802264996061103,
	"sender_has_retransmits":	1,
	"congestion_used":	"cubic",
	"streams":	[{
			"id":	1,
			"bytes":	2714459072,
			"retransmits":	0,
			"jitter":	0,
			"errors":	0,
			"packets":	0,
			"start_time":	0,
			"end_time":	60.0003999999999
		}]
}
get_results
{
	"cpu_util_total":	8.3781607596911,
	"cpu_util_user":	0.19148771178803953,
	"cpu_util_system":	8.1866743458520457,
	"sender_has_retransmits":	-1,
	"congestion_used":	"cubic",
	"streams":	[{
			"id":	1,
			"bytes":	2713331904,
			"retransmits":	-1,
			"jitter":	0,
			"errors":	0,
			"packets":	0,
			"start_time":	0,
			"end_time":	60.040737
		}]
}
interval_len 60.000400 bytes_transferred 2714459072
interval forces keep
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-60.00  sec  2.53 GBytes   362 Mbits/sec    0    618 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-60.00  sec  2.53 GBytes   362 Mbits/sec    0             sender
[  5]   0.00-60.04  sec  2.53 GBytes   362 Mbits/sec                  receiver

```

`Interval`: 时间段，表示发送数据的时间段在哪儿

`Transfer`: 传送数据量

`Bitrate`: 速率

`Retr`: ??

### 示例2

```shell
# 反向测试
# 配置：Host-PC：192.168.5.202，Device-Rv1126:192.168.5.201

# 服务端
iperf3 -s

# 客户端
iperf3 -c 192.168.5.202 -b 1000M -t 60 -d -R
```



### 示例3

```shell
# UDP测试
# 配置：Host-PC：192.168.5.202，Device-Rv1126:192.168.5.201

# 服务端
iperf3 -s

# 客户端
iperf3 -u -c 192.168.5.202 -b 1000M -t 60 -d
# 或者
iperf3 -u -c 192.168.5.202 -b 1000M -t 60 -d -R
```

### 示例4

```shell
# 双向测试
iperf3 -u -c 192.168.5.202 -b 1000M -t 10 --bidir #开发板似乎不支持该选项
```





