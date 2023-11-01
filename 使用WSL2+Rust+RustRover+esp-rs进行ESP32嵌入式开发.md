*本文创作于2023-10-31，其中所提及的内容在您阅读时可能已发生改变。*

最近买了块ESP32，打算入门Rust嵌入式开发。由于Windows上存在最大路径限制：
```
Error: Too long output directory: `**FILTER**`.
Shorten your project path down to no more than 10 characters (or use WSL2 and its native Linux filesystem). Note that tricks like Windows `subst` do NOT work!
```
且命令行提示也推荐使用WSL2，所以试着通过WSL2进行开发。踩的坑实在是太多，特此记录。

## 环境准备
### 内核准备

在Windows Store安装Ubuntu，启动并进行设置后，执行`uname -a`：
```
$ uname -a
Linux WOL-Computer 5.15.90.1-microsoft-standard-WSL2 #1 SMP Fri Jan 27 02:56:13 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
```
内核版本应大于`5.10.60.1`，若未满足需求，于PowerShell中执行`wsl --update`以更新内核。
### usbip安装
在[此处](https://github.com/dorssel/usbipd-win/releases)安装最新版本的`usbipd-win`，随后在WSL2中执行以下命令：
```
$ sudo apt install linux-tools-virtual hwdata
$ sudo update-alternatives --install /usr/local/bin/usbip usbip `ls /usr/lib/linux-tools/*/usbip | tail -n1` 20
```
### 安装Rust及esp-rs所需依赖
```
$ curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
$ source "$HOME/.cargo/env"
$ sudo apt install build-essential libssl-dev libuv1-dev pkg-config libudev-dev # esp-rs crates deps
$ cargo install cargo-generate ldproxy espup
$ cargo install --git https://github.com/SergioGasquez/espflash --branch fix/resets cargo-espflash espflash
$ espup install # be patient
$ . $HOME/export-esp.sh
```
{{< notice tip >}}
由于espflash目前版本存在错误，此处使用了[SergioGasquez/espflash](https://github.com/SergioGasquez/espflash/tree/fix/resets)修改版本，详情可参考[esp-rs/espflash #487](https://github.com/esp-rs/espflash/pull/487)

此错误预计将在`espflash 3.0.0`版本中修复，届时可使用`cargo install cargo-espflash espflash`安装官方版本。
{{< /notice >}} 
### 安装驱动
WSL2默认内核不预载任何驱动，开发者需自行安装。此处以CH340/CH341芯片驱动为例。
```
// WSL2 默认无/lib/modules文件夹，需自行配置
$ sudo apt install linux-headers-generic
$ ll /lib/modules
$ ls /lib/modules
5.15.0-88-generic
$ sudo ln -s /lib/modules/5.15.0-88-generic /lib/modules/5.15.90.1-microsoft-standard-WSL2 // kernel name in uanme -a
```
```
// https://www.wch.cn/downloads/CH341PAR_LINUX_ZIP.html 下载CH341驱动
$ sudo apt install unzip
$ unzip CH341SER_LINUX.ZIP
$ cd CH341SER_LINUX/driver
$ make
$ sudo make install
$ ls /lib/modules/$(uname -r)/kernel/drivers/usb/serial/
ch341.ko
```
## 连接开发板
### 挂载串口设备
```
PS > usbipd wsl list
BUSID  VID:PID    DEVICE                                                        STATE
1-6    1a86:7523  USB-SERIAL CH340 (COM3)                                       Not attached
PS > usbipd wsl attach --busid 1-6
PS > usbipd wsl list
BUSID  VID:PID    DEVICE                                                        STATE
1-6    1a86:7523  USB-SERIAL CH340 (COM3)                                       Attached - WSL
```
### 测试连接
```
$ lsusb
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 003: ID 1a86:7523 QinHeng Electronics CH340 serial converter
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
$ minicom -b 115200 -D /dev/ttyUSB0
// 按下开发板上的RESET按钮，终端内应出现启动信息
$ espflash board-info
[2023-11-01T07:01:31Z INFO ] Serial port: '/dev/ttyUSB0'
[2023-11-01T07:01:31Z INFO ] Connecting...
[2023-11-01T07:01:31Z INFO ] Using flash stub
Chip type:         esp32 (revision v3.1)
Crystal frequency: 40MHz
Flash size:        4MB
Features:          WiFi, BT, Dual Core, 240MHz, Coding Scheme None
MAC address:       d8:bc:38:e5:1a:4c
```
## 项目配置
### 创建项目
由于Windows与WSL2文件系统互操作性较差，项目文件应全部置于WSL2中。此处以`esp-rs/esp-idf-template`为例：
```
$ cargo generate esp-rs/esp-idf-template cargo
$ cd cargo
$ cargo run
I (451) app_start: Starting scheduler on CPU0
I (456) app_start: Starting scheduler on CPU1
I (456) main_task: Started on CPU0
I (466) main_task: Calling app_main()
I (466) esp32_helloworld: Hello, world!
I (466) main_task: Returned from app_main()
```
### 配置RustRover
Projects -> Open -> Home Directory in WSL -> 选择项目文件夹 -> OK
![](https://s2.loli.net/2023/11/01/cBKULz9QIAmw4P6.png)
打开项目后，点击 File -> Settings -> Languages & Frameworks -> Rust，将Toolchain location修改为Windows下的Rust路径，点击OK。
![](https://s2.loli.net/2023/11/01/pbyEio8IcklAeRW.png)
等待索引完成后，点击 Run -> Edit cofigurations -> Add new -> Cargo，将Run on修改为WSL，点击OK。
![](https://s2.loli.net/2023/11/01/FZL3q56EKP8TwVS.png)
点击 Run 'Run' 即可自动编译烧录，由于RustRover终端限制，无法显示进度条。
## 参考文档
https://devblogs.microsoft.com/commandline/connecting-usb-devices-to-wsl/

https://github.com/dorssel/usbipd-win/wiki/WSL-support

https://github.com/esp-rs/esp-idf-template#prerequisites

http://www.elelab.net/linux-ch34x-driver.html
