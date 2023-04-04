## 前 言
本文介绍图像细化算法之** ZhangSuen细化算法**。

## 算法步骤
设当前被处理的像素为p0，使用以下所示方式来表示当前像素的八邻域，背景像素值为0(黑色)，前景像素值为255(白色)，为了方便计算，在算法过程中将前景像素值当作1来计算。
```
p8  p1  p2
p7  p0  p3
p6  p5  p4
```
**第一步** ：循环所有的前景像素点，删除满足如下条件的像素点：  
(1) 2 ≤ N(p0) ≤ 6：即p0的八邻域像素点，为前景像素点的个数  
(2) S(p0) = 1：即p1-p2-p3-p4-p5-p6-p7-p8-p1按序前后两个像素值分别为0、1的个数，例如：  
```
1  1   0
0  p0  1
1  0   1
=> 1-0-1-1-0-1-0-1-1，其中0-1的个数为3
```
则S(p0) = 3  
(3) p1 * p3 * p5 = 0  
(4) p3 * p5 * p7 = 0  

**第二步** ：循环所有的前景像素点，删除满足如下条件的像素点：  
(1) 2 ≤ N(p0) ≤ 6  
(2) S(p0) = 1  
(3) p1 * p3 * p7 = 0  
(4) p1 * p5 * p7 = 0  

详细的算法原理解释，可以参考：https://blog.csdn.net/weixinhum/article/details/80261296

## 算法实现
根据上文的算法步骤，实现的代码如下：
```
#include <iostream>
#include <vector>
#include <opencv2/opencv.hpp>

using namespace std;

void ZhangSuen_Thin(cv::Mat& src, cv::Mat& dst)
{
    if (src.type() != CV_8UC1) {
        cout << "只处理二值化图像，请先二值化图像" << endl;
        return;
    }

    src.copyTo(dst);
    int width = src.cols;
    int height = src.rows;
    int step = dst.step;
    bool ifEnd;
    int p1, p2, p3, p4, p5, p6, p7, p8; //八邻域
    vector<uchar*> flag; //用于标记待删除的点
    uchar* img = dst.data;

    while (true) {
        ifEnd = false;
        for (int i = 0; i < height; ++i) {
            for (int j = 0; j < width; ++j) {
                uchar* p = img + i * step + j;
                if (*p == 0) continue; //如果不是前景点,跳过
                //判断八邻域像素点的值(要考虑边界的情况),若为前景点(白色255),则为1;反之为0
                p1 = p[(i == 0) ? 0 : -step] > 0 ? 1 : 0;
                p2 = p[(i == 0 || j == width - 1) ? 0 : -step + 1] > 0 ? 1 : 0;
                p3 = p[(j == width - 1) ? 0 : 1] > 0 ? 1 : 0;
                p4 = p[(i == height - 1 || j == width - 1) ? 0 : step + 1] > 0 ? 1 : 0;
                p5 = p[(i == height - 1) ? 0 : step] > 0 ? 1 : 0;
                p6 = p[(i == height - 1 || j == 0) ? 0 : step - 1] > 0 ? 1 : 0;
                p7 = p[(j == 0) ? 0 : -1] > 0 ? 1 : 0;
                p8 = p[(i == 0 || j == 0) ? 0 : -step - 1] > 0 ? 1 : 0;
                if ((p1 + p2 + p3 + p4 + p5 + p6 + p7 + p8) >= 2 && (p1 + p2 + p3 + p4 + p5 + p6 + p7 + p8) <= 6) //条件1
                {
                    //条件2的计数
                    int count = 0;
                    if (p1 == 0 && p2 == 1) ++count;
                    if (p2 == 0 && p3 == 1) ++count;
                    if (p3 == 0 && p4 == 1) ++count;
                    if (p4 == 0 && p5 == 1) ++count;
                    if (p5 == 0 && p6 == 1) ++count;
                    if (p6 == 0 && p7 == 1) ++count;
                    if (p7 == 0 && p8 == 1) ++count;
                    if (p8 == 0 && p1 == 1) ++count;
                    if (count == 1 && p1 * p3 * p5 == 0 && p3 * p5 * p7 == 0) { //条件2、3、4
                        flag.push_back(p); //将当前像素添加到待删除数组中
                    }
                }
            }
        }
        //将标记的点删除    
        for (vector<uchar*>::iterator i = flag.begin(); i != flag.end(); ++i) {
            **i = 0;
            ifEnd = true;
        }
        flag.clear(); //清空待删除数组

        for (int i = 0; i < height; ++i) {
            for (int j = 0; j < width; ++j) {
                uchar* p = img + i * step + j;
                if (*p == 0) continue;
                p1 = p[(i == 0) ? 0 : -step] > 0 ? 1 : 0;
                p2 = p[(i == 0 || j == width - 1) ? 0 : -step + 1] > 0 ? 1 : 0;
                p3 = p[(j == width - 1) ? 0 : 1] > 0 ? 1 : 0;
                p4 = p[(i == height - 1 || j == width - 1) ? 0 : step + 1] > 0 ? 1 : 0;
                p5 = p[(i == height - 1) ? 0 : step] > 0 ? 1 : 0;
                p6 = p[(i == height - 1 || j == 0) ? 0 : step - 1] > 0 ? 1 : 0;
                p7 = p[(j == 0) ? 0 : -1] > 0 ? 1 : 0;
                p8 = p[(i == 0 || j == 0) ? 0 : -step - 1] > 0 ? 1 : 0;
                if ((p1 + p2 + p3 + p4 + p5 + p6 + p7 + p8) >= 2 && (p1 + p2 + p3 + p4 + p5 + p6 + p7 + p8) <= 6)
                {
                    int count = 0;
                    if (p1 == 0 && p2 == 1) ++count;
                    if (p2 == 0 && p3 == 1) ++count;
                    if (p3 == 0 && p4 == 1) ++count;
                    if (p4 == 0 && p5 == 1) ++count;
                    if (p5 == 0 && p6 == 1) ++count;
                    if (p6 == 0 && p7 == 1) ++count;
                    if (p7 == 0 && p8 == 1) ++count;
                    if (p8 == 0 && p1 == 1) ++count;
                    if (count == 1 && p1 * p3 * p7 == 0 && p1 * p5 * p7 == 0) {
                        flag.push_back(p);
                    }
                }
            }
        }
        //将标记的点删除    
        for (vector<uchar*>::iterator i = flag.begin(); i != flag.end(); ++i) {
            **i = 0;
            ifEnd = true;
        }
        flag.clear();

        if (!ifEnd) break; //若没有可以删除的像素点，则退出循环
    }
}

void main() {
    cv::Mat src = cv::imread("C:\\Users\\PC\\Desktop\\cv\\src.jpg");
   
    cv::cvtColor(src, src, cv::COLOR_BGR2GRAY);
    //将原图像转换为二值图像
    cv::threshold(src, src, 128, 255, cv::THRESH_BINARY);

    //图像细化
    cv::Mat dst;
    ZhangSuen_Thin(src, dst);

    //显示图像
    cv::imshow("src", src);
    cv::imshow("dst", dst);
    cv::waitKey(0);
}
```
## 效 果

![原图](https://upload-images.jianshu.io/upload_images/22192996-57242b764c892a84.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![ZhangSuen算法细化](https://upload-images.jianshu.io/upload_images/22192996-47d5194d8ea535c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


