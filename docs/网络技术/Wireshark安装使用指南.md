

# **Wireshark安装使用指南**

## **一、Wireshark下载安装**

可以去官网下载最新版本，也可以其它平台下载相应软件。

https://www.wireshark.org/download.html

<img src="https://raw.githubusercontent.com/yanzhg/PicGo/main/202511281401417.png" alt="image-20251128135244620"  />



根据自己操作系统是64位还是32位下载相应版本的软件。

Wireshark有两种安装版本，一种是免安装（Portable）版，一种是安装版。免安装版文件大小大约150M，优点是不需要安装，下载解压就可以直接运行，缺点是文件相对较大，稳定性不如安装版；安装版文件大小大约50M，优点是运行稳定，缺点是需要安装，相较免安装版复杂一些。

对于免安装版本，可以直接下载相应文件，解压到一个目录里直接运行即可。

![image-20251128135324252](https://raw.githubusercontent.com/yanzhg/PicGo/main/202511281401418.png)

![image-20251128135357251](https://raw.githubusercontent.com/yanzhg/PicGo/main/202511281401419.png)

对于安装版本这里以安装平台为Windows 11 专业版64位系统，Wireshark为4.0.3 64位版本为例，其他版本的安装类似。

![image-20251128135416655](https://raw.githubusercontent.com/yanzhg/PicGo/main/202511281401420.png)

1、以管理员身份运行安装包。

![image-20251128135433188](https://raw.githubusercontent.com/yanzhg/PicGo/main/202511281401421.png)

2、保持默认选择，一路点击下一步，这一步需要Npcap。

![image-20251128135444417](https://raw.githubusercontent.com/yanzhg/PicGo/main/202511281401422.png)

3、这一步需要USBpcap。

![image-20251128135452873](https://raw.githubusercontent.com/yanzhg/PicGo/main/202511281401423.png)

4、开始安装Wireshark。

![image-20251128135502645](https://raw.githubusercontent.com/yanzhg/PicGo/main/202511281401424.png)

5、安装到后期阶段会提示安装NPcap，直接点击下一步默认安装即可。

![image-20251128135510335](https://raw.githubusercontent.com/yanzhg/PicGo/main/202511281401425.png)

6、安装完成后点击Next，重启电脑。

![image-20251128135521372](https://raw.githubusercontent.com/yanzhg/PicGo/main/202511281401426.png)

7、启动Wireshark，选择你需要抓取的网卡即可开始抓包。

## **二、Wireshark开始抓包示例**

这里简单介绍一下使用wireshark工具抓取ping命令操作的示例。

1、打开Wireshark，主界面如下：

![image-20251128135534092](https://raw.githubusercontent.com/yanzhg/PicGo/main/202511281401427.png)

2、选择菜单栏上“捕获“ -> ”选项“，勾选抓包网卡（这里需要根据各自电脑网卡使用情况选择，简单的办法可以看使用的IP对应的网卡），选择的抓包网卡一定要启用混杂模式。

![image-20251128135542399](https://raw.githubusercontent.com/yanzhg/PicGo/main/202511281401428.png)

![image-20251128135558320](https://raw.githubusercontent.com/yanzhg/PicGo/main/202511281401429.png)

3、点击左上角蓝色“开始捕获分组”按钮启动抓包。

![image-20251128135609177](https://raw.githubusercontent.com/yanzhg/PicGo/main/202511281401430.png)

4、Wireshark启动后，处于抓包状态中。

![image-20251128135617179](https://raw.githubusercontent.com/yanzhg/PicGo/main/202511281401431.png)

5、执行需要抓包的操作，如在cmd窗口下执行ping www.baidu.com。

![image-20251128135626176](https://raw.githubusercontent.com/yanzhg/PicGo/main/202511281401432.png)

6、操作完成后相关数据包就抓取到了。为避免其他无用的数据包影响分析，可以通过在过滤栏设置过滤条件进行数据包过滤筛选。

7、点击左上角的红色停止“停止捕获分组”按钮停止抓包，选择菜单栏上“文件”-> “另存为”选项保存文件。

![image-20251128135639096](https://raw.githubusercontent.com/yanzhg/PicGo/main/202511281401433.png)

## 三、**Wireshark抓包配置**

这里以捕获Gtmail通信报文为例说明，如何添加过滤器只捕获有用报文。

1、选好抓包的接口，即本机和Gtmail通信的网卡。

![image-20251128135653542](https://raw.githubusercontent.com/yanzhg/PicGo/main/202511281401434.png)

2、设置捕获过滤器（只抓取Gtmail报文）

ip host 34.249.183.66 || 34.249.103.184 || 52.209.42.205 || 52.57.100.58 || 35.157.202.95

![image-20251128135702844](https://raw.githubusercontent.com/yanzhg/PicGo/main/202511281401435.png)