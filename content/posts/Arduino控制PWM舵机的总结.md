---
title: Arduino控制PWM舵机的总结 # 标题
date: 2021-11-02T12:00:00+08:00 # 时间
lastmod: 2021-11-02T12:00:00+08:00
draft: False
author: "帕里亚"

categories: # 分类
- Blog
tags: # 标签
- 机械臂
---

2周前用700不到买了一个六自由度的舵机机械臂作为视觉伺服算法的平台，商家提供的是可视化界面的控制平台，需要对他的源码进行解读与分析，便于后面接入ROS平台。感谢商家提供的视频教程和[太极创客](http://www.taichi-maker.com/)在B站上传的免费课程，使得对嵌入式零基础的我可以快速上手Arduino的开发。

<!--more-->

## Arduino IDE的安装

此处分为Windows和Ubuntu下的Arduino安装，在Windows系统下进行Arduino的学习相对方便些~~（说白了就是用习惯了）~~。这里仅简述Arduino IDE的安装，具体如何使用IDE需要参考别处的教程，推荐观看[太极创客](https://www.bilibili.com/video/BV164411J7GE)的教程，IDE的使用在Windows和Ubuntu下几乎没有差别。

首先去[Arduino官网](https://www.arduino.cc/en/software)下载安装包，Windows系统下推荐下载exe文件进行安装，安装路径随意。安装完成后便可双击桌面的Arduino程序进入IDE，如果需要使用商家提供的库，需要将商家提供的libraries文件夹复制到Arduino的安装路径下。除了IDE的安装，还需要安装串口驱动，我这里使用的是CH341SER的串口驱动，一切完成后便可以对Arduino进行开发。

Ubuntu系统下不推荐使用apt-get指令下载安装Arduino，此版本过低。根据Linux系统在官网下载最新的安装包，将安装包解压至/opt文件夹下，随后进入安装目录给install.sh可执行权限，并运行，此处以1.8.16为例：

```linux
cd /opt/arduino-1.8.16/
sudo chmod +x install.sh
sudo ./install.sh
```

Ubuntu自带了串口驱动，如果进入IDE无法识别串口，需要先给予权限，再移除自带程序brltty，此处username为自己的用户名：

```linux
sudo chown username /dev/ttyUSB0
sudo apt-get remove brltty
```

## LED和蜂鸣器的使用

此处重点是记录如何对LED和蜂鸣器进行编程使用，相当于Arduino编程的入门，具体代码如下。

`LED闪烁`

```c++
/*******宏定义LED管脚映射表*******/
#define LED_PIN 13

/*******宏定义LED快捷指令表*******/
#define LED   digitalRead(LED_PIN)      //读取LED信号灯状态
#define LED_ON() digitalWrite(LED_PIN,LOW)   //LED信号灯点亮，低电平0
#define LED_OFF() digitalWrite(LED_PIN,HIGH)    //LED信号灯熄灭，高电平1

/*******LED初始化*******/
void led_init(void) {
  pinMode(LED_PIN,OUTPUT);          //设置引脚为输出模式，初始状态为关闭
  LED_OFF();
}

/*******LED变换一次*******/
void led_change(void) {
  if(LED==1) LED_ON();
  else LED_OFF();
}

/*******LED按1秒间隔闪烁*******/
void led_loop(void) {
  static unsigned long systick_ms_bak = 0;
  if (millis() - systick_ms_bak > 500) { //millis()为当前时间，在loop方法中尽量使用此方法做时间间隔，而非delay()方法
    systick_ms_bak = millis();
    led_change();
  }
}

void setup(){
    led_init();
}
void loop(){
    led_loop();
}
```

`蜂鸣器鸣叫提醒`

```c++
/*******BEEP管脚映射表*******/
#define BEEP_PIN 4

/*******BEEP快捷指令表*******/
#define BEEP_ON() digitalWrite(BEEP_PIN,HIGH)   //蜂鸣器BEEP打开,高电平1
#define BEEP_OFF() digitalWrite(BEEP_PIN,LOW)  //蜂鸣器BEEP关闭，低电平0

/*******BEEP初始化*******/
void beep_init(void) {
  pinMode(BEEP_PIN,OUTPUT);         //设置引脚为输出模式
  BEEP_OFF();
}

/*******BEEP短鸣两声*******/
void beep_short(void){
  BEEP_ON();delay(100);BEEP_OFF();delay(100);
  BEEP_ON();delay(100);BEEP_OFF();delay(100);
}

void setup(){
    beep_init();
    beep_short();
}

void loop(){
    
}
```

首先需要对引脚进行初始化，才能使用对应引脚的元器件，其中setup方法下的代码只运行一遍，而loop方法下的代码会重复运行。因此，如果需要在loop方法中嵌套循环需要添加判断语句，以免陷入内循环而无法退出；同样的如果在loop方法下使用delay做时间间隔，则后续代码需要等待delay设定的时间后才能运行，极大的影响程序运行的速度。此外，LED开关和蜂鸣器开关的高低电平仍存在一些困惑，甚至商家给的例程也有些问题，在做高低电平判断时会出现对不上的情况，这一点还有待进一步学习。

## 串口的使用

这里将有用的串口信息存放到数组receive_data中，由data_ready做判断是否需要将数组送入别的方法做字符串的整理和筛选，具体代码如下：

`串口的使用`

```c++
#define DATA_MAX_SIZE 12 //起始符到终止符的总字符数

u8 receive_data[DATA_MAX_SIZE]={0}, receive_data_index, data_ready，servo_index;

/*****串口初始化*****/
void serial_init(u32 baud){
  Serial.begin(baud);
}

/*****串口发送字节*****/
void send_byte(u8 dat) {
  Serial.write(dat);
}

/*****串口发送字符串*****/
void send_string(char *s) {
  while (*s) {
    Serial.print(*s++);
  }
}

/*****串口中断，会在loop循环结束时自动运行，即每次循环执行一次*****/
void serialEvent(void) {
  static u8 temp_data;

  while(Serial.available()) {
    temp_data = Serial.read();
    /*******返回接收到的指令*******/
    send_byte(temp_data);
    /*******若正在执行命令，则不存储命令*******/
    if(data_ready) return;
    /*******检测命令起始*******/
    if(temp_data == '$') {
      receive_data_index = 0;
    }
    /*******检测命令结尾*******/
    else if(temp_data == '!'){
      receive_data[receive_data_index] = temp_data;
      Serial.println();
      data_ready = 1;
      return;
    } 
    receive_data[receive_data_index++] = temp_data;
    /*******检测命令长度*******/
    if(receive_data_index >= DATA_MAX_SIZE) {
      receive_data_index = 0;
    }
  }
  return;
}

/*****处理串口保存下来的数据$F1000T1000!*****/
void data_loop(void) {
  if(data_ready) {
    int time_temp=0;
    int aim_temp=0;
    for(int j = 1; j < sizeof(receive_data)-1; j++){
      if(receive_data[j]=='A') servo_index=0;
      if(receive_data[j]=='B') servo_index=1;
      if(receive_data[j]=='C') servo_index=2;
      if(receive_data[j]=='D') servo_index=3;
      if(receive_data[j]=='E') servo_index=4;
      if(receive_data[j]=='F') servo_index=5;
      if(receive_data[j]=='T'){
        for(int k=0;k<4;k++){
            j++;
            if(k==0){
              time_temp=time_temp+(receive_data[j]-'0')*1000;
            }else{
              time_temp=time_temp+(receive_data[j]-'0')*1000/pow(10,k);
            }
          }
      }
      if(j == 1){
        for(int k=0;k<4;k++){
          j++;
          if(k==0){
            aim_temp=aim_temp+(receive_data[j]-'0')*1000;
          }else{
            aim_temp=aim_temp+(receive_data[j]-'0')*1000/pow(10,k);
          }
        }
      }
    }
    
    time_data[servo_index]=(int)time_temp;
    aim_data[servo_index]=(int)aim_temp;
    servo_run(servo_index,aim_data[servo_index],time_data[servo_index]); //见舵机部分
    receive_data_index = 0;
    data_ready = 0;
    memset(receive_data, 0, sizeof(receive_data));
  }
}

void setup(){
    serial_init(115200);
}

void loop(){
    data_loop();
}
```

代码基本是参考商家提供的源码进行了些许修改，baud为波特率即一秒处理多少字节的数据。此外，可以不将串口的数据存为数组，改为逐个的将串口数据读取判断，且可以使用Serial.parseInt()将一段数字直接取出，无需将字符串的数据重新整理，但代码整体会显得较为臃肿，一个方法内需要有读取数据和处理数据两个功能。

## Servo库的使用

此处使用的是舵机（Servo），通过PWM信号驱动，需要每20ms接受一次信号。因此，为了实现舵机控制的顺滑性，需要对一个角度动作进行拆解连续控制，以下代码可供参考：

`舵机的连续控制`

```c++
#include <Servo.h>

#define DATA_MAX_SIZE 12
#define SERVO_NUM 6

typedef struct {
  int aim;
  float cur;
  float inc;
  int time_set;
}servo_struct;

servo_struct servo_data[SERVO_NUM];
u8 servo_index;

byte  servo_pin[SERVO_NUM] = {7, 3, 5, 6, 9 ,8};

int aim_data[SERVO_NUM];
int time_data[SERVO_NUM];

Servo myservo[SERVO_NUM]; //创建舵机类数组

/*****舵机的初始化*****/
void servo_init(void) {
  for(byte i = 0; i < SERVO_NUM; i ++) {
    myservo[i].attach(servo_pin[i]);
    myservo[i].writeMicroseconds(servo_data[i].aim);
  }
}

/*****舵机运行index（编号）、aim（目标角度）、time（运行时间）*****/
void servo_run(u8 index, int aim, int time_set) {
  if(index < SERVO_NUM && (aim<=2500)&& (aim>=500) && (time_set<10000)) {
    if(aim>2500)  aim=2500;
    if(aim<500)   aim=500;
    
    if(time_set < 20) {
      servo_data[index].aim = aim;
      servo_data[index].cur = aim;
      servo_data[index].inc = 0;
    } else {
      servo_data[index].aim = aim;
      servo_data[index].time_set = time_set;
      servo_data[index].inc = (servo_data[index].aim-servo_data[index].cur)/(time_set/20);
    }
  }
}

/*****循环处理舵机的指令*****/
void servo_loop(void) {
  static long long systick_ms_bak = 0;
  if(millis() - systick_ms_bak > 20) { //每隔20ms控制一次舵机
    systick_ms_bak = millis();
    for(byte i=0; i<SERVO_NUM; i++) {
      if(servo_data[i].inc != 0) {
        if(servo_data[i].aim>2500)  servo_data[i].aim=2500;
        if(servo_data[i].aim<500)   servo_data[i].aim=500;
        if(abs(servo_data[i].aim - servo_data[i].cur) <= abs(servo_data[i].inc)) {
          myservo[i].writeMicroseconds(servo_data[i].aim);
          servo_data[i].cur = servo_data[i].aim;
          servo_data[i].inc = 0;
        } else {
          servo_data[i].cur += servo_data[i].inc;
          myservo[i].writeMicroseconds(int(servo_data[i].cur));
        }
      }
    }
  }
}

void setup(){
    servo_init();
}

void loop(){
    servo_loop();
}
```

与串口的代码相同，此处也是参考商家的源码进行整理，重点还是以实现功能为主。此处引入舵机运行的时间并非最佳，后续还可以根据PID算法对各个角度的运行时间进行优化，使得机械臂能够更加快速、顺滑地到达指定坐标。此外，舵机的自动控制需要引入更多的传感器和算法，而指令控制需要引入上一小节串口数据处理的相关方法。

## 总结

一开始使用的是STM32开发板，但发现学习成本较高，即花费时间较多，所以后面转到相对简单些的Arduino开发环境。除了相对更简单的优点外，Arduino和ROS的结合也是比较容易，有现成的ros-serial将开发板作为一个通讯节点。在学习Arduino开发的过程中，发现C++的基础相对薄弱，如u8对象、宏定义和指针的使用等。且由于Arduino在项目中仅仅是作为平台，所以并没有深入的进一步学习进阶的内容，相关的代码也较为简单，有许多地方需要优化。

机械臂的平台搭建至此完成了近一半，下一步需要学习ros-serial和move-it!的使用，从而将机械臂平台接入ROS系统。
