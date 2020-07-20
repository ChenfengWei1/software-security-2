
## 实验原理

### 模糊测试

> 模糊测试 （fuzz testing, fuzzing）是一种软件测试技术。其核心思想是将自动或半自动生成的随机数据输入到一个程序中，并监视程序异常，如崩溃，断言（assertion）失败，以发现可能的程序错误，比如内存泄漏。模糊测试常常用于检测软件或计算机系统的安全漏洞。 
> 
> 模糊测试工具主要分为两类，变异测试（mutation-based）以及生成测试（generation-based）。模糊测试可以被用作白盒，灰盒或黑盒测试

### 黑盒、白盒、灰盒测试

> - 黑盒测试关注程序的功能是否正确，面向实际用户；
> - 白盒测试关注程序源代码的内部逻辑结构是否正确，面向编程人员；
> - 灰盒测试是介于白盒测试与黑盒测试之间的一种测试。
> 
> 所以通俗来讲黑盒测试就是看我开始用一个软件，它可以满足我的需求不出错吗？白盒测试就是我写的程序代码是不是没有问题呢，我得在源程序中看看。灰盒测试就是综合两种测试策略

### boofuzz

> Boofuzz is a fork of and the successor to the venerable Sulley fuzzing framework. Besides numerous bug fixes, boofuzz aims for extensibility. The goal: fuzz everything.

### BinWalk

> Binwalk是一个快速、易用的工具，用于分析、逆向工程和提取固件图像。

### QEMU

> QEMU是一个通用的开源机器仿真器和虚拟化器

### Firmadyne

> Firmadyne是一款自动化分析嵌入式Linux系统安全的开源软件，由卡内基梅隆大学的Daming D. Chen开发完成的。它支持批量检测，整个系统包括固件的爬取、root文件系统的提取、QEMU模拟执行以及漏洞的挖掘。

### Firmware Analysis Toolkit

**Note:**

