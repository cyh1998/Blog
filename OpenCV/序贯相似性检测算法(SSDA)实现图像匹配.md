## 简 介
算法的原理，可以参考这个篇博客：[基于灰度的模板匹配算法](https://blog.csdn.net/hujingshuang/article/details/47759579)。这里不再赘述，本文使用序贯相似性检测算法(下面简称：SSDA)，实现对图像模板的匹配

## 算法思路
参考博客，可以整理出SSDA算法的整体思路：  
1. 确定子图的左上角坐标范围
2. 遍历所有可能的子图
3. 求模板图和子图的图像均值
4. 随机取不重复的点，求模板图和子图中该点的值与其自身图像均值的绝对误差
5. 累加绝对误差，直到超过阈值
6. 计算累加次数，则次数最多的子图为匹配的最终图像

## 代码
根据算法思路，demo如下：
```
import time
import random
import cv2
import numpy as np

def Image_Mean_Value(Image):
    res = np.sum((Image.astype("float"))) / float(Image.shape[0] * Image.shape[1])
    return res

def Image_SSDA(search_img, example_img):
    # 确定子图的范围
    M = search_img.shape[0]
    m = example_img.shape[0]
    N = search_img.shape[1]
    n = example_img.shape[1]
    Range_x = M - m - 1
    Range_y = N - n - 1
    search_img = cv2.cvtColor(search_img, cv2.COLOR_RGB2GRAY)
    example_img = cv2.cvtColor(example_img, cv2.COLOR_RGB2GRAY)
    # 求模板图的均值
    example_MV = Image_Mean_Value(example_img)
    R_count_MAX = 0
    Best_x = 0
    Best_y = 0
    # 遍历所有可能的子图
    for i in range(Range_x):
        for j in range(Range_y):
            R_count = 0
            Absolute_error_value = 0
            # 从搜索图中截取子图
            subgraph_img = search_img[i:i+m, j:j+n]
            # 求子图的均值
            search_MV = Image_Mean_Value(subgraph_img)
            # 判断是否超过阈值
            while Absolute_error_value < SSDA_Th:
                R_count += 1
                # 取随机的像素点
                x = random.randint(0, m - 1)
                y = random.randint(0, n - 1)
                pv_s = subgraph_img[x][y]
                pv_e = example_img[x][y]
                # 计算绝对误差值并累加
                Absolute_error_value += abs((pv_e - example_MV) - (pv_s - search_MV))
            # 将次数最多的子图为匹配图像
            if R_count >= R_count_MAX:
                R_count_MAX = R_count
                Best_x = i
                Best_y = j
    # 返回最匹配图像的坐标
    return Best_x, Best_y

if __name__ == '__main__':
    # 原图路径
    srcImg_path = "C:\\Users\\PC\\Desktop\\SSDA\\src.jpg"
    # 搜索图像路径
    searchImg_path = "C:\\Users\\PC\\Desktop\\SSDA\\find.jpg"
    # SSDA算法阈值
    SSDA_Th = 10

    src_img = cv2.imread(srcImg_path)
    search_img = cv2.imread(searchImg_path)

    start = time.perf_counter()
    Best_x, Best_y = Image_SSDA(src_img, search_img)
    end = time.perf_counter()
    print("time:", end - start)

    cv2.rectangle(src_img, (Best_y, Best_x), (Best_y + search_img.shape[1], Best_x + search_img.shape[0]), (0, 0, 255), 3)
    cv2.imshow("src_img", src_img)
    cv2.imshow("search_img", search_img)
    cv2.waitKey(0)
```
说明：  
- 阈值的设定：一开始将阈值设置的较大一些，保证算法的准确度；在多次测试后可以将阈值降低到合适的值。阈值越大，算法的精度约高，耗时也相对的会增加一些
- 随机像素点的选取：算法中指出，应当选取不重复的像素点，代码中直接使用了`random.randint()` 来生成随机数，是会有重复的可能，但是概率不大，这里近似于不重复

## 结果
运行代码，匹配结果如下：

![SSDA算法结果](https://upload-images.jianshu.io/upload_images/22192996-29d055b37c4db9bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 总结
算法中指出，SSDA算法是对传统模板匹配算法的改进，比传统匹配算法快几十到几百倍。又上文可知，SSDA算法匹配耗时18秒，我们使用传统的SSD算法来实现相同的功能来测试下耗时。SSD算法的实现可以参考：[基于SSD的图像匹配算法](./基于SSD的图像匹配算法.md)  

结果如下：  

![SSD算法结果](https://upload-images.jianshu.io/upload_images/22192996-7c29ad848dfd76b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到两者都达到了较好的匹配效果，但SSD算法耗时28秒，确实SSDA算法的速度优于SSD算法
