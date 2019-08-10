## 系统时间

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

