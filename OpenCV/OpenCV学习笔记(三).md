## 一、透明背景转白色
我们经常会处理一些透明背景的png格式图片，可以使用下面的demo实现将透明背景转白色
```
cv::Mat alpha2white_opencv2(cv::Mat& src) {
    for (int i = 0; i < src.rows; ++i) {
        for (int j = 0; j < src.cols; ++j) {
            if (src.at<cv::Vec4b>(j, i)[3] != 255) {
                src.at<cv::Vec4b>(j, i) = cv::Vec4b(255, 255, 255, 255);
            }
        }
    }
    return src;
}
```
在读取图像信息，切记要将flags置为-1，即读取alpha通道的值
```
cv::Mat src = cv::imread("C:\\Users\\PC\\Desktop\\cv\\test.png"，-1);
```
效果：

![透明背景转白色](https://upload-images.jianshu.io/upload_images/22192996-eb64afda8017f25b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 二、使用grabCut实现图像前景分割
可以使用 `grabCut()` 来分割出图像前景，函数原型：
```
void grabCut( InputArray img, InputOutputArray mask, Rect rect,
                           InputOutputArray bgdModel, InputOutputArray fgdModel,
                           int iterCount, int mode = GC_EVAL );
```
参数说明：  
`img` ：待分割的源图像，必须是8位3通道（CV_8UC3）图像  
`mask` ：输入输出掩码图像，保存处理后的结果，8位单通道掩码。mask元素值只能为以下四种值：  
- GCD_BGD（=0），背景；
- GCD_FGD（=1），前景；
- GCD_PR_BGD（=2），可能的背景；
- GCD_PR_FGD（=3），可能的前景。

如果没有手动标记GCD_BGD或者GCD_FGD，那么结果只会有GCD_PR_BGD或GCD_PR_FGD  
`rect` ：进行分割的图像范围，即只有该矩形范围内的图像部分才会被处理  
`bgdModel` ：背景模型  
`fgdModel` ：前景模型  
`iterCount` ：函数迭代次数  
`mode` ：用于指示grabCut函数进行什么操作，可选的值有：  
- GC_INIT_WITH_RECT（=0），用矩形窗初始化GrabCut；
- GC_INIT_WITH_MASK（=1），用掩码图像初始化GrabCut；
- GC_EVAL（=2），执行分割。
- GC_EVAL_FREEZE_MODEL （=3），用固定的模型，只迭代一次grabCut

简单的示例：
```
#include <iostream>
#include <vector>
#include <opencv2/opencv.hpp>

void main() {
    cv::Mat src = cv::imread("C:\\Users\\PC\\Desktop\\cv\\hua.jpg");
    //src = alpha2white_opencv2(src);

    cv::Rect rect(0, 0, src.rows, src.cols);
    cv::Mat result; //输出结果
    cv::Mat bgModel, fgModel; //两个临时矩阵变量，作为算法的中间变量使用

    cv::grabCut(src, result, rect, bgModel, fgModel, 5, cv::GC_INIT_WITH_RECT);
    
    cv::compare(result, cv::GC_PR_FGD, result, cv::CMP_EQ); //比较result的值，将可能的前景像素输出到result中
    cv::Mat foreground(src.size(), CV_8UC3, cv::Scalar(0, 0, 0)); //自定义黑色输出背景
    src.copyTo(foreground, result); //将输出结果result拷贝到背景foreground中
    
    cv::imshow("res", foreground); //显示最终结果
    cv::waitKey(0);
}
```
执行结果：

![原图](https://upload-images.jianshu.io/upload_images/22192996-7795be8315769465.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![处理结果](https://upload-images.jianshu.io/upload_images/22192996-b02233aaf8bc9f08.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

迭代次数越多，效果越好，耗时也越长