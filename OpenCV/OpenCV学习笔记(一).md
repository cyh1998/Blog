#### 前言
OpenCV环境的搭建，详见：[Ubuntu安装配置OpenCV](https://www.jianshu.com/p/59608e83becb)  
Windows下配置OpenCV，可以参考：[基于VS2019配置opencv4.0](https://blog.csdn.net/qq_26884501/article/details/90770131)

#### 一、基础
首先了解一下OpenCV中的一些基本操作，在OpenCV中图像的基本容器定义为`Mat`对象。  
1.`imread()`打开图像，参数为图像的路径
```
Mat srcImage = imread("opencv.jpg");
```
2.`imshow()`显示图像，参数为窗口标题，显示图像的Mat对象
```
imshow("Display window",srcImage);
```
3.`imwrite()`保存图像，参数为文件名，保存图像的Mat对象
```
imwrite("New_opencv.jpg", srcImage);
```
4.图像的拷贝  
**注：** 单纯的使用`=`来赋值Mat对象，属于浅拷贝，其本质是指针之间的赋值！！！
```
Mat Image_1 = imread("opencv.jpg");
Mat Image_2;
Image_2 = Image_1; //当修改Image_2时，Image_1也会被改变
```
所以在OpenCV中，使用`clone()`或`copyTo()`来复制Mat对象
```
Mat Image_1 = imread("opencv.jpg");
Mat Image_2;
//clone()
Image_2 = Image_1.clone();
//copyTo()
Image_1.copyTo(Image_2);
```
#### 二、颜色空间转换
`cvtColor()`用于将图像从一个颜色空间转换到另一个颜色空间的转换，例如常用的转化为灰度图
```
cvtColor(srcImage, grayImage, COLOR_BGR2GRAY);
```
参数说明：  
`srcImage`为输入图像的Mat对象；  
`grayImage`为输出图像的Mat对象；  
`COLOR_BGR2GRAY`为颜色映射码，即将什么制式的图像转换成什么制式的图像；  
最后还有一个参数为指定目标图像的通道数，默认为0。  
此函数支持多种颜色空间之间的转换，其支持的转换类型和转换码有：  `COLOR_BGR2RGB`、`COLOR_RGB2BGR`、`COLOR_GRAY2BGR`等等，具体可以参考官方文档。  
#### 三、getStructuringElement函数
`getStructuringElement()`用于返回指定形状和尺寸的卷积核
```
Mat getStructuringElement(int shape, Size esize, Point anchor = Point(-1, -1));
```
`shape`表示内核的形状，有三种形状：  
- 矩形：MORPH_RECT;  值为 0
- 交叉形：MORPH_CROSS;  值为 1
- 椭圆形：MORPH_ELLIPSE;  值为 2

`esize`表示内核的尺寸；`anchor`表示锚点的位置，默认值为Point(-1, -1)，表示锚点位于中心。  
此函数通常在侵蚀和扩张操作之前使用，用于生成其操作的内核。  
#### 四、图像侵蚀和膨胀
`erode()`用于图像侵蚀操作；`dilate()`用于图像膨胀操作
```
erode(src, dst, element);
dilate(src, dst, element);
```
`src`为输入图像；`dst`为输出图像；`element`即为上文使用  `getStructuringElement()`生成的操作内核。  
当然，`erode()`和`dilate()`有多种重载，例如操作内核可以不使用`getStructuringElement()`来生成，直接将变量作为函数参数写入等等。具体可以参考官方文档。  
#### 五、图像二值化
二值化，即将图像上的像素点的灰度值设置为0或255，可以理解为黑白图。常用的方法为：简单阈值`threshold()`和自适应阈值`adaptiveThreshold()`  
注：图像二值化之前，首先需要将图像转化为灰度图。  
**简单阈值`threshold()`**：
```
threshold(Mat src, Mat dst, int thresh, int maxVal, thresholdType);
```
参数说明：  
src：原图像。  
dst：结果图像。  
thresh：设置的当前阈值。  
maxVal：最大阈值，一般为255。  
thresholdType：阈值类型，主要有下面几种：  
```
THRESH_BINARY	 //二进制阈值化
THRESH_BINARY_INV  //反二进制阈值化
THRESH_TRUNC  //截断阈值化
THRESH_TOZERO  //阈值化为0
THRESH_TOZERO_INV  //反阈值化为0
THRESH_MASK  //MASK阈值化
THRESH_OTSU  //OTSU阈值化
THRESH_TRIANGLE  
```
不同的阈值类型表示不同的阈值计算方法，这里不再赘述。  

**自适应阈值`adaptiveThreshold()`**  
```
adaptiveThreshold(Mat src, Mat dst, double maxValue, int adaptiveMethod, int thresholdType, int blockSize, double C)
```
参数说明：  
`src`：源图像。  
`dst`：输出图像。  
`maxVal`：最大阈值，一般为255。  
`adaptiveMethod`：表示在一个邻域内计算阈值所采用的算法，分别为 `ADAPTIVE_THRESH_MEAN_C` 和 `ADAPTIVE_THRESH_GAUSSIAN_C`。  
```
ADAPTIVE_THRESH_MEAN_C：表示计算出局部领域块的平均值再减去第七个参数C的值
ADAPTIVE_THRESH_GAUSSIAN_C：表示计算出局部领域块的高斯均值再减去第七个参数C的值
```
`thresholdType`：表示阈值类型，分别为 `THRESH_BINARY` 和 `THRESH_BINARY_INV`。(即二进制阈值或反二进制阈值)  
`blockSize`：表示计算阈值的像素邻域块的大小，值一般为3、5、7等等。  
`C`：即偏移值调整量，可以是负数。  

对于有明显的光暗反差的图像，使用全局阈值(简单阈值)的效果会非常差，推荐根据图像的特征使用自适应阈值进行二值化。
