## 前 言
图像细化是图像处理中一种常用的处理手段，常见的细化算法有：Hilditch算法、Rosenfeld算法、Pavlidis算法、Zhang-Suen算法以及基于索引表查询的细化算法等等。本文主要介绍Hilditch算法以及C++的实现。

## 算法原理
Hilditch算法的核心思想通过循环遍历图像像素，根据规则判断像素是否删除，从而来达到细化图像的目的。具体原理可以参考一下：  
https://www.pianshen.com/article/2778260406/  
https://www.cnblogs.com/mikewolf2002/p/3327183.html  
这里不再赘述  

## 算法步骤
参考算法原理，可以将Hilditch算法分为一下步骤：   
首先，Hilditch算法是基于二值化图像的基础上进行的，设当前被处理的像素为p0，使用以下所示方式来表示当前像素的八邻域
```
p8  p1  p2
p7  p0  p3
p6  p5  p4
```
对此，为了方便算法过程的实现，设置b[i]=1(i=0…8)，来表示八邻域的像素值，即：
```
b8  b1  b2
b7  b0  b3
b6  b5  b4
```
算法过程中若像素值为255(白色)，b[i] = 1；若像素值为0(黑色)，b[i] = 0；像素值为128(灰色，表示该像素点在上一次的遍历中被标记为待删除像素)，b[i] = -1  

遍历图像的每一个点，当同时满足一下6个条件时，将当前像素设为待删除像素：  
1. p0的像素值为255(b[0] = 1)，即当前像素为前景点  
2. p0轴向邻域的像素值不全为255(b[1]，b[3]，b[5]，b[7]中至少有一个等于0)，即当前像素为边界点  
3. p0的八邻域中至少有两个的像素值为255，即当前像素不是端点也不是孤立点  
4. p0的八邻域中未被标记为待移除像素且像素值为255的至少有一个，即删除待删除像素后，当前像素变成孤立点(由于此条件对细化的结果基本没影响，很多文章在介绍时直接去掉了)  
5. p0的八邻域连接数为1，即Nc(p) = 1  
6. 对于第6个条件有两种说法：  
**(1)** 如果p1和p3已经标记为待删除像素，分别用b1 = 0和b3 = 0的值重新计算八邻域连接数且要等于1  
**(2)** p0的八邻域必须满足要么不是待删除点；要么记为待删除，但b[i] = 0时，p0的八邻域连接数为1  

**注：** 对于条件6，测试发现第二种说法细化出来的效果更好，第一种会出现图像断裂现象。  
当所有像素都扫描一遍后，完成一次循环。此时将设为待删除像素(灰色128)的值设为0，即删除。无限循环，直至上一次循环待删除像素的个数为零，退出循环。  

## 算法实现
首先根据八邻域连接数公式，实现函数  

