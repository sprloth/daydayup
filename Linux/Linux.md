## Time

### 硬件时钟

又名实时时钟（RTC）或CMOS时钟，主要指主板上的时间。

```shell
# 查看硬件时间
hwclock --show
```

### 系统时钟

又名软件时间，与硬件时间分别维护，主要指操作系统里的时间。大多数操作系统的标准行为是：

- 启动时根据硬件时钟设置系统时钟
- 运行时通过时间同步联网校正系统时钟
- 关机时根据系统时钟设置硬件时钟

```shell
# 查看系统时间
timedatectl
```

### 时间标准

**localtime**：依赖于当前时区

**UTC**：与时区无关的全球时间标准

Windows的硬件时钟默认是localtime，macOs和Linux的硬件时钟默认是UTC。

```shell
# 硬件时钟设置为localtime
timedatectl set-local-rtc true
# 硬件时钟设置为UTC
timedatectl set-local-rtc false
```

Windows和Linux双系统时，显示时间会不一致，**建议让 Windows 使用 UTC，而非让 Linux 使用地方时**。

```shell
# 以管理员身份启动命令行执行
reg add "HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /d 1 /t REG_QWORD /f
```

### 时区

```shell
# 查看可用时区
timedatectl list-timezones
# 修改时区
timedatectl set-timezone Asia/Shanghai
```

### 时钟同步

```shell
# 使用网络时间协议（NTP）与网络时间同步
timedatectl set-ntp true
```

上面的命令启动环境变量`$SYSTEMD_TIMEDATED_NTP_SERVICES`中列出的第一个服务，如果未设置`$SYSTEMD_TIMEDATED_NTP_SERVICES`，则使用`systemd-timesyncd.service`服务。

`systemd-timesyncd.service`是用于跨网络同步系统时钟的守护服务，实现了一个SNTP客户端，没有NTP这么复杂。`systemd-timesyncd.service`服务配置文件是`/etc/systemd/timesyncd.conf`、`/etc/systemd/timesyncd.conf.d/*.conf`、`/run/systemd/timesyncd.conf.d/*.conf`、`/usr/lib/systemd/timesyncd.conf.d/*.conf`。`NTP=`配置会与`systemd-networkd.service`配置的NTP服务器合并，依次尝试连接直到同步成功为止，如果没有成功，则使用`FallbackNTP=`中定义的服务器。`FallbackNTP=`配置如果为空字符串，则此前的设置；如果没设置，则使用编译时设置的默认备用NTP服务器。

