# 无线Daplink使用手册

参考[高速无线DAP调试器Lite - 立创开源硬件平台](https://oshwhub.com/ylj2000/dap_hs_esp_open)设计，修改线序为Jlink线序，主控板Jtag即可兼容Jlink和无线Daplink

## 固件烧录与主从配对

硬件焊接完成后，usb插电脑，设备管理器会不停刷新。长按住A后重新再上电，ESP32进入烧录模式，设备管理器的通用串行总线设备显示USB JTAG/serial debug unit，即进入烧录模式。

打开flash_download_tool，选择ESP32-S3、Develop、USB。

![1](Picture/Daplink1.png)

在文件夹中选择需要烧录的固件，后方的起始地址填0，选择ESP32的串口，先点ERASE进行擦除，擦除完后点START进行烧录。（需要多点几下才会开始擦除和烧录）

![](Picture/Daplink2.png)

烧录完成后重新上电，初始化是有线模式，State灯为红色，此时设备管理器的通用串行总线设备显示Horco CMSIS-DAP v2。

## 模式切换



