## 一、图像卷积运算函数
`filter2D()` 用于使用自定义的内核矩阵对图像进行卷积，即使用 `kernel` 矩阵计算图像的每个像素值，函数原型如下：
```
CV_EXPORTS_W void filter2D( InputArray src, OutputArray dst, int ddepth,
                            InputArray kernel, Point anchor = Point(-1,-1),
                            double delta = 0, int borderType = BORDER_DEFAULT );
```
参数说明：  
`InputArray src` : 输入图像  
`OutputArray dst` : 目标图像  
`int ddepth` : 目标图像深度，如果未指定则生成与输入图像深度相同的图像。也可以使用 `src.depth()` (函数返回图像深度)或者-1，来表示目标图像和输入图像深度保持一致   
`InputArray kernel` : 卷积核，即自定义内核矩阵  
`Point anchor` : 内核的基准点，默认值为(-1,-1)，表示位于kernel的中心位置  
`double delta` : 在将过滤后的像素存储在dst中之前添加到它们的可选值，默认值为0  
`int borderType` : 像素外推法，默认值是 `BORDER_DEFAULT`，即对全部边界进行计算  

根据内核矩阵的不同，使用 `filter2D()` 函数可以达到不同的图像处理效果  
**1. 锐化**  
```
//内核矩阵
Mat kernel = (Mat_<char>(3,3) <<  0, -1,  0,
                                 -1,  5, -1,
                                  0, -1,  0);
```
示例程序如下：
```
#include <iostream>
#include <opencv2/opencv.hpp>
 
using namespace std;
using namespace cv;

int main()
{
    Mat srcImage = imread("src.jpg");
    Mat sharpen_image;
    //内核矩阵
    Mat kernel = (Mat_<char>(3,3) <<  0, -1,  0,
                                     -1,  5, -1,
                                      0, -1,  0);
    //图片锐化
    filter2D(srcImage,sharpen_image,-1,kernel);

	imshow("Display window",srcImage);
    imshow("Display sharpen window",sharpen_image);
	waitKey(0);
	return 0;
}
```
运行结果：  

![锐化效果](https://upload-images.jianshu.io/upload_images/22192996-c3248c27e438d095.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**2. 浮雕**  
```
//内核矩阵
Mat kernel = (Mat_<char>(3,3) << -2, -1,  0,
                                 -1,  1,  1,
                                  0,  1,  2);
```
代码与上面类似，只需修改krenel的值。运行结果：

![浮雕效果](https://upload-images.jianshu.io/upload_images/22192996-49778bc9ba85ed27.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**3. 大纲**  
```
//内核矩阵
Mat kernel = (Mat_<char>(3,3) << -1, -1, -1,
                                 -1,  8, -1,
                                 -1, -1, -1);
```
运行结果：

![大纲效果](https://upload-images.jianshu.io/upload_images/22192996-957dcdf0362b71ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当然还有其他很多效果，如：模糊、索贝尔、拉普拉斯算子等等，这里不再赘述。

## 二、形态学操作
`morphologyEx()` 用于图像形态转换，原型如下：
```
CV_EXPORTS_W void morphologyEx( InputArray src, OutputArray dst,
                                int op, InputArray kernel,
                                Point anchor = Point(-1,-1), int iterations = 1,
                                int borderType = BORDER_CONSTANT,
                                const Scalar& borderValue = morphologyDefaultBorderValue() );
```
参数说明：  
`InputArray src` : 输入图像  
`OutputArray dst` : 目标图像  
`int op` : 形态学运算的类型，有5个类型：  
- 开运算：MORPH_OPEN：2
- 闭运算：MORPH_CLOSE：3
- 形态学梯度：MORPH_GRADIENT：4
- 顶帽：MORPH_TOPHAT：5
- 黑帽：MORPH_BLACKHAT：6

`InputArray kernel` : 操作内核，可以使用 `getStructuringElement()` 来生成内核  
`Point anchor` : 内核的基准点，默认值为(-1,-1)，表示位于kernel的中心位置  
`int iterations` : 迭代使用函数的次数，默认值为1  
`int borderType` : 像素外推法，默认值是 `BORDER_DEFAULT`，即对全部边界进行计算  
`borderValue` : 当边界为常数时的边界值  

通常使用函数的前四个参数，其他参数为默认值  
对于不同的形态学运算类型，这里阐述下一般功能和应用场景，具体的效果就不一一展示了  
参考：[https://blog.csdn.net/keen_zuxwang/article/details/72768092](https://blog.csdn.net/keen_zuxwang/article/details/72768092)  
- 开运算：先腐蚀，再膨胀，可清除一些小东西(亮的)，放大局部低亮度的区域
- 闭运算：先膨胀，再腐蚀，可清除小黑点
- 形态学梯度：膨胀图与腐蚀图之差，提取物体边缘
- 顶帽：原图像-开运算图，突出原图像中比周围亮的区域
- 黑帽：闭运算图-原图像，突出原图像中比周围暗的区域

## 三、图像平滑
图像平滑处理函数有：归一化块滤波器 `blur()`；高斯滤波器 `GaussianBlur()`；中值滤波器 `medianBlur()`；双边滤波器 `bilateralFilter()`。  
具体的原理和参数说明可以参考：[OpenCV官方文档](http://www.opencv.org.cn/opencvdoc/2.3.2/html/doc/tutorials/imgproc/gausian_median_blur_bilateral_filter/gausian_median_blur_bilateral_filter.html)

**1.归一化块滤波器**
```
CV_EXPORTS_W void blur( InputArray src, OutputArray dst,
                        Size ksize, Point anchor = Point(-1,-1),
                        int borderType = BORDER_DEFAULT );
```
**2.高斯滤波器**
```
CV_EXPORTS_W void GaussianBlur( InputArray src, OutputArray dst, Size ksize,
                                double sigmaX, double sigmaY = 0,
                                int borderType = BORDER_DEFAULT );
```
**3.中值滤波器**
```
CV_EXPORTS_W void medianBlur( InputArray src, OutputArray dst, int ksize );
```
**4.双边滤波器**
```
CV_EXPORTS_W void bilateralFilter( InputArray src, OutputArray dst, int d,
                                   double sigmaColor, double sigmaSpace,
                                   int borderType = BORDER_DEFAULT );
```




