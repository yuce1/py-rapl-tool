# py-rapl-tool
此工具使用python语言开发，通过msr寄存器的方式完成Intel RAPL的设置和读取。此工具需要在Intel的处理器上运行，操作系统要求linux操作系统


依赖
============

需要三个环境依赖:
- Python 2.7
- MSR 寄存器
- terminaltables 库

安装
----------

1) 进入主目录:

	>$ cd py-rapl-tool
 
2) 加载msr寄存器:

	>$ sudo modprobe msr

3) 安装 python terminaltables包:

	>$ sudo pip install terminaltables

4) 在root权限下运行程序:

	>$ sudo ./rapl-sety


使用
----------

```
usage: rapl-sety [-h] [-s {0,1}] [-p {pkg_limit_1,pkg_limit_2,dram,pp0,pp1}]
                 [-l POWER_LIMIT] [-t TIME_WINDOW] [-el] [-dl] [-ec] [-dc]
                 [-sl] [-pl {0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16}]

Tool for RAPL setting

optional arguments:
  -h, --help                 show this help message and exit
  -s {0,1}, --socket {0,1}   Select the socket (default: all the sockets)
  -p {pkg_limit_1,pkg_limit_2,dram,pp0,pp1}, --power-lane {pkg_limit_1,pkg_limit_2,dram,pp0,pp1}
                             Select the power lane
  -l POWER_LIMIT, --power-limit POWER_LIMIT
                             Set power limit (uW)
  -t TIME_WINDOW, --time-window TIME_WINDOW
                             Set time window (us)
  -el, --enable-limit        Force enable to power limit
  -dl, --disable-limit       Force disable to power limit
  -ec, --enable-clamping     Force enable to clamping limit
  -dc, --disable-clamping    Force disable to clamping limit
  -sl, --set-lock            Set lock for power lane (WARNING: all write attempts are ignored until next RESET!)
  -pl {0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16}, --priority-level {0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16}
                             Select the priority level
```
