Telegram的表情分为三种：静态图片（PNG），动态图片（WEBM）和动画（Lottie），官方提供的表情包以动画格式为主。由于Telegram在动画表情上使用的TGS格式仅仅只是对Lottie/JSON进行了压缩，而Lottie导出后的文件可以重新导入Adobe After Effects进行编辑，对Telegram的动画表情进行成为了可能。本文以简单的添加图层为例，展示修改Telegram动画表情的过程。

## 0. 环境准备

所需要的软件为：Adobe After Effects，Adobe Illustrator，Adobe Creative Cloud，其中，Adobe After Effects需要安装[Bodymovin](https://github.com/airbnb/lottie-web/blob/master/build/extension/bodymovin.zxp)与[Bodymovin for Telegram Stickers](https://github.com/TelegramMessenger/bodymovin-extension)插件。

## 1. 素材准备

### 1.1 添加图片处理

Telegram的动画表情在Adobe After Effects中体现为多个形状图层的叠加，因此所添加的图层也必须预先转换为矢量图。本文使用Vectorizer.AI完成此步骤。
打开[Vectorizer.AI](https://vectorizer.ai/)，上传所需的图像，等待处理完成后导出为SVG格式。
![Pasted image 20240102105635.png](https://s2.loli.net/2024/01/02/xar9yfhv2MALHt4.png)

由于Telegram要求.tgs文件不得大于64KB，所以还需对SVG文件进行压缩。打开[svgomg](https://jakearchibald.github.io/svgomg/)，导入先前生成的SVG文件，将`Number precision`调至0，可以看到SVG文件大小大幅度下降。
![Pasted image 20240102110058.png](https://s2.loli.net/2024/01/02/kA4Y5nZWEzKIfPR.png)

将压缩后的SVG文件导入Adobe Illustrator中，再存储为Adobe Illustrator (.ai)格式即可。

### 1.2 Telegram动画表情处理

向(@Stickerdownloadbot)[https://t.me/Stickerdownloadbot]发送动画表情后，Bot将返回三种格式的文件。此处下载最后一种，文件名以tgs开头的压缩包，解压后可得到.tgs文件。
![Pasted image 20240102110517.png](https://s2.loli.net/2024/01/02/YeXiCVmvIWgj84n.png)

打开(lottie-editor)[https://michielp1807.github.io/lottie-editor/#/]，导入.tgs文件，确认动画正常后点击`Save as Lottie/JSON`按钮，即可获得可导入Adobe After Effects的Lottie/JSON格式动画。
![Pasted image 20240102110839.png](https://s2.loli.net/2024/01/02/RoPptKLOi13AuXN.png)

## 2. 导入文件

在Adobe After Effects中新建项目，随后点击`窗口-扩展-Bodymovin`，点击`Import Lottie Animation-Import Local File`，选择在步骤1.2中生成的.json文件，等待导入完成。
![Pasted image 20240102111403.png](https://s2.loli.net/2024/01/02/r3tmR7bMacvBCG4.png)

随后导入步骤1.1中生成的.ai文件，在项目框中右键单击，点击`基于所选项新建合成`。
![Pasted image 20240102111911.png](https://s2.loli.net/2024/01/02/ARnmpXfieZqtW8M.png)

点击合成框，右键选择`创建-从矢量图层创建形状`，
![Pasted image 20240102112100.png](https://s2.loli.net/2024/01/02/9NZP47fYAganrh2.png)

形状创建完成后在项目框中删除导入的.ai矢量插画文件。

随后在合成框中右键选择`变换-缩放`，将宽度或高度调整为512像素。在项目框中右键刚刚新建的合成，点击`合成设置`，将宽度或高度调整为520像素。

## 3.修改动画
Telegram动画表情导入后可能会出现多个`Lottie_Main_Comp`合成，通常第一个合成存放实际的动画。
![Pasted image 20240102113753.png](https://s2.loli.net/2024/01/02/CADzIOkXesLU8fy.png)

点击第一个`Lottie_Main_Comp`合成，将步骤2中创建的合成拖入合成框中进行调整。

## 4. 导出动画

点击`窗口-扩展-Bodymovin for Telegram Stickers`，选择第一个合成，设置导出路径，点击`Render`即可。
![Pasted image 20240102114808.png](https://s2.loli.net/2024/01/02/lBISsZrbDjJg2RG.png)
