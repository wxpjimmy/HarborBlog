title: Linux常用命令之top
date: 2015-11-14 16:32:40
tags: [linux,top]
---
top是linux下常用的性能分析工具，能够实时显示系统中各个进程的资源占用状况，.该命令可以按CPU使用.内存使用和执行时间对任务进行排序，而且该命令的很多特性都可以通过交互式命令或者在个人定制文件中进行设定
下面分几个部分介绍一下这个命令：  
1. 命令相关参数介绍。
2. 常用命令示例。

# 命令相关参数介绍  
在命令行执行top命令显示如下：  
{% asset_img top命令-1.png [top命令示例] %}  
## 统计信息区前五行是系统整体的统计信息。  
### 第一行是任务队列信息,其内容如下：
> top - 15:55:44 up 58 days, 21:49,  1 user,  load average: 0.00, 0.05, 0.07

其中`15:55:44`表示当前时间；`up 58 days, 21:49` 表示系统已经连续运行了58天21小时49分钟; `1 user` 表示当前登录的用户数; `load average: 0.00, 0.05, 0.07` 表示系统负载，也即任务队列的平均长度，三个数值分别代表过去1分钟、5分钟以及15分钟的系统平均负载，如果这个数除以逻辑CPU的数量，结果高于5的时候就表明系统在超负荷运转了。  
### 第二行是任务（进程）信息，其内容为：  
> Tasks: 197 total,   1 running, 196 sleeping,   0 stopped,   0 zombie

这一行表示系统现在共有197个进程，其中处于运行中的有1个，196个在休眠（sleep），stopped状态的有0个，zombie（僵尸）状态的有0个  
### 第三行是CPU信息：  
> Cpu(s):  9.6%us,  0.7%sy,  0.0%ni, 89.6%id,  0.0%wa,  0.0%hi,  0.1%si,  0.0%st

`9.6%us` 表示用户空间占用CPU的百分比为9.6%；`0.7%sy` 表示内核空间占用CPU的百分比为0.7%；`0.0%ni` 表示改变过优先级的进程占用CPU的百分比为0；`89.6%id` 表示空闲CPU的百分比为89.6%; `0.0%wa` 表示IO等待占用CPU的百分比为0；`0.0%hi` 表示硬中断（Hardware Interrupts）占用CPU的百分比为0；`0.1%si` 表示软中断（Software Interrupts）占用CPU的百分比为0.1%；`0.0%st` 表示虚拟机占用CPU的百分比为0。  
### 第四行是内存信息：  
> Mem:  32850724k total, 32569108k used,   281616k free,   491520k buffers

`32850724k total` 表示总物理内存为32G；`32569108k used` 表示已经使用的物理内存大小为将近32G；`281616k free` 表示当前空闲的物理内存大小为270多M；`491520k buffers` 表示用作内核缓存的内存大小约为480M；  
### 第五行是swap交换分区信息：
> Swap: 12582904k total,        0k used, 12582904k free, 20604232k cached

`12582904k total` 表示交换区总量为12G；`0k used` 表示当前已经使用的交换区大小为0；`12582904k free` 表示空闲的交换区总量为12G；`20604232k cached` 表示缓冲的交换区总量,内存中的内容被换出到交换区，而后又被换入到内存，但使用过的交换区尚未被覆盖，该数值即为这些内容已存在于内存中的交换区的大小,相应的内存再次被换出时可不必再对交换区写入。 

这里要说明一下，第四行中使用中的内存总量（used）指的是现在系统内核控制的内存数，空闲内存总量（free）是内核还未纳入其管控范围的数量。纳入内核管理的内存不见得都在使用中，还包括过去使用过的现在可以被重复利用的内存，内核并不把这些可被重新使用的内存交还到free中去，因此在linux上free内存会越来越少，但不用为此担心。  
如果出于习惯去计算可用内存数，这里有个近似的计算公式：第四行的free + 第四行的buffers + 第五行的cached，按这个公式此台服务器的可用内存：281616k+491520k+20604232k = 20GB左右。
对于内存监控，在top里我们要时刻监控第五行swap交换分区的used，如果这个数值在不断的变化，说明内核在不断进行内存和swap的数据交换，这是真正的内存不够用了。  
## 5行之后列出的是详细的进程统计信息，top命令提供的统计信息包含：  


