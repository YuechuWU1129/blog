
# 主动源OBS数据处理简单教程

**写在前面:**
2021年暑期的最后一个月简单学习了主动源OBS的数据处理，主要是中科院南海所丘学林老师课题组的一套方法，以及自己一点点补充，现在将这套方法整理出来。感谢丘老师的`sac2y`程序，以及黄信锋师兄在程序编译过程中的帮助，感谢李子正和王建师兄提供的一些脚本和指导。由于本人学习这套方法只是为了看实验室仪器的数据，若使用这套方法写文章，建议联系南海所**丘老师**(xlqiu@scsio.ac.cn)，至少要给出相应的引用。


<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->


<!-- /code_chunk_output -->

* [数据预处理](#数据预处理)
   * [数据预处理](#数据预处理)
   * [地震数据文件预处理](#地震数据文件预处理)
* [炮点数据预处理](#炮点数据预处理)
* [数据处理](#数据处理)
* [画图](#画图)
  * [Seismic Unix](#SeismicUnix)
  * [Matlab](#Matlab)
  * [SeisSee](#SeisSee)

# 数据预处理

要绘制主动源OBS地震剖面，至少需要**地震数据文件**和**炮点数据文件**。

## 地震数据文件预处理

需要的地震数据文件，是放炮时期（一般来说，炮点数据与地震数据的时间都是UTC时间）的`SAC`格式地震数据。由于大部分地震原始数据是`miniseed`格式，还需要使用[mseed2sac](https://github.com/iris-edu/mseed2sac)将地震数据转成`SAC`格式。
这里提供一个简单的脚本，可以批量将OBS Lab 仪器的地震数据转成我们需要的格式。

```bash
#!/bin/bash
#Convert the effective time miniseed format data into sac format,by YuechuWU 2021-08-24

#all station folder names
allStation=(67-S09 72-S14 73-S07 74-S02 75-S13 80-S03)
#air gun working time
allDay=(210712 210713)

for station in ${allStation[@]}
do
cd $station
echo $station

for day in ${allDay[@]}
do
cd $day
echo $day
mv *.mseed ../
cd ../
rmdir $day
done
mseed2sac *.mseed
rm *.mseed
rm *.BHH.*.SAC
cd ../
done

```

  这个脚本可以将OBS Lab存储方式的地震原始数据转化成需要的`SAC`地震数据，只需要修改`allStation`和`allDay`即可，OBS Lab存储方式的地震原始数据指的是*每个台站一个文件夹*->*每天一个文件夹*->*每个分量一个miniseed文件*。若原始数据存储方式不同，对脚本稍加修改即可。

## 炮点数据预处理

我们需要的炮点数据，是一套标准格式的txt文件，称之为ukooa文件，其中最重要信息的是每个炮点的**时刻**和**经纬度**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/7eb8ead169e8406fb0e5fd46350b14d6.png#pic_center)

ukooa文件中第二列表示每个炮点的时刻，与SAC中的时间是一样的格式，即`nzjday, nzhour, nzmin, nzsec, nzmsec`。

*分别表示“一年的第几天”、“时”、“分”、“秒”、“毫秒”。这五个变量构成了每个炮点的唯一的绝对时刻。*

ukooa文件中第五列表示每个炮点的经纬度，纬度在前，经度在后，需要注意的是经纬度都是**度分秒**的格式。
下图是一个标准的`ukooa.txt`文件：

大多数情况下，我们拿到的炮点数据不是标准的ukooa格式（如下），因此需要做一些简单的处理，主要是时刻和经纬度的格式转换，其余几列不参与运算可忽略。

![在这里插入图片描述](https://img-blog.csdnimg.cn/adfb686e40504c778a7f11eadd121f06.png#pic_center)



下面的脚本`Y2ukooa.sh`可以将上图炮点文件`pre_ukooa.txt`转化成标准的ukooa文件`ukooa.txt`（李子正提供）。


```bash
# YOU_2 clock is 1 second slower (1s less) 
# gwak shell to convert YOU_2 clock data to ukooa format

{

split ($2,d,"-")
mday=d[3]

split ($3,b,":")
h=b[1]
m=b[2]
s=b[3]

#?????
split ($4,a,".")
vel=a[1]
angle=a[2]

deg1=int($10/100);
min1=int($10-deg1*100);
sec1=($10-deg1*100-min1)*60;
lat=deg1+min1/60+sec1/3600;

deg2=int($11/100);
min2=int($11-deg2*100);
sec2=($11-deg2*100-min2)*60;
lon=deg2+min2/60+sec2/3600;

#3 means julian day 60
jday=59+mday;


# output ukooa format old
printf("S3  %3d%02d%02d%06.3f%7d %02d%02d%05.2fN%02d%02d%05.2fE%8.1lf%9.2lf%5d. %5d\n", \
         jday,h,m,s,$1,deg1,min1,sec1,deg2,min2,sec2,angle,vel,depth,$5);
}

```

使用如下命令执行上述脚本：

```bash
gawk -f Y2ukooa.sh pre_ukooa.txt>ukooa.txt
```


# 数据处理

预处理完成之后，使用`sac2y`程序将`SAC`文件转成`sgy`文件，SEG-Y格式是由SEG （Society of Exploration Geophysicists）提出的标准磁带数据格式之一，它是石油勘探行业地震数据的最为普遍的格式之一。
相关的程序源码已经放在OBS Lab服务器`/data/public/programs/sac2y_src`，由于这个程序年代久远，直接在64位机器上编译运行会出错，黄信锋修复解决了这个bug，并且写了`Makefile`文件，使源码更容易编译运行。

使用如下命令即可编译`sac2y_v2.1`源码。
```bash
make sac2y_v2.1
```
**注意**：编译时还需要`gmt_proj.h, sac_new.h,  segy.h`这三个文件，都位于`sac2y_src`文件夹中。

实际上，`sac2y_v2.1`应为`sac2y_v2.1_ew`画**东西**测线，与之对应的还有`sac2y_v2.1_ns`画**南北**测线。

编译之后，会出现一个可执行文件`sac2y_v2.1`，使用如下命令将经过预处理的`SAC`文件和`ukooa`文件转成`sgy`文件。

```bash
sac2y_v2.1 sacfile.SAC ukooafile.txt sgyfile.sgy
```
**注意**：使用`sac2y_v2.1`除了需要`SAC`文件和`ukooa`文件之外，还需要`drift.in`（台站钟漂）`loc.in`（台站位置）`par.in`（折合速度，时间长度，在零点之前的时间长度，台站位置所在高斯区域的中央经线，校正时间，放炮数）这三个文件，这三个文件的模板已上传至OBS Lab服务器`/data/public/programs/sac2y_src/parameter`，对不同的台站做出相应的修改即可。

其实所谓的数据处理，只有这一步，得到sgy文件之后，接下来的步骤就是画图了，可以使用[Seismic Unix](https://wiki.seismic-unix.org/doku.php)、Matlab(南科大购买了版权，[安装方法](https://lib.sustech.edu.cn/gjyrj_116/list.htm))以及石油勘探的很多可视化软件(如[SeiSee](https://www.softpedia.com/get/Science-CAD/SeiSee.shtml))进行画图。




# 画图
## SeismicUnix

首先需要安装Seismic Unix，安装方法（王建提供）：

```bash
#Install required packages
For fedora
yum install gcc gcc* libx* freeglut-devel mu
yum install gcc-gfortran
yum install xorg-x11-server-devel libXt-devel
 
for Ubuntu:
 sudo apt-get install build-essential
  sudo apt-get install libx11-dev
  sudo apt-get install libxt-dev
  sudo apt-get install freeglut3-dev
  sudo apt-get install libxmu-dev
  sudo apt-get install libxi-dev
  sudo apt-get install gfortran
 
#install the Seismic Unix
mkdir -p /home/wangj/programs/cwp
cd ~/programs/cwp
wget ftp://ftp.cwp.mines.edu/pub/cwpcodes/cwp_su_all_43R3.tgz
tar -zxvf cwp_su_all_43R3.tgz
 
#write the following two lines into the file ​~/.bashrc​
export CWPROOT=:/home/wangj/programs/cwp
export PATH=$PATH: /home/wangj/programs/cwp/bin
 
source ~/.bashrc
 
edit the /src/Makefile: CWPROOT =/home/wangj/programs/cwp
 
#compile
cd $CWPROOT/src
make install
make xtinstall
make finstall 
make mglinstall
make xminstall  (optional)
make sfinstall 
make utils
 
#Testing the install
suplane | suximage title="My First Plot"
suplane | suxwigb
```

**注意**：需要将路径中的`/home/wangj/`替换成自己的路径。
若`wget`不能下载成功，可以去GitHub下载[安装包](https://github.com/JohnWStockwellJr/SeisUnix)，我也将安装包上传至OBS Lab服务器`/data/public/programs/cwp`。

一个例子的结果如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/09328913223e49b082e55864469db783.png#pic_center)



**注意**：此脚本绘制的剖面是以Offset排列的。

## Matlab

使用Matlab处理`sgy`数据主要需要使用[SegyMAT](http://segymat.sourceforge.net)程序包，可以在GitHub下载[程序包](https://github.com/cultpenguin/segymat)。
使用如下脚本即可读取`sgy`文件并绘制剖面。
```javascript
clear all;close all;
[seismic_data]=ReadSegyFast(datafile);
% wiggle(seismic_data);
imagesc(seismic_data);
xlabel('TraceCount','FontSize',15,'FontWeight', 'bold');
ylabel('SampleCount','FontSize',15,'FontWeight', 'bold');
```
一个例子的结果如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/1d96e9c2b467444b84b452f74b2b5de4.png#pic_center)


**注意**：此脚本绘制的剖面是以Trace排列的。

## SeisSee
SeiSee是一个功能强大的应用程序，以一种快速的方法来可视化SEG-Y和CST格式的地震数据。只需将`sgy`文件导入程序，调整参数即可，由于参数调整都是可视化的，因此使用起来比较简单。遗憾的是该程序只有Windows版本。

一个例子的结果如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/164f5df97e4d45c39df050722ead930b.png#pic_center)



>若对此文档内容有疑惑或补充建议，欢迎联系吴越楚(12131066@mail.sustech.edu.cn)。
