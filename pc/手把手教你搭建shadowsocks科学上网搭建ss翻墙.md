# 手把手教你搭建shadowsocks科学上网 搭建ss翻墙

本文从零开始，手把手教你搭建自己的shadowsocks代理服务器实现科学上网。可用翻墙方法，史上最全的小白搭建ss教程。内容包括VPS购买，连接VPS，一键搭建shadowsocks，开启bbr加速，客户端配置shaodowsocks。

## 购买VPS

VPS（Virtual private server，虚拟专用服务器），个人用来搭建一些博客，跑跑脚本足够了。今天的教程就用VPS来搭建属于自己的shaodowsocks，一个人独占一条线路。

**Vultr**是美国的一个VPS服务商，全球有15个数据中心，可以一键部署服务器。采用**小时计费策略**，可以在任何时间新建或者摧毁VPS。价格低廉，最便宜的只要2.5一个月，**支持支付宝**。

### 新用户注册

优惠注册链接：[www.vultr.com](https://tiaozhuan.win/vultr?utm_source=textarea.com&utm_medium=textarea.com&utm_campaign=article)![img](https://www.textarea.com/image/baea1f91e0f15d4dac35bc3d84d1969d.png!w1920!h1080!t.png)填写邮箱、密码（至少10个字符，并且有一个大写字母&一个小写字母&一个数字），最后点击后面的Create Account即可。注册完会收到一封验证邮件，验证即可~

### 充值

Vultr实际上是折算成小时来计费的，比如服务器是5美元1个月，那么每小时收费为5/30/24=0.0069美元 会自动从账号中扣费，只要保证账号有钱即可~而费用计算是从你开通时开始计算的，不管你有没有使用都会扣费，即使你处于关机状态，唯一停止计费的方法是Destroy掉这个服务器！Vultr提供的服务器配置包括：

2.5美元/月的服务器配置信息：单核 512M内存 20G SSD硬盘 100M带宽 500G流量/月

5美元/月的服务器配置信息：单核 1G内存 25G SSD硬盘 100M带宽 1000G流量/月

10美元/月的服务器配置信息：单核 2G内存 40G SSD硬盘 100M带宽 2000G流量/月

20美元/月的服务器配置信息：2cpu 4G内存 60G SSD硬盘 100M带宽 3000G流量/月

40美元/月的服务器配置信息：4cpu 8G内存 100G SSD硬盘 100M带宽 4000G流量/月

验证并登录后我们会跳转到充值界面，或者从**Billing**->**Make Patment**进入：![img](https://www.textarea.com/image/bc38e0af1ef78755b679db9bdcd82a90.png!w1920!h1080!t.png)支持支付宝~充值10刀，按小时扣费，只要保证账户有余额，你的服务器就会一直运行~

### 新机器创建

选择右上角的蓝色+号按钮，进入**Deploy**页面，选择服务器配置：![img](https://www.textarea.com/image/5c8c311b969bca4a5a618dff4a6c299d.png!w1920!h1080!t.png)目前2.5的还有迈尔密和纽约的没有售罄，对于ping值和速度要求不是特别高的可以选择这里的（毕竟美国东海岸城市，离国内有点远）~

推荐服务器使用**洛杉矶**的~ 

**Server Type**选择**Ubuntu 16.04**。

之后在**Additional Features**中勾选**Enable IPv6**:![img](https://www.textarea.com/image/c3b790ca431964c81634ae68678da91b.png!w1920!h1080!t.png)其他都直接默认即可~最后点击右下角的**Deploy Now**开始新建~

### 获取VPS登录信息

选择Deploy后，过个几分钟，就可以看到自己的服务器信息了，具体位置在**Servers**->**Instances**，点击选择你新建的实例：![img](https://www.textarea.com/image/b00bb1e61deb874d79f0dff3f653adb0.png!w1920!h1080!t.png)其中，红框选中的部分从上到下依次是IP，用户名和密码~

## 连接VPS

### Windows连接VPS

1.下载Xsehll 直接在[百度软件中心](http://rj.baidu.com/soft/detail/15201.html?ald&utm_source=textarea.com&utm_medium=textarea.com&utm_campaign=article)下载，下载后正常安装即可~

2.连接linux 选择**文件**->**新建**：![img](https://www.textarea.com/image/3c4ca8a1af7621607327df0f7ec3041f.png!w1920!h1080!t.png)在主机位置输入你的VPS IP：![img](https://www.textarea.com/image/2ae387eedb766bdfdd07b5c28a0ae07d.png!w1920!h1080!t.png)确定后会让你输入你的Linux用户名：![img](https://www.textarea.com/image/795cdb762723679afad6f468513f16a1.png!w1920!h1080!t.png)之后是Linux用户密码：![img](https://www.textarea.com/image/6130c1f938175cc3a7f58933f4e145b7.png!w1920!h1080!t.png)如果显示如下图所示就表示连接成功了（如果是Vultr，那么连接成功标志应该是root@vultr）：![img](https://www.textarea.com/image/861e487054321d744c35316b4273e3fb.png!w1920!h1080!t.png)

### Mac OS连接VPS

直接打开Terminal终端，输入：ssh root@43.45.43.21，之后输入你的密码就可以登录了（输入密码的时候屏幕上不会有显示）![img](https://www.textarea.com/image/2af570f78623505b973da2b0a9de5215.png!w1920!h1080!t.png)

## 一键搭建shaodowsocks

这里采用一键脚本[一键脚本搭建shadowoscks并开启bbr内核加速](https://github.com/flyzy2005/ss-fly?utm_source=textarea.com&utm_medium=textarea.com&utm_campaign=article)

1.下载一键搭建ss脚本文件（直接复制这段代码运行即可）

```
git clone https://github.com/Flyzy2005/ss-fly
```

![img](https://www.textarea.com/image/8621c5a4c526b42cff6811f60abd3180.png!w1920!h1080!t.png)2.运行搭建ss脚本代码

```
ss-fly/ss-fly.sh -i password 1024
```

![img](https://www.textarea.com/image/12d675a67606b3649bb128937524fc3e.png!w1920!h1080!t.png)其中**password**换成你要设置的shadowsocks的密码即可，密码最好只包含密码+数字，一些特殊字符可能会导致冲突。而第二个参数1024是端口号，也可以不加，不加默认是1024~（**举个例子**，脚本命令可以是`ss-fly/ss-fly.sh -i qwerasd`，也可以是`ss-fly/ss-fly.sh -i qwerasd 8585`，后者指定了服务器端口为8585，前者则是默认的端口号1024）。 出现如下界面就说明搭建好了~![img](https://www.textarea.com/image/088f8e125f07cdb4c022420236d57dc5.png!w1920!h1080!t.png)**注**：如果需要改密码或者改端口，只需要重新再执行一次**搭建ss脚本代码**就可以了~

## 开启BBR加速

BBR是Google开源的一套内核加速算法，可以让你搭建的shadowsocks速度上一个台阶。

1.检测Ubuntu内核版本

BBR支持4.9以上的，如果你的版本高于这个则会直接开启BBR加速，如果低于这个版本则会自动下载4.10的并重启，执行如下脚本命令：

```
ss-fly/ss-fly.sh -bbr
```

![img](https://www.textarea.com/image/ad1928c9f80ccf87191e23c9f8d76d10.png!w1920!h1080!t.png)第一次会检测内核版本并自动更新，更新后会重启VPS，再根据**连接VPS**部分教程重新连接VPS即可。 2.开启BBR加速

```
ss-fly/ss-fly.sh -bbr
```

![img](https://www.textarea.com/image/3f89b96ac5e0ad8bc19b35fe3ab288cc.png!w1920!h1080!t.png)重新连接后，再次运行一次这个命令即可开启bbr加速。

## 本机配置shadowsocks

各版本的shadowsocks客户端下载地址可以参考：[Android/Windows/iOS/Mac/Linux shadowsocks客户端下载地址](https://www.flyzy2005.com/fan-qiang/shadowsocks/ss-clients-download/?utm_source=textarea.com&utm_medium=textarea.com&utm_campaign=article)

### WIndows客户端配置

双击运行shadowsocks.exe，之后会在任务栏有一个小飞机图标，右击小飞机图标，选择**服务器**->**编辑服务器**：![img](https://www.textarea.com/image/b4bd4c8c40540c970e5571bc4d7f8bf4.png!w1920!h1080!t.png)在shadowsocks的windows客户端中，**服务器IP**指你购买的VPS的IP，**服务器端口**指你服务器的配置文件中的端口，**密码**指你服务器的配置文件中的密码，**加密**指你服务器的配置文件中的加密方式，**代理端口**默认为1080不需要改动。**其他都可以默认**。设置好后，点击添加按钮即可。

### Mac OS客户端配置

双击运行shadowsocksX-NG.app，之后会在任务栏有一个小飞机图标，右击小飞机图标，选择**服务器**->**服务器设置**：![img](https://www.textarea.com/image/2e7d14c10bc88e8deac665c86d576ba6.png!w1920!h1080!t.png)在shadowsocks的Mac OS客户端中，**地址**指你购买的VPS的IP，冒号后面跟上配置文件中的**端口**，**密码**指你服务器的配置文件中的密码，**加密**指你服务器的配置文件中的加密方式。**其他都可以默认**。设置好后，点击确认即可。

### 安卓客户端配置

下载apk安装好后，打开影梭客户端，点击主界面左上角的**编辑按钮**（铅笔形状）：![img](https://www.textarea.com/image/c762a98dadeed9905d405a279e471b2a.png!w1920!h1080!t.png)在shadowsocks安卓客户端的配置中填入相应配置信息，其中，**功能设置**中，**路由**改成如上图所示，其他都可以默认。

### 苹果客户端配置

shadowsocks苹果客户端经常会被App Store下架，可以在App Store搜索关键字**shadowsock**或者**wingy**，找到一个软件截图中包括填写ip，加密方式，密码的软件一般就是对的了（目前可以用的是FirstWingy）。当然，你也可以下载PP助手，之后在PP助手上下载Wingy（Wingy支持ssr）或者shadowrocket（shadowrocket支持ssr）。![img](https://www.textarea.com/image/7473e0bc142d3edd9758ca76d0d6ed0c.png!w1920!h1080!t.png)

## 总结

如上就是手把手教你搭建shadowsocks的全部内容。在国内，VPN是不允许的了，所以还不如自己搭建ss，可以独享一个线路。