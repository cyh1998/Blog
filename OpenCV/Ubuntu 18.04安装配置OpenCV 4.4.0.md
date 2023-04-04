## 概述
本文介绍ubuntu下OpenCV的编译安装以及环境配置，ubuntu版本18.04

## OpenCV下载
下载地址 [OpenCV官网](https://opencv.org/releases/)，选择最新的4.4.0版本(如果下载速度太慢，复制链接地址，使用迅雷)  

![opencv官网](https://upload-images.jianshu.io/upload_images/22192996-e9ecdc9482e98061.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

将下载好的压缩文件拷贝到虚拟机中  
##  编译与安装
**1. 安装cmake**  
OpenCV需要使用cmake进行编译  
```
sudo apt-get install cmake
```
**2. 安装依赖**  
```
sudo apt-get install build-essential pkg-config libgtk2.0-dev libavcodec-dev libavformat-dev libjpeg-dev libswscale-dev libtiff5-dev
```
**3. 解压**  
```
unzip opencv-4.4.0
```
**4. 进入文件目录，创建build目录并进入**  
```
cd opencv-4.4.0/
mkdir build
cd build
```
**5. 使用cmake生成makefile文件**  
```
cmake -D CMAKE_BUILD_TYPE=RELEASE -D CMAKE_INSTALL_PREFIX=/usr/local -D WITH_GTK=ON -D OPENCV_GENERATE_PKGCONFIG=YES ..
```
`CMAKE_BUILD_TYPE=RELEASE` ：表示编译发布版本  
`CMAKE_INSTALL_PREFIX` ：表示生成动态库的安装路径，可以自定义  
`WITH_GTK=ON` ：这个配置是为了防止GTK配置失败：即安装了 `libgtk2.0-dev` 依赖，还是报错未安装  
`OPENCV_GENERATE_PKGCONFIG=YES` ：表示自动生成OpenCV的pkgconfig文件，否则需要自己手动生成。  
**6. 编译**  
```
make -j8
```
`-j8` 表示使用多个系统内核进行编译，从而提高编译速度，不清楚自己系统内核数的，可以使用 `make -j$(nproc)`  
如果编译时报错，可以尝试不使用多个内核编译，虽然需要更长的编译时间，但是可以避免一些奇怪的报错  
**7. 安装**  
```
sudo make install
```
**注：** 如果需要重新cmake，请先将build目录下的文件清空，再重新cmake，以免发生错误

## 环境配置
**1. 将OpenCV的库添加到系统路径**  
方法一：配置ld.so.conf文件
```
sudo vim /etc/ld.so.conf
```
在文件中加上一行 `include /usr/loacal/lib`，这个路径是cmake编译时填的动态库安装路径加上/lib

![配置ld.so.conf文件](https://upload-images.jianshu.io/upload_images/22192996-ef4258f9973ec97b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

方法二：手动生成opencv.conf文件
```
sudo vim /etc/ld.so.conf.d/opencv.conf
```
是一个新建的空文件，直接添加路径，同理这个路径是cmake编译时填的动态库安装路径加上/lib
```
/usr/local/lib
```
以上两种方法配置好后，执行如下命令使得配置的路径生效
```
sudo ldconfig
```
**2. 配置系统bash**  
因为在cmake时，选择了自动生成OpenCV的pkgconfig文件，在`/usr/local/lib/pkgconfig`路径可以看到文件  

![opencv4.pc](https://upload-images.jianshu.io/upload_images/22192996-6fda6826fd18e64a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

确保文件存在，执行如下命令
```
sudo vim /etc/bash.bashrc
```
在文末添加
```
PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/local/lib/pkgconfig
export PKG_CONFIG_PATH
```
如下：

![bash.bashrc](https://upload-images.jianshu.io/upload_images/22192996-4fde1ccde9a6e7e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

保存退出，然后执行如下命令使配置生效
```
source /etc/bash.bashrc
```
至此，Linux\Ubuntu18.04环境下OpenCV的安装以及配置已经全部完成，可以使用以下命令查看是否安装和配置成功
```
pkg-config --modversion opencv4
pkg-config --cflags opencv4
pkg-config --libs opencv4
```
![结果](https://upload-images.jianshu.io/upload_images/22192996-073398fd20136fbd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 测试
新建一个 `demo.cpp` 文件，代码如下
```
#include <iostream>
#include <opencv2/opencv.hpp>
 
using namespace std;
using namespace cv;

int main()
{
	Mat srcImage = imread("opencv.jpg");
    imshow("Display Image window",srcImage);
	waitKey(0);
	return 0;
}
```
同级目录放一张图片，名为 `opencv.jpg`，编译
```
g++ `pkg-config opencv4 --cflags` demo.cpp  -o demo `pkg-config opencv4 --libs`
```
运行
```
./demo
```
结果如下：

![测试图片](https://upload-images.jianshu.io/upload_images/22192996-38c8e4a550b3476b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**大功告成！！！**
