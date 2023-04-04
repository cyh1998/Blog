## 概述
OpenCV可以简单快速的实现检测出物体的轮廓。本文介绍OpenCV中的查找轮廓函数 `findContours()` 和描绘轮廓函数 `drawContours()`。

## 函数原型
**findContours**  
```
void findContours(InputArray image, OutputArrayOfArrays contours,
                 OutputArray hierarchy, int mode,
                 int method, Point offset = Point());
```
参数说明：  
`image` ：图像源，8位单通道图像。  
`contours` ：检测到的轮廓，类型为 `vector<vector<Point>>`，每个轮廓被存储为一组由Point点构成的集合。数组中保存多组点集，即多个轮廓。  
`hierarchy` ：图像轮廓的拓扑信息，类型为 `vector<Vec4i>`，与
轮廓的个数一致。数组内每一个元素的4个变量hierarchy[i][0] ~hierarchy[i][3]，分别表示第i个轮廓的**后一个轮廓**、**前一个轮廓**、**子轮廓**、**父轮廓**的索引编号。如果没有对应的轮廓，则相应位置的值为-1。  
`mode` ：轮廓检索模式，取值：  
- `RETR_EXTERNAL` ：只检测最外侧轮廓。  
- `RETR_LIST` ：检索所有轮廓，但不建立任何层次结构，彼此相互独立。  
- `RETR_CCOMP` ：检测所有轮廓，然后组成两级层次结构。一级是最外部轮廓，二级是其内部洞的轮廓。如果内部还有嵌套轮廓，会组成新的二级层次结构。  
- `RETR_TREE` ：检索所有轮廓，组成完整的嵌套层次结构。  
- `RETR_FLOODFILL`  

`method` ：轮廓近似模式，取值：  
- `CHAIN_APPROX_NONE` ：存储轮廓上所有连续的点。  
- `CHAIN_APPROX_SIMPLE` ：存储轮廓上的拐点。  
- `CHAIN_APPROX_TC89_L1` ：应用Teh-Chin链近似算法中的一种风格。  
- `CHAIN_APPROX_TC89_KCOS` ：应用Teh-Chin链近似算法中的一种风格。  

`offset` ：偏移量  

**drawContours**  
```
void drawContours(InputOutputArray image, InputArrayOfArrays contours,
                 int contourIdx, const Scalar& color,
                 int thickness = 1, int lineType = LINE_8,
                 InputArray hierarchy = noArray(),
                 int maxLevel = INT_MAX, Point offset = Point());
```
参数说明：  
`image` ：绘制的目标图像。  
`contours` ：待绘制的轮廓，即 `findContours()` 得到的轮廓。  
`contourIdx` ：待绘制的轮廓序号。例如：0为绘制第1个轮廓 contours[0]；1为绘制第2个轮廓 contours[1]，依次类推；-1为绘制所有轮廓。  
`color` ：绘制轮廓的颜色。  
`thickness` ：轮廓粗细。如果是-1，则为填充模式。  
`lineType` ：轮廓粗细的线型，4联通 或 8联通，默认LINE_8。  
`hierarchy` ：可选，有关层次结构的信息。即 `findContours()` 得到的层次结构信息。  
`maxLevel` ：绘制轮廓的最大级别。如果为0，则仅绘制**当前轮廓**。  
如果为1，则绘制**该轮廓和其子轮廓**。如果是2，则绘制**该轮廓和所有的嵌套子轮廓**。这个参数只有当有可用的层次结构信息时才考虑。  
`offset` ：偏移量。  

