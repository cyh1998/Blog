## 前 言
本文介绍图像细化算法之 **Rosenfeld算法**。

## 算法步骤
网上有关Rosenfeld算法原理的讲解很少，这里主要参考：https://www.cnblogs.com/mikewolf2002/p/3327318.html   
Rosenfeld算法一样是基于二值化图像进行的，设当前被处理的像素为p0，使用以下所示方式来表示当前像素的八邻域，背景像素值为0(黑色)，前景像素值为255(白色)，为了方便计算，在算法过程中将前景像素值当作1来计算。  
```
p8  p1  p2
p7  p0  p3
p6  p5  p4
```
首先，介绍几个算法中的概念：  
**一、方位边界点**  
对于当前像素p0：  
若p1 = 0，p0为北部边界点  
若p5 = 0，p0为南部边界点  
若p7 = 0，p0为西部边界点  
若p3 = 0，p0为东部边界点 

**二、孤立点**  
若p0周围8个像素的值都为0，则p0为孤立点。 

**三、端点**  
若p0周围8个像素有且只有1个像素值为1，则p0为端点。  

**四、8 simple**  
即将p0的值设置为0后，不会改变周围8个像素的8连通性。  

了解了算法中的基本概念，Rosenfeld算法的步骤如下：  
**1.** 扫描所有像素，如果该像素是北部边界点，且为8simple，但不是孤立点和端点，则删除。  
**2.** 扫描所有像素，如果该像素是南部边界点，且为8simple，但不是孤立点和端点，则删除。  
**3.** 扫描所有像素，如果该像素是东部边界点，且为8simple，但不是孤立点和端点，则删除。  
**4.** 扫描所有像素，如果该像素是西部边界点，且为8simple，但不是孤立点和端点，则删除。  

执行完上述4个步骤后，即完成了一次迭代，重复执行上面的过程，直到图像中再也没有可以删除的像素点后，退出循环。  

## 算法实现
根据上述的算法概念和步骤，整体代码如下
```
#include <iostream>
#include <vector>
#include <opencv2/opencv.hpp>

using namespace std;

void Rosenfeld_Thin(cv::Mat& src, cv::Mat& dst) {
    if (src.type() != CV_8UC1) {
        cout << "只处理二值化图像，请先二值化图像" << endl;
        return;
    }

    dst = src.clone();

    int width = src.cols;
    int height = src.rows;
    int step = src.step;
    int  p1, p2, p3, p4, p5, p6, p7, p8; //八邻域
    uchar* img;
    bool ifEnd;
    cv::Mat tmpimg;
    int dir[4] = { -step, step, -1, 1 };

    while (1) {
        ifEnd = false;
        for (int n = 0; n < 4; n++) { //分四个迭代过程，分别为北，南，西，东四个边界情况
            dst.copyTo(tmpimg);
            img = tmpimg.data;
            for (int i = 0; i < height; i++) {
                for (int j = 0; j < width; j++) {
                    uchar* p = img + i * step + j;
                    if (p[0] == 0 || p[dir[n]] != 0) continue; //如果p0是背景点或不是方位边界点，跳过本次循环
                    p1 = p[(i == 0) ? 0 : -step] > 0 ? 1 : 0;
                    p2 = p[(i == 0 || j == width - 1) ? 0 : -step + 1] > 0 ? 1 : 0;
                    p3 = p[(j == width - 1) ? 0 : 1] > 0 ? 1 : 0;
                    p4 = p[(i == height - 1 || j == width - 1) ? 0 : step + 1] > 0 ? 1 : 0;
                    p5 = p[(i == height - 1) ? 0 : step] > 0 ? 1 : 0;
                    p6 = p[(i == height - 1 || j == 0) ? 0 : step - 1] > 0 ? 1 : 0;
                    p7 = p[(j == 0) ? 0 : -1] > 0 ? 1 : 0;
                    p8 = p[(i == 0 || j == 0) ? 0 : -step - 1] > 0 ? 1 : 0;
                    //判断8 simple
                    int is8simple = 1;
                    //以下6种情况，当p0=0时，不会改变8连通性
                    if (p1 == 0 && p5 == 0 && (p8 == 1 || p7 == 1 || p6 == 1) && (p2 == 1 || p3 == 1 || p4 == 1)) {
                        is8simple = 0;
                    }
                    if (p3 == 0 && p7 == 0 && (p8 == 1 || p1 == 1 || p2 == 1) && (p4 == 1 || p5 == 1 || p6 == 1)) {
                        is8simple = 0;
                    }
                    if (p7 == 0 && p1 == 0 && p8 == 1 && (p2 == 1 || p3 == 1 || p4 == 1 || p5 == 1 || p6 == 1)) {
                        is8simple = 0;
                    }
                    if (p3 == 0 && p1 == 0 && p2 == 1 && (p4 == 1 || p5 == 1 || p6 == 1 || p7 == 1 || p8 == 1)) {
                        is8simple = 0;
                    }
                    if (p7 == 0 && p5 == 0 && p6 == 1 && (p2 == 9 || p1 == 1 || p2 == 1 || p3 == 1 || p4 == 1)) {
                        is8simple = 0;
                    }
                    if (p3 == 0 && p5 == 0 && p4 == 1 && (p6 == 1 || p7 == 1 || p8 == 1 || p1 == 1 || p2 == 1)) {
                        is8simple = 0;
                    }
                    int adjsum;
                    adjsum = p1 + p2 + p3 + p4 + p5 + p6 + p7 + p8;
                    if (adjsum != 1 && adjsum != 0 && is8simple == 1) { //判断是否为孤立点或者端点
                        dst.at<uchar>(i, j) = 0; //满足删除条件，设当前像素为0
                        ifEnd = true;
                    }
                }
            }
        }
        if (!ifEnd) break; //若没有可以删除的像素点，则退出循环
    }
}

void main() {
    cv::Mat src = cv::imread("C:\\Users\\PC\\Desktop\\cv\\src.jpg");
    cv::cvtColor(src, src, cv::COLOR_BGR2GRAY); //将原图像转换为灰度图
    cv::threshold(src, src, 128, 255, cv::THRESH_BINARY); //二值化图像

    //图像细化
    cv::Mat dst;
    Rosenfeld_Thin(src, dst);
    //显示图像
    cv::imshow("src", src);
    cv::imshow("dst", dst);
    cv::waitKey(0);
}
```
## 效果

![原图](https://upload-images.jianshu.io/upload_images/22192996-82ee3f6c5cf09ba4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Rosenfeld算法细化](https://upload-images.jianshu.io/upload_images/22192996-ab395c9630b6adfc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
