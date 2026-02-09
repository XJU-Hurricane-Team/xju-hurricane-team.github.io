MCU Viewer目前**仅支持JLink，ST-Link**！正点原子的无线下载器DAP暂不支持。建议用JLink，JLink可以边在Keil调试边在MCU Viewer里看曲线，很方便。

从github上下载MCU Viewer：[地址](https://github.com/klonyyy/MCUViewer/releases/tag/v1.1.0)                               ![Viewerdl](./Picture/Viewerdl.png)

安装后打开，选择Option->Acquistion settings打开采样设置窗口

![图片2](./Picture/图片2.png)

在第一个elf文件位置选择编译elf文件，在build/Debug下：

![图片3](./Picture/图片3.png)



![图片4](./Picture/图片4.png)

JLink需要选择芯片，点...后会弹出一个窗口，在Device里输入芯片型号，然后选图中的就可以了，完成后点OK

![图片5](./Picture/图片5.png)

点Import variables from *elf，输入变量名搜索想要看的变量，仅支持全局变量，如果没有想要查看的变量需要在代码里加：

![图片6](./Picture/图片6.png)

添加完变量后点Done，然后把要查看变量拖到图里：

![图片7](./Picture/图片7.png)

添加完毕后点STOPPED开始，变成Running，并出现曲线就OK，如果报错检查一下接线或者其他问题：

![图片8](./Picture/图片8.png)

![图片9](./Picture/图片9.png)

可以选择Export to csv保存为csv文件后导入matlab分析：

![图片10](./Picture/图片10.png)

改代码重新编译以后需要更新一下变量地址，重新编译可能使变量地址变化，点Click to reload *.elf changes更新：

![图片11](./Picture/图片11.png)

可以保存此绘图设置，文件后缀是cfg，方便下次打开使用，免得重新设置一遍变量：

![图片12](./Picture/图片12.png)

可以点File->Open打开保存的配置文件：

![图片13](./Picture/图片13.png)