序号 | 列名 | 含义
---- | ---- | ----
a | PID | 进程id
b | PPID | 父进程ID
c | RUSER | Real user name
d | UID |  进程所有者的用户ID
e | USER | 进程所有者的用户名
f | GROUP | 进程所有者的组名
g | TTY | 启动进程的终端名，不是从终端启动的进程显示为？
h | PR | 进程优先级
i | NI | 进程nice值。负值表示高优先级，正值表示低优先级，值越小代表优先级越高。
j | P | 最后使用的CPU，仅在多CPU环境下有意义
k | %CPU | 上次更新到现在的CPU时间占用百分比
l | TIME | 进程使用的CPU时间总计，单位秒
m | TIME+ | 进程使用的CPU时间总计，单位1/100秒
n | %MEM |  进程使用的物理内存百分比
o | VIRT | 进程使用的虚拟内存总量，单位kb。VIRT=SWAP+RES
p | SWAP | 进程使用的虚拟内存中，被换出的大小，单位kb。
q | RES | 进程使用的、未被换出的物理内存大小，单位kb。RES=CODE+DATA
r | CODE | 可执行代码占用的物理内存大小，单位kb
s | DATA | 可执行代码以外的部分(数据段+栈)占用的物理内存大小，单位kb
t | SHR | 共享内存大小，单位kb
u | nFLT | 页面错误次数
v | nDRT | 最后一次写入到现在，被修改过的页面数。
w | S | 进程状态(D=不可中断的睡眠状态,R=运行,S=睡眠,T=跟踪/停止,Z=僵尸进程)
x | COMMAND | 进程启动的命令行
y | WCHAN | 若该进程在睡眠，则显示睡眠中的系统函数名
z | Flags | 任务标志，参考 sched.h

示例中目前列出来的主要有：
> PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND

* 如果想更改显示出来的内容, 可以按 f 键进入选择显示的内容视图。按 f 键之后会显示列的列表，按 a-z 即可显示或隐藏对应的列，当前显示的列会用*号以及大写序号标识，没有显示的用小写标识，选择完成后按回车键确定。示例如下：  

{% asset_img top命令-2.png [top命令示例] %}

* 如果想要更改显示的列的顺序，可以按 o 键进入调整显示列顺序的视图。在该视图中可以按小写的 a-z 可以将相应的列向右移动，而大写的 A-Z 可以将相应的列向左移动。最后按回车键确定。 示例如下：  

{% asset_img top命令-3.png [top命令示例] %}  
* 如果想要更改排序的列（默认是%CPU), 可以按大写的 F 或 O 键，然后按 a-z 可以将进程按照相应的列进行排序。而大写的 R 键可以将当前的排序倒转。示例如下：

{% asset_img top命令-4.png [top命令示例] %}  

# top交互命令
top交互命令总结如下：  


交互命令 | 说明
-------- | ----
h或? | 显示帮助画面，给出一些简短的命令总结说明
Z | 修改Summary/Messages or Prompts/Column Heads/Task Information的显示颜色
z | 分颜色显示on/off
B | 打开或者关闭Bold显示模式（summary信息以及正在运行的Task）
b | 只有当x或者y正在使用的时候有效，取消／显示高亮
x | 高亮当前正在排序的列（on/off)
y | 高亮当前正在运行的Task (on/off)
l | 显示／隐藏load信息
t | 显示／隐藏task/cpu的信息
m | 显示／隐藏mem的信息
1 | 显示／隐藏每个CPU的详细信息
I | 打开／关闭Irix/Solaris模式
f | 增加／减少显示的列
o | 更改列显示的顺序
F/O | 更改排序的列
> | 向右更改排序的列
< | 向左更改排序的列
R | 顺序／逆序显示
H | 显示具体的线程信息
c | 显示／隐藏程序启动命令
i | 显示／隐藏idle的Task（进程）
S | 打开／关闭cumulative time
u | 输入某个用户名可以只显示属于该用户的进程
n/# | 设置最多显示的进程的数目
k | 输入进程ID杀掉该进程
r | 输入进程ID重新设置该进程的nice值
d/s | 设置刷新时间
W | 将更改后的配置保存到本地~/.toprc文件中
q | 退出

{% asset_img top命令-5.png [top命令示例] %}

# 常用命令示例
## 多核CPU监控(按1）
{% asset_img top命令-6.png [top命令示例] %}
## 高亮当前正在运行的进程(按y，如果没有高亮，接着按b)
{% asset_img top命令-8.png [top命令示例] %}
## 高亮当前用于排序的列(按x，如果没有高亮，接着按b）
{% asset_img top命令-7.png [top命令示例] %}
## 通过”shift + >”或”shift + <”可以向右或左改变排序列
{% asset_img top命令-9.png [top命令示例] %}
## 显示进程启动命令（按c）
{% asset_img top命令-10.png [top命令示例] %}
## 只显示某个进程（top -p <pid>)
{% asset_img top命令-11.png [top命令示例] %}