## 实现
示例代码如下：
```
#include <vector>
#include <opencv2/opencv.hpp>

void main() {
    cv::Mat src = cv::imread("C:\\Users\\PC\\Desktop\\cv\\contour.jpg");
    cv::Mat dst = src.clone();
    vector<vector<cv::Point>> contours;
    vector<cv::Vec4i> hierarchy;

    //简单预处理
    cv::cvtColor(src, src, cv::COLOR_BGR2GRAY);
    cv::threshold(src, src, 128, 255, cv::THRESH_BINARY);

    //检索轮廓
    cv::findContours(src, contours, hierarchy, cv::RETR_TREE, cv::CHAIN_APPROX_NONE);

    //两种写法都可以绘制出所有轮廓
    cv::drawContours(dst, contours, -1, cv::Scalar(0, 0, 255), 2, 8, hierarchy); //contourIdx == -1，绘制所有轮廓
    /*for (int i = 0; i < contours.size(); i++) { //遍历所有轮廓
        cv::drawContours(dst, contours, i, cv::Scalar(0, 0, 255), 2, 8, hierarchy, 0); //分别绘制当前轮廓
    }*/
    
    //显示图像
    cv::imshow("src", src);
    cv::imshow("dst", dst);
    cv::waitKey(0);
}
```
## 效果

![原图](https://upload-images.jianshu.io/upload_images/22192996-6f83ae3cdef042f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![绘制轮廓](https://upload-images.jianshu.io/upload_images/22192996-dcbe1b57e4382804.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

将轮廓检测模式改为 `RETR_EXTERNAL`，只检测最外侧轮廓
```
//检索轮廓
cv::findContours(src, contours, hierarchy, cv::RETR_EXTERNAL, cv::CHAIN_APPROX_NONE);

//绘制轮廓
cv::drawContours(dst, contours, -1, cv::Scalar(0, 0, 255), 2, 8, hierarchy); 
```
效果如下：

![检测最外侧轮廓](https://upload-images.jianshu.io/upload_images/22192996-230ff7020e6a2121.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 进一步讨论
### 1. 预处理
预处理目的是为轮廓查找提供高质量的输入源图像。  
由于上文测试的图片是简单的黑白图，所以只是进行了简单二值化处理。  
常见的预处理步骤包括：  
- 图像去噪：使用高斯滤波 `GaussianBlur()`
- 形态学操作: `morphologyEx()`
- 边缘检测: 使用canny算子`Canny()`

根据图像的具体内容，综合使用以上的方法等，具体这里就不再赘述。

### 2. 轮廓筛选
当了解了图像检测出轮廓的层次结构，我们可以根据 `hierarchy` 轮廓层次结构的信息以及 `maxLevel` 绘制轮廓的最大级别来对检测出轮廓进行筛选绘制。  
首先检测模式使用 `RETR_TREE` 来检索所有轮廓，并组成完整的嵌套层次结构。  
**2.1 绘制最内部轮廓**  
利用 `hierarchy[2]` 的值，来判断没有子轮廓的轮廓，代码如下：  
```
//检索轮廓
cv::findContours(src, contours, hierarchy, cv::RETR_TREE, cv::CHAIN_APPROX_NONE);

for (int i = 0; i < contours.size(); i++) { //遍历所有轮廓
    if (hierarchy[i][2] == -1) { //最内部轮廓
        cv::drawContours(dst, contours, i, cv::Scalar(0, 0, 255), 2, 8, hierarchy, 0); //绘制当前轮廓
    }
}
```
效果如下：

![绘制最内部轮廓](https://upload-images.jianshu.io/upload_images/22192996-b24a3e3148feee35.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**2.2 绘制最外部轮廓及其子轮廓**  
利用 `hierarchy[3] == -1` 来判断是否为最外部轮廓，并利用 `maxLevel` 来绘制该轮廓和其子轮廓，代码如下：
```
//检索轮廓
cv::findContours(src, contours, hierarchy, cv::RETR_TREE, cv::CHAIN_APPROX_NONE);

for (int i = 0; i < contours.size(); i++) { //遍历所有轮廓
    if (hierarchy[i][3] == -1) { //最外部轮廓
        cv::drawContours(dst, contours, i, cv::Scalar(0, 0, 255), 2, 8, hierarchy, 1); //绘制该轮廓和其子轮廓
    }
}
```
效果如下：

![绘制最外部轮廓及其子轮廓](https://upload-images.jianshu.io/upload_images/22192996-4d12c2cd10b9af36.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

利用 `hierarchy` 配合 `maxLevel` 可以实现不同的筛选需求，这里不再一一赘述。