# windows下Pyqt + guiqwt环境的搭建

# 事情的起因
这几天为了搭一个PyQt的生产环境遇到了好一些坑。
本身搭建PyQt的目的是为了进行试验，使用PyQt开发曾经开发过的项目的难度。
原项目是使用MFC开发，利用自己开发的折线绘制控件，进行高速的工控数据采集后，显示到界面上。难点主要在，高速采集之后，绘图的效率问题。
最终通过自己开发了一套平移显示绘图控件，在新增数据的时候，每一次只绘制一部分数据，然后通过平移之前的图像与之后的链接在一起，这样达到比较高的绘图效率，才解决了问题。但是MFC界面开发复杂，在MFC界面上花费过多的时间。同时本身绘图控件的开发，可能可以利用现有开源控件进行开发来避免重复造轮子。
于是利用PyQt+guiqwt绘图库来开发的方案构建起来。guiqwt本身是QT下常用的绘图控件，效率足够高，经过一些优化是能够使用的。相对Python经常使用的matplotlib来说不够漂亮，但是进行实时数据的平移绘图，效率高出很多。PyQt是模仿QT写的图形库，可以利用QTCreate进行快速的界面构建，而Python相对C++来说，需要处理的坑更少，出现各类崩溃的异常的情况会更少一些。
本文主要讲的是PyQt+guiqwt在windows上的搭建。
目的是在之后能够通过本文进行快速的开发环境的搭建。

<!--more-->

# 开始搭建

## 安装Python
[Python官网](https://www.python.org/downloads/)  
通过Python官网下载相应版本的Python,我这里下载的是：python-3.5.2-amd64。
64位版本，记住版本号，用于之后的库的选择。

## 安装PyQt
PyQt有完整的安装包，过程中会自动安装PyQt+Qt等完整的依赖库。
[PyQt下载站点](https://sourceforge.net/projects/pyqt/?source=directory)  
浏览所有文件找到对应的PyQt完整安装包。
我下的是下面网址中的64位版本。
PyQt5-5.6-gpl-Py3.5-Qt5.6.0-x64-2
[PyQt完整安装包下载](https://sourceforge.net/projects/pyqt/files/PyQt5/PyQt-5.6/)  
安装一路下一步完成。

## 安装NumPy
为了能够使用guiqwt必须安装依赖的库。
首先安装numpy，注意的是需要安装+mk1的numpy库，否则scipy无法正常安装后使用。
[下载地址](http://www.lfd.uci.edu/~gohlke/pythonlibs/)  
搜索Numpy找到相应的位置。
我选择下载的是，与Python版本对应：numpy-1.11.2+mkl-cp35-cp35m-win_amd64.whl
使用pip3.5安装

## 安装scipy
同上，在上面的网站找到scipy。
版本我选择的是：scipy-0.18.1-cp35-cp35m-win_amd64
使用pip3.5在numpy之后安装

## 安装guiqwt
安装guiqwt
上面网站搜索guiqwt
选择guiqwt-3.0.3-cp35-cp35m-win_amd64。
在sicpy之后使用pip3.5安装

## 安装eric6
[下载地址](https://sourceforge.net/projects/eric-ide/)  
我选择的是：eric6-6.1.10和eric6-i18n-zh_CN-6.1.10
解压，放到python的安装目录下：python35/eric6
运行python35/eric6、install.py进行安装。
运行Python35\Scripts\eric6.bat 就打开了eric6

至此开发环境搭建完成。