NTP服务器可以在[NTP项目](http://www.pool.ntp.org/)中查找，中国的服务器有`0.cn.pool.ntp.org`、`1.cn.pool.ntp.org`、`2.cn.pool.ntp.org`、`3.cn.pool.ntp.org`。

```shell
# 显示时间同步信息
timedatectl show-timesync --all
```

## Java

### Java环境切换

```sh
# 列出已安装的Java环境
# arch
archlinux-java status
# debian
update-java-alternatives --list

# 设置默认的Java环境
# arch
archlinux-java set <JAVA_ENV_NAME>
# debian
update-java-alternatives --set <JAVA_ENV_NAME>
```

## APT



## Iptables

iptables是一个配置Linux内核防火墙的命令行工具，是netfilter项目的一部分。iptables用于IPv4，ip6tables用于IPv6。

### 表链结构

iptables采用表和链的分层结构，表由一组预先定义的链组成，链包含按顺序遍历的规则。

- 表（table）

  1. filter表：用于过滤数据包，存放防火墙规则，是默认表，包含INPUT、FORWARD、OUTPUT链。
  2. nat表：用于网络地址转换，包含PREROUTING、POSTROUTING、OUTPUT链。
  3. raw表：用于决定数据包是否被状态跟踪机制处理，包含PREROUTING、OUTPUT链。
  4. mangle表：用于对特定数据包的修改，实现服务质量（QOS），包含PREROUTING、INPUT、OUTPUT、FORWARD、POSTROUTING链。

  大部分情况仅需要使用filter表和nat表。

- 链（chain）

  1. INPUT链：处理输入数据包。
  2. OUTPUT链：处理输出数据包。
  3. FORWARD链：处理转发数据包。
  4. PREROUTING链：处理路由选择前数据包。
  5. POSTOUTING链：处理路由选择后数据包。

  默认情况下，任何链中都没有规则。链的默认动作通常设置为ACCEPT，默认动作总是在一条链的最后生效，所以在默认动作生效前数据包需要通过所有存在的规则。用户可以加入自己定义的链，从而使规则集更有效并且易于修改。

- 规则（rule）

  规则由条件匹配和相应的动作（也称为目标）组成，如果数据包头符合条件，该动作就会被执行。动作可以是内置的特定目标，也可以是用户定义的链。内置目标包括ACCEPT（接收）、DROP（丢弃）、REJECT（拒绝）、REDIRECT（重定向）、SNAT（源地址转换）、DNAT（目标地址转换）、MASQUERADE（地址伪装）、LOG（日志记录）等。

### 数据包传输

从任意网络接口进入的数据包，首先进入PREROUTING链，再通过第一个路由策略决定数据包的目的地是本地主机（进入INPUT链，由本地进程处理），还是其他主机（进入FORWARD链）；本地进程发送的数据包，首先进入OUTPUT链，通过中间的路由策略决定分配给传出数据包的网络接口，进入OUTPUT链；最后一个路由策略存在是因为先前的 mangle 与 nat 链可能会改变数据包的路由信息。

## Network UPS Tools

### 守护进程

NUT 有 3 个与之关联的守护进程：

- 与 UPS 通信的驱动程序，对应`nut-driver.service`服务，被`nut-server.service`服务依赖启动。
- 将 UPS 状态从驱动程序提供给网络的服务器 （upsd），对应`nut-server.service`服务。
- 监控 upsd 服务器并根据收到的信息采取行动的客户端 （upsmon），对应`nut-monitor.service`服务。

### 配置文件

- `/etc/nut/nut.conf`

  配置启用哪些组件

  ```properties
  # 运行模式，包括：none 不自动启动；standalone 仅本地服务；netserver 本地+网络服务；netclient 使用其他主机的服务
  MODE=netserver
  ```

- `/etc/nut/ups.conf`

  配置使用的驱动程序

  ```properties
  [BK650M2-CH]
      driver = usbhid-ups
      port = auto
      serial = xxxxxx
      # 如果UPS不需要关闭
      sdorder = -1
  ```

- `/etc/nut/upsd.conf`

  配置 NUT 服务器

  ```shell
  # 服务监听的地址和端口，本地环回地址放在最后，防止获取不到IP启动失败
  LISTEN 192.168.0.xxx 3493
  LISTEN 127.0.0.1 3493
  ```

- `/etc/nut/upsd.users`

  配置 NUT 服务器用户及相关权限

  ```properties
  # 命令行用户，upsrw、upscmd命令需要使用
  [admin]
      password = xxxxxx
      actions = set
      actions = fsd
      instcmds = all
  
  # 主系统用户
  [monmaster]
      password = xxxxxx
      upsmon master
  
  # 从系统用户
  [monslave]
      password = xxxxxx
      upsmon slave
  ```

- `/etc/nut/upsmon.conf`

  配置 NUT 客户端

  ```shell
  # 监控UPS，用户密码为上面配置的，数字1表示该UPS为系统供电的电源数量，最后的master表示主系统，如果是从系统为slave
  MONITOR BK650M2-CH 1 monmaster xxxxxx master
  
  # 关闭系统的命令
  SHUTDOWNCMD "/sbin/shutdown -h +0"
  
  # UPS状态变化时执行的命令
  NOTIFYCMD /sbin/upssched
  
  # UPS状态变化时执行的操作，SYSLOG 记录日志；WALL 使用/bin/wall广播消息；EXEC 执行NOTIFYCMD命令；IGNORE 忽略，不执行操作
  # UPS状态变化包括：ONLINE 电源重新接通；ONBATT 电源断开，使用电池；LOWBATT 电池电量不足；FSD UPS正在强制关机；COMMOK 与UPS建立通信；COMMBAD 与UPS丢失通信；SHUTDOWN 系统正在关闭；REPLBATT UPS电池需要更换；NOCOMM 无法与UPS连接；NOPARENT upsmon父进程消失
  NOTIFYFLAG ONLINE SYSLOG+WALL+EXEC
  ```

- `/etc/nut/upssched.conf`

  配置 UPS 状态变化后定时执行的操作

  ```shell
  # 触发计时器时执行的命令，需要在所有AT前定义
  CMDSCRIPT /usr/local/sprloth/nut/notify.sh
  
  # 进程间通信的套接字文件
  PIPEFN /var/run/nut/upssched.pipe
  
  # 防止出现竞争的临时文件
  LOCKFN /var/run/nut/upssched.lock
  
  # 当任何UPS（*）使用电池时，启动一个名为onbattwarn的计时器，它将在30秒后触发
  AT ONBATT * START-TIMER battery 120
  # 当任何UPS（*）电源重新接通时，取消名为onbattwarn的计时器
  AT ONLINE * CANCEL-TIMER battery
  # 当任何UPS（*）电池需要更换时，立即执行replace操作
  AT REPLBATT * EXECUTE replace
  ```

### UPS 状态

- OL - On line (mains is present) - 在线，电源接通
- OB - On battery (mains is not present) - 使用电池，电源断开
- LB - Low battery - 电池电量不足
- HB - High battery  - 电池电量较高
- RB - The battery needs to be replaced - 需要更换电池
- CHRG - The battery is charging - 电池正在充电
- DISCHRG - The battery is discharging (inverter is providing load power) - 电池不在充电，直接由电源供电
- BYPASS - UPS bypass circuit is active - no battery protection is available - 没有电池，直接由电源供电
- CAL - UPS is currently performing runtime calibration (on battery) - 正在执行运行时校准，使用电池
- OFF - UPS is offline and is not supplying power to the load - 离线，没有供电
- OVER - UPS is overloaded - 过载
- TRIM - UPS is trimming incoming voltage (called "buck" in some hardware) - 正在调整输入电压
- BOOST - UPS is boosting incoming voltage - 正在提高输入电压
- FSD - Forced Shutdown (restricted use, see the note below) - 强制关机

### 电源故障处理流程

1. UPS 电池电量不足或手动触发强制关机（FSD）
2. 主系统 upsmon 设置强制关机（FSD）标志，以告诉所有从系统即将关闭（如果没有从系统，跳到第 5 步）
3. 从系统 upsmon 监控到强制关机（FSD）标志，生成 NOTIFY_SHUTDOWN 事件，等待 FINALDELAY 秒 （默认 5 秒），调用 SHUTDOWNCMD 命令，与服务器断开连接
4. 主系统 upsmon 最多等待 HOSTSYNC 秒（默认 15 秒），以便从系统断开连接
5. 主系统 upsmon 生成 NOTIFY_SHUTDOWN 事件，等待 FINALDELAY 秒 （默认 5 秒），创建 POWERDOWNFLAG 文件（默认 /etc/killpower），调用 SHUTDOWNCMD 命令
6. 关机程序会检查 POWERDOWNFLAG 文件，如果存在，则调用驱动程序关闭 UPS

### 相关命令

```shell
# 查看UPS信息
upsc BK650M2-CH
# 查看UPS状态
upsc BK650M2-CH ups.status
# 查看UPS变量
upsrw BK650M2-CH
# 查看UPS命令
upscmd -l BK650M2-CH

# 临时防止客户端采取行动（关机）
systemctl stop nut-monitor.service

# 手动触发关机（谨慎使用）
upsmon -c fsd
```

## 备忘

LANG=C

https://www.cnblogs.com/h2zZhou/p/5324385.html

http://www.mamicode.com/info-detail-1866747.html

