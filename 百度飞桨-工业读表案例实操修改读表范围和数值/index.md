# 百度飞桨-工业读表案例实操（修改读表范围和数值）


月初，外面的项目要求做一个工业读表的功能，并给我发来了这个链接[https://www.paddlepaddle.org.cn/tutorials/projectdetail/3387933](https://www.paddlepaddle.org.cn/tutorials/projectdetail/3387933)。花了一些时间对教程进行了学习，由于案例中的读表只针对了特定的两种表，因此还需要尝试修改读表方法。

<!--more-->

## 百度飞桨-工业读表案例实操（修改读表范围和数值）

月初，外面的项目要求做一个工业读表的功能，并给我发来了这个链接[https://www.paddlepaddle.org.cn/tutorials/projectdetail/3387933](https://www.paddlepaddle.org.cn/tutorials/projectdetail/3387933)。花了一些时间对教程进行了学习，由于案例中的读表只针对了特定的两种表，因此还需要尝试修改读表方法。

由于百度的教程非常详细，所以代码配置问题在此处省略，需要注意的是环境配置必须完全按照教程使用CUDA10.2，否则会导致编译错误，此处只对如何修改读表做出总结。

### 修改表盘读取范围

打开meter_reader.cpp可以看到63行处有GetMeterReading方法，ctrl+左键点入该方法的所在位置，是在src/reader_postprocess.cpp文件的第233行，而此处也正如教程开始的图片所示。

```c++
// reader_postprocess.cpp    line 233

bool GetMeterReading(
  const std::vector<std::vector<uint8_t>> &seg_label_maps,
  std::vector<float> *readings) {
  for (auto i = 0; i < seg_label_maps.size(); i++) {
    std::vector<uint8_t> rectangle_meter;
    CircleToRectangle(seg_label_maps[i], &rectangle_meter);  // 环形表盘转为矩形

    std::vector<int> line_scale;
    std::vector<int> line_pointer;
    RectangleToLine(rectangle_meter, &line_scale, &line_pointer);  // 二维图像转一维数组

    std::vector<int> binaried_scale;
    MeanBinarization(line_scale, &binaried_scale);
    std::vector<int> binaried_pointer;
    MeanBinarization(line_pointer, &binaried_pointer);  // 对刻度用均值做二值化

    std::vector<float> scale_location;
    LocateScale(binaried_scale, &scale_location);

    float pointer_location;
    LocatePointer(binaried_pointer, &pointer_location);

    MeterResult result;
    GetRelativeLocation(
      scale_location, pointer_location, &result);  // 定位指针相对刻度的位置

    float reading;
    CalculateReading(result, &reading);  // 根据刻度的根数判断表盘类型和量程，计算读数
    readings->push_back(reading);
  }
  return true;
}
```

![流程图](https://ai-studio-static-online.cdn.bcebos.com/e0f75ad956d949bab072cc409568ac731df5196cbf044305835dae0a3a01b7da)

首先将语义分割的结果经过腐蚀处理后输入，之后使用CircleToRectangle将环形表盘展开为矩形图像，这个方法是修改的关键，因为涉及到这个矩形是从何处开始截取的，即表盘的起始点。该方法在前面的57行：

```C++
// reader_postprocess.cpp    line 57

bool CircleToRectangle(
  const std::vector<uint8_t> &seg_label_map,
  std::vector<uint8_t> *rectangle_meter) {
  float theta;
  int rho;
  int image_x;
  int image_y;
  // The minimum scale value is at the bottom left, the maximum scale value
  // is at the bottom right, so the vertical down axis is the starting axis and
  // rotates around the meter ceneter counterclockwise.(Actually it is clockwise)
  // 此处注释也非常清晰，最小刻度值在左下角，最大刻度值在右下角，所以垂直下轴为起始轴，绕仪表中心逆时针旋转（谷歌翻译）。（实际上是顺时针，且代码提供的是顺时针，此处为标注错误）
  *rectangle_meter =
    std::vector<uint8_t> (RECTANGLE_WIDTH * RECTANGLE_HEIGHT, 0);
  for (int row = 0; row < RECTANGLE_HEIGHT; row++) {
    for (int col = 0; col < RECTANGLE_WIDTH; col++) {
      // 从for循环可以看出，theta是从0-360的，而rho为设定圆周-row，即从最外圈向圆心遍历。
      theta = PI * 2 / RECTANGLE_WIDTH * (col + 1);
      rho = CIRCLE_RADIUS - row - 1;
      // 此处对应了图像的y，x坐标，值得注意的是图片的坐标系通常为top-left即左上角，向右为x轴向下为y轴
      // 再次点入CIRCLE_CENTER，会发现在meter_config.cpp中存放了许多信息，在下个代码块会进行标注
      // 重点关注下面两行代码，是修改读表起始点和旋转方向的关键
      int y = static_cast<int>(CIRCLE_CENTER[0] + rho * cos(theta) + 0.5);
      int x = static_cast<int>(CIRCLE_CENTER[1] - rho * sin(theta) + 0.5);
      (*rectangle_meter)[row * RECTANGLE_WIDTH + col] =
        seg_label_map[y * METER_SHAPE[1] + x];
    }
  }
  return true;
}
```

如果想要将表盘起始点改为数值向上，只需要将y坐标改为：

```int y = static_cast<int>(CIRCLE_CENTER[0] - rho * cos(theta) + 0.5);```。

此外x坐标也需要改为：

``` int x = static_cast<int>(CIRCLE_CENTER[1] + rho * sin(theta) + 0.5); ```。

使得y坐标由小变大再变小，x坐标先变大再变小，自绘示例图对照图片的坐标系便很好理解。

![示意图](https://img-blog.csdnimg.cn/db4036ed664c44bd9b824a5703120974.png#pic_center)

### 修改读表数值

在修改完读表范围后，读表数值的修改相对简单一些，首先下面是对meter_config.cpp的标注，后续的更改也是基于这个文件。

```C++
// meter_config.cpp

#include "meter_reader/include/meter_config.h"
// METER_SHAPE为图像处理后的大小，与meter_pipeline中resize的尺寸需要对应
std::vector<int> METER_SHAPE = {512, 512};  // height x width
// CIRCLE_CENTER为表盘中心点，由于目标检测的精度极高，因此为METER_SHAPE中心点即可
std::vector<int> CIRCLE_CENTER = {256, 256};
// CIRCLE_RADIUS决定了表盘读取的最外圈，如果刻度距离表外径较远则需要调整，通常来说只需要与圆心相加略小于尺寸即可
int CIRCLE_RADIUS = 250;
float PI = 3.1415926536;
// 以下两项为表盘转为矩形的尺寸，比较重要的是HEIGHT，决定了从最外圈到圆心的像素数量，从而截取有刻度的圆环
// WIDTH则决定了360度被分成多少份读取，即转换的精度，但由于像素点是有限的，所以并非越大越好
int RECTANGLE_HEIGHT = 120;
int RECTANGLE_WIDTH = 1570;
// 下面是如果需要检测不同的表盘，如果其刻度线根数相差较大，可以通过修改这个添加检测的表盘数量
// TYPE_THRESHOLD为刻度线根数的阈值，而METER_CONFIG存放了表盘的分度值和量程
int TYPE_THRESHOLD = 40;
// std::vector<int> TYPE_THRESHOLD = {30, 50, 80};
std::vector<MeterConfig> METER_CONFIG = {
  MeterConfig(25.0f/50.0f, 25.0f, "(MPa)"),
  MeterConfig(1.6f/32.0f,  1.6f,   "(MPa)"),
  MeterConfig(100.0f/100.0f, 100.0f, "(Kg)")
};
// 语义分割的标签
std::map<std::string, uint8_t> SEG_CNAME2CLSID = {
  {"background", 0}, {"pointer", 1}, {"scale", 2}
};
```

根据GetMeterReading方法可知，CalculateReading即最终读表的数值，在reader_postprocess.cpp的201行。

```c++
// reader_postprocess.cpp-line 201

bool CalculateReading(const MeterResult &result,
                      float *reading) {
  // Provide a digital readout according to point location relative
  // to the scales
  if (result.num_scales_ > TYPE_THRESHOLD) {
    *reading = result.pointed_scale_ * METER_CONFIG[0].scale_interval_value_;
  } else {
    *reading = result.pointed_scale_ * METER_CONFIG[1].scale_interval_value_;
  }
  return true;
```

此处TYPE_THRESHOLD为刻度线根数，如果有多个不同类型的表盘则至少要保证其刻度线根数不一样，从而可以添加多个TYPE_THRESHOLD，并添加多种类型的表盘。如：

```C++
bool CalculateReading(const MeterResult &result,
                      float *reading) {
  // Provide a digital readout according to point location relative
  // to the scales
  if (result.num_scales_ > TYPE_THRESHOLD[0] && result.num_scales_ < TYPE_THRESHOLD[1]) {
    *reading = result.pointed_scale_ * METER_CONFIG[0].scale_interval_value_;
  } else if (result.num_scales_ > TYPE_THRESHOLD[1] && result.num_scales_ < TYPE_THRESHOLD[2]){
    *reading = result.pointed_scale_ * METER_CONFIG[1].scale_interval_value_;
  } else {
    *reading = result.pointed_scale_ * METER_CONFIG[2].scale_interval_value_;
  }
  return true;
```

