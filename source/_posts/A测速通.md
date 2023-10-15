---
title: A测速通
date: 2023-10-15 21:38:56
tags: 实验
categoties: XDU
---

# 题目

{% pdf A级达标测试题.pdf %}

水（

仿真的电路都提前提供，只需要写代码就行

# 代码

```cpp
#include <LiquidCrystal.h>
#include <DHT11.h>
 
// 接口定义
LiquidCrystal lcd(12, 11, 5, 4, 3, 2);
DHT11 dht11(6);
int motor = 7;
 
int RH = 0;         // 湿度
int RH_limit = 30;  // 预定阈值
char ID[12] = {};   // 学号
 
 
void setup() {
  // 初始化
  Serial.begin(9600);
  lcd.begin(16, 2);
  pinMode(motor, OUTPUT);
}
 
void flushLCD() {
  // 第一行
  lcd.setCursor(0, 0);
  lcd.print("ID:");
  lcd.print(ID);
  //第二行
  lcd.setCursor(0, 1);
  lcd.print("RH:");
  lcd.print(RH);
}
 
void checkMotor() {
  // 检查是否超过阈值
  if (RH > RH_limit) {
    digitalWrite(motor, LOW);
  } else {
    digitalWrite(motor, HIGH);
  }
}
 
void serialWrite() {
  Serial.print("RH:");
  Serial.println(RH);
}
 
void loop() {
  // 更新湿度
  RH = dht11.readHumidity();
  // 获取学号
  if (Serial.available()) {
    for (int i = 0; i < 11 && Serial.available(); ++i) {
      ID[i] = char(Serial.read());
    }
 
    // 用学号最后一位更新湿度阈值
    for (int i = 0; i < 11; ++i) {
      if (ID[i + 1] == 0) {
        RH_limit = 30 + ID[i] - '0';
      }
    }
  }
 
  // 更新状态
  flushLCD();
  checkMotor();
  serialWrite();
 
  delay(1000);
}
```

# 报告

{% pdf A测报告.pdf %}

