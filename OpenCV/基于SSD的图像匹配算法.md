## 简 介
常见的图像匹配算法有：平均绝对差算法（MAD）、绝对误差和算法（SAD）、误差平方和算法（SSD）、平均误差平方和算法（MSD）、归一化积相关算法（NCC）、序贯相似性检测算法（SSDA）、hadamard变换算法（SATD）  
本文主要介绍基于误差平方和算法（SSD）的图像匹配

## SSD算法
误差平方和算法（Sum of Squared Differences，简称SSD算法），也叫差方和算法。即计算子图与模板图的L2距离。公式如下，这里不再赘述。

![Sum of Squared Differences](https://upload-images.jianshu.io/upload_images/22192996-df66300095cc302b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 代 码
根据公式，很容易写出代码，配合OpenCV可以实现图像模板匹配的功能
demo如下：
```
import sys
import cv2
import numpy as np

def Image_SSD(src_img, search_img):
    # 1.确定子图的范围
    # 2.遍历子图
    # 3.求模板图和子图的误差平方和
    # 4.返回误差平方和最小的子图左上角坐标
    M = src_img.shape[0]
    m = search_img.shape[0]
    N = src_img.shape[1]
    n = search_img.shape[1]
    Range_x = M - m - 1
    Range_y = N - n - 1
    src_img = cv2.cvtColor(src_img, cv2.COLOR_RGB2GRAY)
    search_img = cv2.cvtColor(search_img, cv2.COLOR_RGB2GRAY)
    min_res = sys.maxsize
    Best_x = 0
    Best_y = 0
    for i in range(Range_x):
        for j in range(Range_y):
            subgraph_img = src_img[i:i+m, j:j+n]
            res = np.sum((search_img.astype("float") - subgraph_img.astype("float")) ** 2) # SSD公式
            if res < min_res:
                min_res = res
                Best_x = i
                Best_y = j
    return Best_x, Best_y

if __name__ == '__main__':
    # 原图路径
    srcImg_path = "C:\\Users\\PC\\Desktop\\SSD\\src.jpg"
    # 搜索图像路径
    searchImg_path = "C:\\Users\\PC\\Desktop\\SSD\\search.jpg"

    src_img = cv2.imread(srcImg_path)
    search_img = cv2.imread(searchImg_path)
    Best_x, Best_y = Image_SSD(src_img, search_img)

    cv2.rectangle(src_img, (Best_y, Best_x), (Best_y + search_img.shape[1], Best_x + search_img.shape[0]), (0, 0, 255), 3)
    cv2.imshow("src_img", src_img)
    cv2.imshow("search_img", search_img)
    cv2.waitKey(0)
```
## 结 果
运行代码，可以看到匹配到的最终效果

![匹配结果](https://upload-images.jianshu.io/upload_images/22192996-7919bf60f72203ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 引 申
此外的平均绝对差算法（MAD）、绝对误差和算法（SAD）、平均误差平方和算法（MSD）和SSD算法如出一辙，只是其相似度测量公式不同  

**1.平均绝对差算法（MAD）**  

![Mean Absolute Differences](https://upload-images.jianshu.io/upload_images/22192996-777dde2b4dcbd63b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**2.绝对误差和算法（SAD）**  

![Sum of Absolute Differences](https://upload-images.jianshu.io/upload_images/22192996-b462c4fea0df57d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**3.平均误差平方和算法（MSD）**  

![Mean Square Differences](https://upload-images.jianshu.io/upload_images/22192996-f0f52715f0348404.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