![八邻域连接数公式](https://upload-images.jianshu.io/upload_images/22192996-666445892e527a2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
//八邻域连接数函数
//根据b[i]的值，定义d[i]，如果b[i]等于0，则d[i]=0；否则d[i]=1
int func_NC8(int* b) {
    int i, j, sum, d[10];

    for (i = 0; i <= 9; i++) {
        j = i;
        if (i == 9) j = 1;
        if (*(b + j) == 0) {
            d[i] = 0;
        } else {
            d[i] = 1;
        }
    }

    sum = 0;
    for (i = 1; i <= 4; i++) {
        sum = sum + (d[2 * i - 1] - d[2 * i - 1] * d[2 * i] * d[2 * i + 1]);
    }
    return (sum);
}
```
根据上述的算法步骤，整体代码如下
```
#include <iostream>
#include <vector>
#include <opencv2/opencv.hpp>
#include <opencv2/core/core.hpp>

using namespace std;

//八邻域连接数函数
int func_NC8(int* b) {
    int i, j, sum, d[10];

    for (i = 0; i <= 9; i++) {
        j = i;
        if (i == 9) j = 1;
        if (*(b + j) == 0) {
            d[i] = 0;
        } else {
            d[i] = 1;
        }
    }
    sum = 0;
    for (i = 1; i <= 4; i++) {
        sum = sum + (d[2 * i - 1] - d[2 * i - 1] * d[2 * i] * d[2 * i + 1]);
    }
    return (sum);
}

void Hilditch_Thin(cv::Mat& src, cv::Mat& dst) {
    if (src.type() != CV_8UC1) {
        cout << "只处理二值化图像，请先二值化图像" << endl;
        return;
    }

    dst = src.clone();

    int offset[9][2] = { {0,0},{0,-1},{1,-1},{1,0},{1,1},{0,1},{-1,1},{-1,0},{-1,-1} }; //八邻域的偏移量
    int px, py;
    int b[9]; //八邻域灰度信息
    int counts; //待删除像素计数
    int sum; //条件计数

    uchar* img = dst.data;;
    int width = dst.cols;
    int height = dst.rows;
    int step = dst.step;
    do{
        counts = 0;
        for (int y = 0; y < height; y++) {
            for (int x = 0; x < width; x++) {
                for (int i = 0; i < 9; i++) { //根据像素值，初始化灰度信息b[i]
                    b[i] = 0;
                    px = x + offset[i][0];
                    py = y + offset[i][1];
                    if (px >= 0 && px < width && py >= 0 && py < height) {
                        if (img[py * step + px] == 255) {
                            b[i] = 1;
                        } else if (img[py * step + px] == 128) {
                            b[i] = -1;
                        }
                    }
                }

                //条件1
                if (b[0] != 1) continue;

                //条件2
                if(abs(b[1]) + abs(b[3]) + abs(b[5]) + abs(b[7]) == 4) continue;

                //条件3
                if(abs(b[1]) + abs(b[2]) + abs(b[3]) + abs(b[4]) + abs(b[5]) + abs(b[6]) + abs(b[7]) + abs(b[8]) < 2) continue;

                //条件4
                //此条件不影响最后的细化结果，先注释
               /* sum = 0;
                for (i = 1; i <= 8; i++) {
                    if (b[i] == 1) sum++;
                }
                if (sum == 0) continue;*/

                //条件5
                if (func_NC8(b) != 1) continue;
                
                //条件6
                //第一种说法：
                /*if (b[1] == -1){
                    b[1] = 0;
                    if (func_NC8(b) != 1) continue;
                    b[1] = -1;
                }
                if (b[3] == -1){
                    b[3] = 0;
                    if (func_NC8(b) != 1) continue;
                    b[3] = -1;
                }*/
                //第二种说法：
                sum = 0;
                for (int i = 1; i <= 8; i++) {
                    if (b[i] != -1) {
                        sum++;
                    } else {
                        b[i] = 0;
                        if (func_NC8(b) == 1) sum++;
                        b[i] = -1;
                    }
                }
                if (sum != 8) continue;

                img[y * step + x] = 128;
                counts++;
            }
        }

        //删除待删除像素
        if (counts != 0) {
            for (int y = 0; y < height; y++) {
                for (int x = 0; x < width; x++) {
                    if (img[y * step + x] == 128)
                        img[y * step + x] = 0;
                }
            }
        }

    } while (counts != 0);
}

void main() {
    cv::Mat src = cv::imread("C:\\Users\\PC\\Desktop\\image\\test.jpg");
    cv::Mat dst;
    cv::cvtColor(src, src, cv::COLOR_BGR2GRAY); //转灰度图

    //将原图像转换为二值图像
    cv::threshold(src, src, 128, 255, cv::THRESH_BINARY);
    //cv::adaptiveThreshold(src, src, 255, cv::ADAPTIVE_THRESH_GAUSSIAN_C, cv::THRESH_BINARY, 3, 1);

    //图像细化
    Hilditch_Thin(src, dst);

    //显示结果
    cv::imshow("src", src);
    cv::imshow("dst", dst);
    cv::waitKey(0);
}
```
细化效果：

![原图](https://upload-images.jianshu.io/upload_images/22192996-c2c61a86871f68c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![细化结果](https://upload-images.jianshu.io/upload_images/22192996-0f1d402c84f90e48.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

条件6使用第一种说法的细化结果：

![结果](https://upload-images.jianshu.io/upload_images/22192996-fc3507e49e146f06.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

很明显图像出现了断裂，效果没有第二种好(ps:若代码有问题，欢迎大佬指正)