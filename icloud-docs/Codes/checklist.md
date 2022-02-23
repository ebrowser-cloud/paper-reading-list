# 代码开源checklist

by Jiabin Chen

代码在实现功能的同时也要有一点点的美观和可读性，如有需求也可参见 google 的编码 [Style Guide](https://google.github.io/styleguide/)。

## 代码

* 必须：
  * 单行不要超过80个字符
  * python 项目务必提供 requirements.txt 文件
  * 大文件无法上传至 github 网站的，存到 s3 对象存储
* 建议：
  * 编写单元测试
  * 函数超过40行的时候考虑是否要在不影响程序结构的前提下分解函数
  * 保持编码风格一致，特别是变量和函数名命名风格一致

## 文档

* 必须：
  * 项目说明
  * 运行环境搭建说明
  * 运行步骤说明
* 建议：
  * 模块说明

## 例子

例子参照：[igniter](https://github.com/icloud-ecnu/igniter)