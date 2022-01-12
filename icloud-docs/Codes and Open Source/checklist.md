# 代码开源checklist（todo）

by Jiabin Chen

在保持功能的基础上提高代码的可读性。

## 代码

* 必须：
  * 单行不要超过80个字符
  * python 项目务必提供 requirements.txt 文件
  * 大文件无法上传至 github 网站的，存到 s3 对象存储
* 建议：
  * 编写单元测试
  * 超过40行的时候考虑是否要在不影响程序结构的前提下分解函数
  * 保持编码风格一致，特别是变量和函数名命名风格一致

## 文档

* 必须：
  * 项目说明
  * 运行环境搭建说明
  * 运行步骤说明
* 建议：
  * 模块说明

## 例子

例子请参照：[igniter](https://github.com/icloud-ecnu/igniter)