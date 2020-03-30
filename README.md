# SeetaFace 入门教程
-----
## 0. 目录
[TOC]

----
## 1. 前言

该文档通过`leanote`软件编辑，可以去[博客](http://leanote.com/blog/post/5e7d6cecab64412ae60016ef)获取更好的阅读体验。

`SeetaFace6` 是中科视拓技术开发体系最新的版本，该版本为开放版，免费供大家使用。该版本包含了人脸识别、活体检测、属性识别、质量评估模块。

本文档基于`SeetaFace`进行讲解，同时也会给出一些通用的算法使用的结论和建议，希望能够给使用`SeetaFace`或者使用其他人脸识别产品的读者提供帮助。

本文档不会提及如何进行代码编译和调试，会对一些关键的可能产生问题的部分进行提醒，开发者可以根据提醒查看是否和本身遇到的问题相符合。

本文档适用于有一定C++基础，能够看懂核心代码片段，并且将代码完成并在对应平台上编译出可执行程序。

其他的预备知识包含`OpenCV`库的基本使用，和一些基本的图像处理的了解。以上都是一个接口的调用，没有使用过也不影响对文档的理解。本书中使用的`OpenCV`库为`2`或`3`版本。

如果您是对C++语言本身存在疑问，而对于库调用已经有经验的话，本文档可能不能直接帮助您。建议先将C++语言作为本文档的前置课程进行学习。同时本文档也会尽可能照顾刚刚接触C++语言的读者。

为了避免过长的代码段影响大家的阅读，因此会把一些必须展示的代码块当道`附录`的`代码块`节。通过这些完整的代码块以便读者可以只根据文档陈列的内容就可以理解`SeetaFace`的基本设定和原理。当然不深入这些繁杂的代码块，也不会完全影响理解和使用。

为了表达的简洁，文档中出现的`代码块`在保留了语义一致性的基础上，还可能经过了删减和调整。实际实现部分的代码，以最终下载的代码为准。

为了照顾不同层次的理解，会对一些基本概念作出解释，一些已经在图像处理或者AI领域的开发者对概念熟悉的读者可以略过相关章节，当理解有偏差时再详细阅读。

因为文档中出现了很多代码，必然要使用英文单词。在文档解释中使用的是中文关键字。限于作者的能力，可能没有办法对文档中出现的所有英文关键字和中文关键字做出一一解释对应。好在代码中使用的英文单词为常用单词，在不影响理解的前提下，其直译的中文含义就是对应上下文中的中文关键字。

写在最后，阅读本文档之前最好先处理好编译的环境，遇到问题直接编译运行。

----
## 2. 基本概念

----
### 2.1 图像

----
#### 2.1.1 结构定义

作为图像处理的C++库，图像存储是一个基本数据结构。
接口中图像对象为`SeetaImageData`。

```cpp
struct SeetaImageData
{
    int width;              // 图像宽度
    int height;             // 图像高度
    int channels;           // 图像通道
    unsigned char *data;    // 图像数据
};
```

这里要说明的是`data`的存储格式，其存储的是连续存放的试用8位无符号整数表示的像素值，存储为`[height, width, channels]`顺序。彩色图像时三通道以`BGR`通道排列。
如下图所示，就是展示了高4宽3的彩色图像内存格式。

![Image Layout](leanote://file/getImage?fileId=5e7eb2acab64412ae600565c)

该存储在内存中是连续存储。因此，上图中表示的图像`data`的存储空间为`4*3*3=36` `bytes`。

> 提示： `BGR`是`OpenCV`默认的图像格式。在大家很多时候看到就是直接将`cv::Mat`的`data`直接赋值给`SeetaImageData`的`data`。

`data`的数据类型为`uint8`，数值范围为`[0, 255]`，表示了对应的灰度值由最暗到最亮。在一些用浮点数  `float`表示灰度值的平台上，该范围映射到为[0, 1]。

这里详细描述格式是为了对接不同应用场景下的图像库的图像表述格式，这里注意的两点，1. 数据类型是`uint8`表示的`[0, 255]`范围；2. 颜色通道是`BGR`格式，而很多图像库常见的是`RGB`或者`RGBA`(`A`为`alpha`不透明度)。

> 提示：颜色通道会影响识别算法的精度，在不同库的转换时需要小心注意。

这种纯C接口不利于一些资源管理的情况。SeetaFace 提供给了对应`SeetaImageData`的封装`seeta::ImageData`和`seeta::cv::ImageData`。

----
#### 2.1.2 使用示例

以下就是使用`OpenCV`加载图像后转换为`SeetaImageData`的操作。

```cpp
cv::Mat cvimage = cv::imread("1.jpg", cv::IMREAD_COLOR);

SeetaImageData simage;
simage.width = cvimage.cols;
simage.height = cvimage.rows;
simage.channels = cvimage.channels();
simage.data = cvimage.data;
```

> 注意：`simage.data`拿到的是“借来的”临时数据，在试用的期间，必须保证`cvimage`的对象不被释放，或者形式的更新。
> 注意：原始图像可能是空的，为了输入的合法性，需要通过`cvimage.empty()`方法，判断图像是否为空。

这里通过`cv::imread`并且第二个参数设置为`cv::IMREAD_COLOR`，获取到的`cv::Mat`的图像数据是连续存储的，这个是使用`SeetaImageData`必须的。如果不确定是否是连续存储的对象，可以调用下述代码段进行转换。

```cpp
if (!cvimage.isContinuous()) cvimage = cvimage.clone();
```

当然，根据`cv::Mat`和`SeetaImageData`，对象的转换可以逆向。

```cpp
cv::Mat another_cvimage = cv::Mat(simage.height, simage.width, CV_8UC(simage.channels), simage.data);
```

`seeta::ImageData`和`seeta::cv::ImageData`也是基于这种基本的类型定义进行的封装，并加进了对象声明周期的管理。

这里展示了一种封装：

```cpp
namespace seeta
{
    namespace cv
    {
        // using namespace ::cv;
        class ImageData : public SeetaImageData {
        public:
            ImageData( const ::cv::Mat &mat )
                : cv_mat( mat.clone() ) {
                this->width = cv_mat.cols;
                this->height = cv_mat.rows;
                this->channels = cv_mat.channels();
                this->data = cv_mat.data;
            }
        private:
            ::cv::Mat cv_mat;
        };
    }
}
```

这样`SeetaImageData`使用代码就可以简化为：

```cpp
seeta::cv::ImageData = cv::imread("1.jpg");
```

因为`seeta::cv::ImageData`继承了`SeetaImageData`，因此在需要传入`const SeetaImageData &`类型的地方，可以直接传入`seeta::cv::ImageData`对象。

> 参考：`CStruct.h`、`Struct.h` 和 `Struct_cv.h`。

----
### 1.2 坐标系

----
#### 1.2.1 结构定义

我们在表示完图像之后，我们还需要在图像上表示一些几何图形，例如矩形和点，这两者分别用来表示人脸的位置和人脸关键点的坐标。

这我们采用的屏幕坐标系，也就是以左上角为坐标原点，屏幕水平向右方向为横轴，屏幕垂直向下方向为纵轴。坐标以像素为单位。

![Screen Coordinates](leanote://file/getImage?fileId=5e7eb2adab64412ae600565d)

坐标值会出现负值，并且可以为实数，分别表示超出屏幕的部分，和位于像素中间的坐标点。

有了以上的基本说明，我们就可以给出矩形、坐标点或其他的定义。

```cpp
/**
 * 矩形
 */
struct SeetaRect
{
    int x;      // 左上角点横坐标
    int y;      // 左上角点纵坐标
    int width;  // 矩形宽度
    int height; // 矩形高度
};
/**
 *宽高大小
 */
struct SeetaSize
{
    int width;  // 宽度
    int height; // 高度
};
/**
 * 点坐标
 */
struct SeetaPointF
{
    double x;   // 横坐标
    double y;   // 纵坐标
};
```

> 注意：从表达习惯上，坐标和形状大小的表示，通常是横坐标和宽度在前，这个和图片表示的时候内存坐标是高度在前的表示有差异，在做坐标变换到内存偏移量的时候需要注意。

----
#### 1.2.2 使用示例

这种基本数据类型的定义，在拿到以后就确定的使用方式，这里距离说明，如果拿到这种图片，如何绘制到图片上。

首先我们假定图像、矩形和点的定义。

```cpp
cv::Mat canvas = cv::imread("1.jpg");
SeetaRect rect = {20, 30, 40, 50};
SeetaPointF point = {40, 55};
```

当变量已经具有了合法含义的时候，就可以进行对应绘制。

```cpp
// 绘制矩形
cv::rectangle(canvas, cv::Rect(rect.x, rect.y, rect.width, rect.height), CV_RGB(128, 128, 255), 3);
// 绘制点
cv::circle(canvas, cv::Point(point.x, point.y), 2, CV_RGB(128, 255, 128), -1);
```

> 参考：`CStruct.h`和`Struct.h`。

### 1.3 模型设置

在算法开发包中，除了代码库本身以外，还有数据文件，我们通常称之为模型。

这里给出`SeetaFace`的模型设置的基本定义：

```cpp
enum SeetaDevice
{
    SEETA_DEVICE_AUTO = 0,
    SEETA_DEVICE_CPU  = 1,
    SEETA_DEVICE_GPU  = 2,
};

struct SeetaModelSetting
{
    enum SeetaDevice device;
    int id; // when device is GPU, id means GPU id
    const char **model; // model string terminate with nullptr
};
```

这里要说明的是`model`成员，是一种C风格的数组，以`NULL`为结束符。一种典型的初始化为，设定模型为一个`fr_model.csta`，运行在`cpu:0`上。

```cpp
SeetaModelSetting setting;
setting.device = SEETA_DEVICE_CPU;
setting.id = 0;
setting.model = {"fr_model.csta", NULL};
```

当然`SeetaFace`也对应封装了`C++`版本设置方式，继承关系为，详细的实现见`附录`中的`代码块`：
```cpp
class seeta::ModelSetting : SeetaModelSetting;
```

实现代码见`Struct.h`，使用方式就变成了如下代码段，默认在`cpu:0`上运行。

```cpp
seeta::ModelSetting setting;
setting.append("fr_model.csta");
```

有了通用模型的设置方式，一般实现时的表述就会是`加载一个人脸识别模型`，模型加载的方式都是构造`SeetaModelSetting`，传入对应的算法对象。

> 参考：`CStruct.h`和`Struct.h`。

### 1.4 对象生命周期

对象生命周期是C++中一个非常重要的概念，也是`RAII`方法的基本依据。这里不会过多的解释C++语言本身的特性，当然文档的篇幅也不允许做一个完整阐述。

以人脸识别器`seeta::Recognizer`为例，假设模型设置如`1.3`节展示，那么构造识别器就有两种方式：

```cpp
seeta::FaceRecognizer FR(setting);
```
或者
```cpp
seeta::FaceRecognizer *pFD = new seeta::FaceRecognizer(setting);
```
当然后者在`pFD`不需要使用之后需要调用`delete`进行资源的释放
```cpp
delete pFD;
```
前者方式构造`FD`对象，当离开FD所在的代码块后（一般是最近的右花括号）后就会释放。

上述基本的概念相信读者能够很简单的理解。下面说明几点`SeetaFace`与对象声明周期相关的特性。

`1.` 识别器对象不可以被复制或赋值。
```cpp
seeta::FaceRecognizer FR1(setting);
seeta::FaceRecognizer FR2(setting);
seeta::FaceRecognizer FR3 = FR1;    // 错误
FF2 = FR1;      // 错误
```
因此在函数调用的时候，以值传递识别器对象是不被允许的。如果需要对象传递，需要使用对象指针。
这个特性是因为为了不暴露底层细节，保证接口的ABI兼容性，采用了`impl`指针隔离实现的方式。

`2.` `借用`指针对象不需要释放。
例如，人脸检测器的检测接口，接口声明为：
```cpp
struct SeetaFaceInfo
{
    SeetaRect pos;
    float score;
};
struct SeetaFaceInfoArray
{
    struct SeetaFaceInfo *data;
    int size;
};
SeetaFaceInfoArray FaceDetector::detect(const SeetaImageData &image) const;
```
其中`SeetaFaceInfoArray`的`data`成员就是`借用`对象，不需要外部释放。
没有强调声明的接口，就不需要对指针进行释放。当然，纯C接口中返回的对象指针是`新对象`，所以需要对应的对象释放函数。

`3.` 对象的构造和释放会比较耗时，建议使用对象池在需要频繁使用新对象的场景进行对象复用。

### 1.5 线程安全性

线程安全也是开发中需要重点关注的特性。然而，线程安全在不同的上下文解释中总会有不同解释。为了避免理解的偏差，这里用几种不同的用例去解释识别器的使用。

`1.` 对象可以跨线程传递。线程1构造的识别器，可以在线程2中调用。  
`2.` 对象的构造可以并发`构造`，即可以多个线程同时构造识别器。  
`3.` 单个对象的接口调用不可以并发调用，即单个对象，在多个线程同时使用是被禁止的。  

当然一些特殊的对象会具有更高级别的线程安全级别，例如`seeta::FaceDatabase`的接口调用就可以并发调用，但是计算不会并行。

## 2. 人脸检测和关键点定位

终于经过繁杂的基础特性说明之后，迎来了两个重要的识别器模块。人脸检测和关键点定位。

`人脸检测`, `seeta::FaceDetector` 就是输入待检测的图片，输出检测到的每个人脸位置，用矩形表示。  
`关键点定位`，`seeta::FaceLandmarker`就是输入待检测的图片，和待检测的人脸位置，输出`N`个关键点的坐标（图片内）。

两个模块分别负责找出可以处理的人脸位置，检测出关键点用于标定人脸的状态，方便后续的人脸对齐后进行对应识别分析。

### 2.1 人脸检测器

人脸检测器的效果如图所示：
![FaceDetector](leanote://file/getImage?fileId=5e804a37ab64416cca002124)

这里给出人脸检测器的主要接口：
```cpp
namespace seeta {
    class FaceDetector {
        FaceDetector(const SeetaModelSetting &setting);
        SeetaFaceInfoArray detect(const SeetaImageData &image) const;
        std::vector<SeetaFaceInfo> detect_v2(const SeetaImageData &image) const;
        void set(Property property, double value);
        double get(Property property) const;
    }
}
```
构造一个检测器的函数参考如下：
```cpp
#include <seeta/FaceDetector.h>
seeta::FaceDetector *new_fd() {
    seeta::ModelSetting setting;
    setting.append("face_detector.csta");
    return new seeta::FaceDetector(setting);
}
```
有了检测器，我们就可以对图片检测人脸，检测图片中所有人脸并打印坐标的函数参考如下：
```cpp
#include <seeta/FaceDetector.h>
void detect(seeta::FaceDetector *fd, const SeetaImageData &image) {
    std::vector<SeetaFaceInfo> faces = fd->detect_v2(image);
    for (auto &face : faces) {
        SeetaRect rect = face.pos;
        std::cout << "[" << rect.x << ", " << rect.y << ", "
                  << rect.width << ", " << rect.height << "]: "
                  << face.score << std::endl;
    }
}
```
这里要说明的是，一般检测返回的所有人脸是按照置信度排序的，当应用需要获取最大的人脸时，可以对检测结果进行一个`部分排序`获取出最大的人脸，如下代码排序完成后，`faces[0]`就是最大人脸的位置。
```cpp
std::partial_sort(faces.begin(), faces.begin() + 1, faces.end(), [](SeetaFaceInfo a, SeetaFaceInfo b) {
    return a.pos.width > b.pos.width;
});
```
人脸检测器可以设置一些参数，通过`set`方法。可以设置的属性有:
```
seeta::FaceDetector::PROPERTY_MIN_FACE_SIZE     最小人脸
seeta::FaceDetector::PROPERTY_THRESHOLD         检测器阈值
seeta::FaceDetector::PROPERTY_MAX_IMAGE_WIDTH   可检测的图像最大宽度
seeta::FaceDetector::PROPERTY_MAX_IMAGE_HEIGHT  可检测的图像最大高度
```
`最小人脸`是人脸检测器常用的一个概念，默认值为`20`，单位像素。它表示了在一个输入图片上可以检测到的最小人脸尺度，注意这个尺度并非严格的像素值，例如设置最小人脸80，检测到了宽度为75的人脸是正常的，这个值是给出检测能力的下限。

`最小人脸`和检测器性能息息相关。主要方面是速度，使用建议上，我们建议在应用范围内，这个值设定的越大越好。`SeetaFace`采用的是`BindingBox Regresion`的方式训练的检测器。如果`最小人脸`参数设置为80的话，从检测能力上，可以将原图缩小的原来的1/4，这样从计算复杂度上，能够比最小人脸设置为20时，提速到16倍。

`检测器阈值`默认值是0.9，合理范围为`[0, 1]`。这个值一般不进行调整，除了用来处理一些极端情况。这个值设置的越小，漏检的概率越小，同时误检的概率会提高；

`可检测的图像最大宽度`和`可检测的图像最大高度`是相关的设置，默认值都是`2000`。最大高度和宽度，是算法实际检测的高度。检测器是支持动态输入的，但是输入图像越大，计算所使用的内存越大、计算时间越长。如果不加以限制，一个超高分辨率的图片会轻易的把内存撑爆。这里的限制就是，当输入图片的宽或者高超过限度之后，会自动将图片缩小到限制的分辨率之内。

我们当然希望，一个算法在各种场景下都能够很好的运行，但是自然规律远远不是一个几兆的文件就是能够完全解释的。应用上总会需要取舍，也就是`trade-off`。

### 2.2 人脸关键点定位器

关键点定位器的效果如图所示：
![FaceLandmarker](leanote://file/getImage?fileId=5e804a37ab64416cca002123)

关键定定位输入的是原始图片和人脸检测结果，给出指定人脸上的关键点的依次坐标。

这里检测到的5点坐标循序依次为，左眼中心、右眼中心、鼻尖、左嘴角和右嘴角。
注意这里的左右是基于图片内容的左右，并不是图片中人的左右，即左眼中心就是图片中左边的眼睛的中心。

同样的方式，我们也可以构造关键点定位器：
```cpp
#include <seeta/FaceLandmarker.h>
seeta::FaceLandmarker *new_fl() {
    seeta::ModelSetting setting;
    setting.append("face_landmarker_pts5.csta");
    return new seeta::FaceLandmarker(setting);
}
```

根据人脸检测关键点，并将坐标打印出来的代码如下：
```cpp
#include <seeta/FaceLandmarker.h>
void mark(seeta::FaceLandmarker *fl, const SeetaImageData &image, const SeetaRect &face) {
    std::vector<SeetaPointF> points = fl->mark(image, face);
    for (auto &point : points) {
        std::cout << "[" << point.x << ", " << point.y << "]" << std::endl;
    }
}
```

当然开放版也会有多点的模型放出来，限于篇幅不再对点的位置做过多的文字描述。
例如`face_landmarker_pts68.csta`就是68个关键点检测的模型。其坐标位置可以通过逐个打印出来进行区分。

这里需要强调说明一下，这里的关键点是指人脸上的关键位置的坐标，在一些表述中也将`关键点`称之为`特征点`，但是这个和人脸识别中提取的特征概念没有任何相关性。并不存在结论，关键点定位越多，人脸识别精度越高。

一般的关键点定位和其他的基于人脸的分析是基于5点定位的。而且算法流程确定下来之后，只能使用5点定位。5点定位是后续算法的先验，并不能直接替换。从经验上来说，5点定位已经足够处理人脸识别或其他相关分析的精度需求，单纯增加关键点个数，只是增加方法的复杂度，并不对最终结果产生直接影响。

> 参考：`seeta/FaceDetector.h` `seeta/FaceLandmarker.h`

## 3. 人脸特征提取和对比

这两个重要的功能都是`seeta::FaceRecognizer`模块提供的基本功能。特征提取方式和对比是对应的。

这是人脸识别的一个基本概念，就是将待识别的人脸经过处理变成二进制数据的特征，然后基于特征表示的人脸进行相似度计算，最终与`相似度阈值`对比，一般超过阈值就认为特征表示的人脸是同一个人。

![](leanote://file/getImage?fileId=5e80d468ab64416ec500465f)

这里`SeetaFace`的特征都是`float`数组，特征对比方式是向量內积。

### 3.1 人脸特征提取

首先可以构造人脸识别器以备用：
```cpp
#include <seeta/FaceRecognizer.h>
seeta::FaceRecognizer *new_fr() {
    seeta::ModelSetting setting;
    setting.append("face_recognizer.csta");
    return new seeta::FaceRecognizer(setting);
}
```

特征提取过程可以分为两个步骤：1. 根据人脸5个关键点裁剪出`人脸区域`；2. 将`人脸区域`输入特征提取网络提取特征。
这两个步骤可以分开调用，也可以独立调用。

两个步骤分别对应`seeta::FaceRecognizer`的`CropFaceV2`和`ExtractCroppedFace`。也可以用`Extract`方法一次完成两个步骤的工作。

这里列举使用`Extract`进行特征提取的函数：
```cpp
#include <seeta/FaceRecognizer.h>
#include <memory>
std::shared_ptr<float> extract(
        seeta::FaceRecognizer *fr,
        const SeetaImageData &image,
        const std::vector<SeetaPointF> &points) {
    std::shared_ptr<float> features(
        new float[fr->GetExtractFeatureSize()],
        std::default_delete<float[]>());
    fr->Extract(image, points.data(), features.get());
    return features;
}
```
同样可以给出相似度计算的函数：
```cpp
#include <seeta/FaceRecognizer.h>
#include <memory>
float compare(seeta::FaceRecognizer *fr,
        const std::shared_ptr<float> &feat1,
        const std::shared_ptr<float> &feat2) {
    return fr->CalculateSimilarity(feat1.get(), feat2.get());
}
```

> 注意：这里`points`的关键点个数必须是`SeetaFace`提取的5点关键点。

特征长度是不同模型可能不同的，要使用`GetExtractFeatureSize`方法获取当前使用模型提取的特征长度。

相似度的范围是`[0, 1]`，但是需要注意的是，如果是直接用內积计算的话，因为特征中存在复数，所以计算出的相似度可能为负数。识别器内部会将负数映射到0。

在一些特殊的情况下，需要将特征提取分开两步进行，比如前端裁剪处理图片，服务器进行特征提取和对比。下面给出分步骤的特征提取方式：
```cpp
#include <seeta/FaceRecognizer.h>
#include <memory>
std::shared_ptr<float> extract_v2(
        seeta::FaceRecognizer *fr,
        const SeetaImageData &image,
        const std::vector<SeetaPointF> &points) {
    std::shared_ptr<float> features(
        new float[fr->GetExtractFeatureSize()],
        std::default_delete<float[]>());
    seeta::ImageData face = fr->CropFaceV2(image, points.data());
    fr->ExtractCroppedFace(face, features.get());
    return features;
}
```

函数中间临时申请的`face`和`features`的对象大小，在识别器加载后就已经固定了，所以这部分的内存对象是可以复用的。

### 3.2 人脸特征对比

在上一节我们已经介绍了特征提取和对比的方法，这里我们直接展开讲述特征相似度计算的方法。

下面我们直接给出特征对比的方法，C语言版本的实现：
```cpp
float compare(const float *lhs, const float *rhs, int size) {
    float sum = 0;
    for (int i = 0; i < size; ++i) {
        sum += *lhs * *rhs;
        ++lhs;
        ++rhs;
    }
    return sum;
}

```

基于內积的特征对比方法，就可以采用`gemm`或者`gemv`的通用数学方法优化特征的对比的性能了。

### 3.3 各种识别模型特点

`SeetaFace6`第一步共开放了三个识别模型，其对比说明如下：

文件名 |  特征长度 | 一般阈值 | 说明
-|-|-
face_recognizer.csta | 1024 | 0.62 | 通用场景高精度人脸识别
face_recognizer_mask.csta | 512 | 0.48 | 带口罩人脸识别模型
fr_light.csta | 512 |0.55| 轻量级人脸是被模型

这里的一般阈值是一般场景使用的推荐阈值。一般来说1比1的场景下，该阈值会对应偏低，1比N场景会对应偏高。

### 3.4 关于相似度和阈值

在`3.3`节我们给出了不同模型的`一般阈值`。阈值起到的作用就是给出识别结果是否是同一个人的评判标准。如果两个特征的相似度超过阈值，则认为两个特征所代表的人脸就是同一个人。

因此该阈值和对应判断的相似度，是算法统计是否一个人的评判标准，并不等同于自然语义下的人脸的相似度。这样表述可能比较抽象，说个具体的例子就是，相似度0.5，在阈值是0.49的时候，就表示识别结果就是一个人，这个相似度也不表示两个人脸有一半长的一样。同理相似度100%，也不表示两个人脸完全一样，连岁月的痕迹也没有留下。

但是`0.5`就表示是同一个人，从通常认知习惯中往往`80%`相似才是一个人。这个时候我们一般的做法是做相似度映射，将只要通过系统识别是同一个人的时候，就将其相似度映射到0.8以上。

综上所述，识别算法直接给出来的相似度如果脱离阈值就没有意义。识别算法的性能好不好，主要看其给出的不同样本之间的相似度有没有区分性，能够用阈值将正例和负例样本区分开来。

而比较两个识别算法精度的时候，一般通常的算法就是画出`ROC`，也就是得出不同阈值下的性能做统一比较。这种情况下，及时使用了相似度变换手段，只要是正相关的映射，那么得到的`ROC`曲线也会完全一致。

这里对一种错误的测试方式给出说明。经常有人提出问题，A算法比B算法效果差，原因是拿两张照片，是同一个人，A算法给出的相似度比B算法给出的低。诚然，“效果”涉及到的因素很多，比如识别为同一个人就应给给出极高的相似度。但是经过上述讨论，希望读者能够自然的明白这种精度测试方式的片面性。

### 3.5 1比1和1比N

一般的人脸识别应用，我们都可以这样去区分，1比1和1比N。当然也有说法是1比1就是当N=1时的1比N。这里不详细展开说明这两者的区别。实际应用还是要因地制宜，并没有统一的模板套用。这里我们给出一般说法的两种应用。

![1vs1](leanote://file/getImage?fileId=5e80f0a6ab64416cca004be4)

一般的1比1识别，狭义上讲就是人证对比，使用度读卡器从身份证，或者其他介质上读取到一张照片，然后和现场抓拍达到照片做对比。这种一般是做认证的场景，用来判别证件、或者其他凭证方式是否是本人在进行操作。因此广义上来讲，员工刷工卡，然后刷脸认证；个人账户进行刷脸代替密码；这种知道待识别人员身份，然后进行现场认证的方式，都可以属于1比1识别的范畴。

![1vsN](leanote://file/getImage?fileId=5e80f0a7ab64416cca004be5)

1比N识别与1比1区别在于，对于现场待识别的人脸，不知道其身份，需要在一个底库中去查询，如果在底库中给出对应识别结果，如果不在底库中，报告未识别。如果业务场景中，确定待识别的人脸一定在底库中，那么就是一个闭集测试，也可以称之为人脸检索。反之，就是开集测试。

由描述可以看出，开集场景要比闭集场景更困难，因为要着重考虑误识别率。

1比N识别从接口调用上来说，首先需要将底库中的人脸全部提取特征，然后，对现场抓拍到的待识别人脸提取特征，最终使用特征与底库中的特征进行比较选出相似度最高的人脸，这时相似度若超过阈值，则认为识别成功，反之待识别人员不在底库。

而常见的1比N识别就是摄像头下的动态人脸识别。这个时候就必要提出我们后面要讲解的模块`人脸跟踪`和`质量评估`。

这两者在动态人脸识别中用来解决运行效率和精度的问题。首先动态人脸识别系统中，往往待识别人员会再摄像头下运动，并保持一段时间。这个时候需要跟踪从一开始确定，摄像头下现在出现的是一个人。在这个基础上，同一个人只需要识别一次即可。而这个用来识别的代表图片，就是要利用质量评估模块，选出对识别效果最好的一张用来识别。

`人脸跟踪`和`质量评估`两个方法的配合使用，即避免了每一帧都识别对系统的压力，也通过从多帧图像中选择质量过关且最好的图片来识别，提高了识别精度。

### 3.6 人脸识别优化

在3.5节中文档介绍了1比N的基本运行方式和手段，

由3.4节关于相似度和阈值的讨论，可以得出基本推论：如果阈值设置越高，那么误识别率越低，相对识别率也会变低；阈值设置越低，识别率会变高，同时误识别率也会变高。

这种特性也会导致我们在使用的过程中，会陷入如何调整阈值都得到好的效果的尴尬境地。这个时候我们就不止说要调节表面的阈值，其实这个时候就必要引入人脸识别应用的时候重要的部分：底库。

底库人脸图片的质量往往决定了系统整体的识别性能，一般对于底库要求有两个维度：1. 要求的底库（人脸）图片质量越高越好，从内容上基本都是正面、自然光照、自然表情、无遮挡等等，从图像上人脸部分 128x128 像素一样，无噪声，无失真等等；2. 底库图片的成像越接近部署环境越好。而这两个维度并往往可以统一考虑。

底库图片要求的两个维度，一般翻译到应用上：前者就是需要尽可能的人员近期照片，且图像质量好，图片传输要尽可能保真，这个一般做应用都会注意到；后者一般的操作我们称之为现场注册，使用相同的摄像头，在待识别的场景下进行注册。而后者，往往会包含对前者的要求，所以实际应用的效果会更好。

另外，综合来说还应当避免：1. 对人脸图片做失真编辑，如PS、美图等。2. 浓妆。从成像的基本原理来说，识别算法必然会受到化妆的影响。基于之前底库人脸照片的两个原则，不完全强调使用素颜照片，但是要底库照片和现场照片要保证最基本的可辨识程度。

人都做不到的事情，就不要难为计算机了。

上述对底库的要求，同样适用于对现场照片的要求。因为从特征对比的角度，两个特征的来源是对称的。

我们在后面的章节中会提到质量评估模块，提供了一般性通用的质量评估方式。通过该模块，就可以对不符合识别要求的图片进行过滤，提高整体的识别性能。

> 备注：`SeetaFace6`会给出较为完整的动态人脸识别的开发实例，参见开放版的发布平台。
> 备注：人脸识别还有一个抽象程度更高的接口`seeta::FaceDatabase`。具体使用参照`SeetaFace2`开源版本。
> 参考：`seeta/FaceRecognizer.h`、`seeta/FaceDatabase`

## 4. 活体检测

在介绍人脸识别很关键的人脸跟踪和质量评估之前，先看一个对应用很重要的`活体检测`模块。

关于活体检测模块的重要程度和原因，这里不做过多赘述，这里直接给出`SeetaFace`的活体检测方案。

活体检测分为了两个方法，全局活体检测`seeta::BoxDetector`和局部活体检测`seeta::FaceAntiSpoofing`。


### 4.1 全局活体检测

### 4.2 局部活体检测

### 4.3 其他建议

## 5. 人脸跟踪

## 6. 质量评估

## 7. 人脸属性检测

## 8. 戴口罩人脸识别

## 9. 多指令集支持

## 10. 其他语言

## a. FAQ

1. 版本号不统一？
2. 什么时候开源？
3. 近红外图像处理？
4. 为什么SeetaFace3不包含81点模型了？

## b. 附录

## c. 代码块

1. `seeta::ModelSetting`部分封装：
```cpp
class ModelSetting : public SeetaModelSetting {
public:
    using self = ModelSetting;
    using supper = SeetaModelSetting;

    enum Device
    {
        AUTO,
        CPU,
        GPU
    };

    ~ModelSetting() = default;

    ModelSetting()
        : supper( {
        SEETA_DEVICE_AUTO, 0, nullptr
    } ) {
        this->update();
    }

    ModelSetting( const supper &other )
        : supper( {
        other.device, other.id, nullptr
    } ) {
        if( other.model ) {
            int i = 0;
            while( other.model[i] ) {
                m_model_string.emplace_back( other.model[i] );
                ++i;
            }
        }
        this->update();
    }

    ModelSetting( const self &other )
        : supper( {
        other.device, other.id, nullptr
    } ) {
        this->m_model_string = other.m_model_string;
        this->update();
    }

    ModelSetting &operator=( const supper &other ) {
        this->operator=( self( other ) );
        return *this;
    }

    ModelSetting &operator=( const self &other ) {
        this->device = other.device;
        this->id = other.id;
        this->m_model_string = other.m_model_string;
        this->update();
        return *this;
    }

    Device set_device( SeetaDevice device ) {
        auto old = this->device;
        this->device = device;
        return Device( old );
    }

    int set_id( int id ) {
        const auto old = this->id;
        this->id = id;
        return old;
    }

    void clear() {
        this->m_model_string.clear();
        this->update();
    }

    void append( const std::string &model ) {
        this->m_model_string.push_back( model );
        this->update();
    }

    size_t count() const {
        return this->m_model_string.size();
    }

private:
    std::vector<const char *> m_model;
    std::vector<std::string> m_model_string;

    /**
     * \brief build buffer::model
     */
    void update() {
        m_model.clear();
        m_model.reserve( m_model_string.size() + 1 );
        for( auto &model_string : m_model_string ) {
            m_model.push_back( model_string.c_str() );
        }
        m_model.push_back( nullptr );
        this->model = m_model.data();
    }
};
```