> FAT is a toolkit built in order to help security researchers analyze and identify vulnerabilities in IoT and embedded device firmware. This is built in order to use for the "Offensive IoT Exploitation" training conducted by Attify.
> 
> -   As of now, it is simply a script to automate **[Firmadyne](https://github.com/firmadyne/firmadyne)** which is a tool used for firmware emulation. In case of any issues with the actual emulation, please post your issues in the [firmadyne issues](https://github.com/firmadyne/firmadyne/issues).  
>     
> -   In case you are facing issues, you can try [AttifyOS](https://github.com/adi0x90/attifyos) which has Firmware analysis toolkit and other tools pre-installed and ready to use.

### 固件分析流程

- | **Stage**                                   | **Description**                                                                                       |
  |:------------------------------------------- |:----------------------------------------------------------------------------------------------------- |
  | 1. Information gathering and reconnaissance | Acquire all relative technical and documentation details pertaining to the target device's firmware   |
  | 2. Obtaining firmware                       | Attain firmware using one or more of the proposed methods listed                                      |
  | 3. Analyzing firmware                       | Examine the target firmware's characteristics                                                         |
  | 4. Extracting the filesystem                | Carve filesystem contents from the target firmware                                                    |
  | 5. Analyzing filesystem contents            | Statically analyze extracted filesystem configuration files and binaries for vulnerabilities          |
  | 6. Emulating firmware                       | Emulate firmware files and components                                                                 |
  | 7. Dynamic analysis                         | Perform dynamic security testing against firmware and application interfaces                          |
  | 8. Runtime analysis                         | Analyze compiled binaries during device runtime                                                       |
  | 9. Binary Exploitation                      | Exploit identified vulnerabilities discovered in previous stages to attain root and/or code execution |


## 实验内容

### 安装固件分析工具包

> 固件分析工具包其实就是对Firmadyne的简单的封装，自动化了模拟新固件的过程。

```sh
git clone https://github.com/attify/firmware-analysis-toolkit

cd firmware-analysis-toolkit

./setup.sh

sudo vim fat.config
[DEFAULT]
sudo_password=attify123
firmadyne_path=/home/host/firmadyne
# These provide the sudo password as shown below.Firmadyne requires sudo privileges for some of its operations. The sudo password is provided to automate the process.
```

<img src='%E6%A8%A1%E7%B3%8A%E6%B5%8B%E8%AF%95Fuzzing/2020-07-20-16-00-17.png' width='' height='200'/>

### D-link: Dir-601 Firmware 

实物图

<img src='%E6%A8%A1%E7%B3%8A%E6%B5%8B%E8%AF%95Fuzzing/2020-07-20-15-39-58.png' width='' height='200'/>

固件漏洞

<img src='%E6%A8%A1%E7%B3%8A%E6%B5%8B%E8%AF%95Fuzzing/2020-07-20-15-50-24.png' width='' height='250'/>

- 下载[Firmware for D-link DIR-601](https://rebyte.me/en/d-link/89513/file-592154/)
  - 该固件版本日期为2012/11/16,而漏洞`CVE-2018-12710`则于2018/8/29才被公开

<img src='%E6%A8%A1%E7%B3%8A%E6%B5%8B%E8%AF%95Fuzzing/2020-07-20-16-02-35.png' width='' height='50'/>

### 漏洞复现(CVE-2018-12710)

- [DLink DIR-601 - Credential Disclosure - Hardware webapps Exploit](https://www.exploit-db.com/exploits/45306)

> 在D-Link DIR-601 2.02NA设备上发现了一个问题 在网络本地,并且仅具有 用户 帐户(这是低特权帐户)访问权限,由于管理员密码以XML显示,因此攻击者可以拦截POST请求的响应以获得 管理员权限

Running FAT
```sh
sudo ./fat.py dir601_revB_FW_201.bin
```
- QEMU模拟路由器
- ps:若不以sudo权限运行可能会有`timeout error`

<img src='%E6%A8%A1%E7%B3%8A%E6%B5%8B%E8%AF%95Fuzzing/2020-07-20-17-16-34.png' width='' height='250'/>

- 由于莫名的bug（死循环），FAT没法正常启动QEMU :( .懒得折腾了
- 漏洞复现的大致思路是
  - 用**用户身份**登录
  - 进入后选择**Tools option**->**Do Intercept response from this request**
  - burpsuite 截获以xml显示的**admin**信息

```html
HTTP/1.1 200 OK
Content-type: text/xml
Connection: close
Date: Sat, 01 Jan 2011 00:19:56 GMT
Server: lighttpd/1.4.28
Content-Length: 20088

<?xml version="1.0" encoding="UTF-8"?><root><login_level>0</login_level><admin_user><admin_user_name>admin</admin_user_name>
<admin_user_pwd>testagain</admin_user_pwd><admin_level>1</admin_level></admin_user><user_user><user_user_name>user</user_user_name>
<user_user_pwd></user_user_pwd><user_level>0 ...
```

### Fuzz

安装
- [Installing boofuzz — boofuzz 0.2.0 documentation](https://boofuzz.readthedocs.io/en/stable/user/install.html)

由于不可抗力，只能以`FUZZ ftp服务`案例作为boofuzz使用示例
- ps:正常情况下多一步burp-suite，其他类似

```
python ftp_simple.py
```

<img src='%E6%A8%A1%E7%B3%8A%E6%B5%8B%E8%AF%95Fuzzing/2020-07-20-12-42-12.png' width='' height='200'/>

<img src='%E6%A8%A1%E7%B3%8A%E6%B5%8B%E8%AF%95Fuzzing/2020-07-20-13-03-39.png' width='' height='250'/>

## 参考

- [黑盒测试，白盒测试和灰盒测试的区别？ - 知乎](https://www.zhihu.com/question/35451773/answer/282980432)
- [OWASP Firmware Security Testing Methodology](https://github.com/scriptingxss/owasp-fstm)
- [通过CVE-2017-17215学习路由器漏洞分析，从入坑到放弃 - FreeBuf网络安全行业门户](https://www.freebuf.com/vuls/160040.html)
- [使用boofuzz进行漏洞挖掘(一) - FreeBuf网络安全行业门户](https://www.freebuf.com/vuls/185606.html)