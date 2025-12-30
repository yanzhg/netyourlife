

# **Wireshark安装使用指南**

## **一、Wireshark下载安装**

可以去官网下载最新版本，也可以其它平台下载相应软件。

https://www.wireshark.org/download.html

![image-20251229153806340](https://imgbed.2ff4b404b7448dc05919eb8ecdeae8b3.r2.cloudflarestorage.com/2025/12/14f8d7986b967f8d137d994a228bd397.png)

根据自己操作系统是64位还是32位下载相应版本的软件。

Wireshark有两种安装版本，一种是免安装（Portable）版，一种是安装版。免安装版文件大小大约150M，优点是不需要安装，下载解压就可以直接运行，缺点是文件相对较大，稳定性不如安装版；安装版文件大小大约50M，优点是运行稳定，缺点是需要安装，相较免安装版复杂一些。

对于免安装版本，可以直接下载相应文件，解压到一个目录里直接运行即可。

![image-20251229153832236](https://imgbed.2ff4b404b7448dc05919eb8ecdeae8b3.r2.cloudflarestorage.com/2025/12/52b30ca6ef06f68c603a1ed83c1e9059.png)

![image-20251229153858223](https://imgbed.2ff4b404b7448dc05919eb8ecdeae8b3.r2.cloudflarestorage.com/2025/12/452d05043cffbe35d182e6105ab67b68.png)

对于安装版本这里以安装平台为Windows 11 专业版64位系统，Wireshark为4.0.3 64位版本为例，其他版本的安装类似。

![image-20251229153915453](https://imgbed.2ff4b404b7448dc05919eb8ecdeae8b3.r2.cloudflarestorage.com/2025/12/62702f22fe5fff09aba9eb68691474f8.png)

1、以管理员身份运行安装包。

![image-20251229153925634](https://imgbed.2ff4b404b7448dc05919eb8ecdeae8b3.r2.cloudflarestorage.com/2025/12/14e64612547bb93729f2798a11073f94.png)

2、保持默认选择，一路点击下一步，这一步需要Npcap。

![image-20251229153935744](https://imgbed.2ff4b404b7448dc05919eb8ecdeae8b3.r2.cloudflarestorage.com/2025/12/6187286350e80223bff6f7a4ce41da15.png)

3、这一步需要USBpcap。

![image-20251229153946991](https://imgbed.2ff4b404b7448dc05919eb8ecdeae8b3.r2.cloudflarestorage.com/2025/12/a405af7b99b6b497aa990af546682b05.png)

4、开始安装Wireshark。

![image-20251229153958827](https://imgbed.2ff4b404b7448dc05919eb8ecdeae8b3.r2.cloudflarestorage.com/2025/12/4cca67177774b0be10a9ec8ef5fe2b46.png)

5、安装到后期阶段会提示安装NPcap，直接点击下一步默认安装即可。

![image-20251229154008718](https://imgbed.2ff4b404b7448dc05919eb8ecdeae8b3.r2.cloudflarestorage.com/2025/12/b5788137d3ac4cf7402f4f3a0d722624.png)

6、安装完成后点击Next，重启电脑。

![image-20251229154018313](https://imgbed.2ff4b404b7448dc05919eb8ecdeae8b3.r2.cloudflarestorage.com/2025/12/0e663a99cfa890622cd92a56dae5cf48.png)

7、启动Wireshark，选择你需要抓取的网卡即可开始抓包。

## **二、Wireshark开始抓包示例**

这里简单介绍一下使用wireshark工具抓取ping命令操作的示例。

1、打开Wireshark，主界面如下：

![image-20251229154032889](https://imgbed.2ff4b404b7448dc05919eb8ecdeae8b3.r2.cloudflarestorage.com/2025/12/fbeef4da428244bb50c0bda4f91e5353.png)

2、选择菜单栏上“捕获“ -> ”选项“，勾选抓包网卡（这里需要根据各自电脑网卡使用情况选择，简单的办法可以看使用的IP对应的网卡），选择的抓包网卡一定要启用混杂模式。

![image-20251229154042330](https://imgbed.2ff4b404b7448dc05919eb8ecdeae8b3.r2.cloudflarestorage.com/2025/12/bb2424577deb1fc82425326af8a0a956.png)

![image-20251229154052899](https://imgbed.2ff4b404b7448dc05919eb8ecdeae8b3.r2.cloudflarestorage.com/2025/12/21d9a8068e966fb30520ab4f5e54fe0d.png)

3、点击左上角蓝色“开始捕获分组”按钮启动抓包。

![image-20251229154103278](https://imgbed.2ff4b404b7448dc05919eb8ecdeae8b3.r2.cloudflarestorage.com/2025/12/fbeef4da428244bb50c0bda4f91e5353.png)

4、Wireshark启动后，处于抓包状态中。

![image-20251229154111729](https://imgbed.2ff4b404b7448dc05919eb8ecdeae8b3.r2.cloudflarestorage.com/2025/12/0beeb7b8327841b180618514794989da.png)

5、执行需要抓包的操作，如在cmd窗口下执行ping www.baidu.com。

![image-20251229154121431](https://imgbed.2ff4b404b7448dc05919eb8ecdeae8b3.r2.cloudflarestorage.com/2025/12/9ebe467be38dd76806c4f89e53e748be.png)

6、操作完成后相关数据包就抓取到了。为避免其他无用的数据包影响分析，可以通过在过滤栏设置过滤条件进行数据包过滤筛选。

7、点击左上角的红色停止“停止捕获分组”按钮停止抓包，选择菜单栏上“文件”-> “另存为”选项保存文件。

![image-20251229154133896](https://imgbed.2ff4b404b7448dc05919eb8ecdeae8b3.r2.cloudflarestorage.com/2025/12/a40716162d5754c6fb4c781a9fbb52e3.png)

