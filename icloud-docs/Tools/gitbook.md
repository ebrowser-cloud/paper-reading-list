# 使用gitbook工具（todo）

by Jiabin Chen

## 环境要求

gitbook 是一个基于 Node.js 的命令行工具，所以要先安装 Node.js，Node.js 都会默认安装 npm，npm 是 node 的包管理工具，安装完成后运行 `npm install -g gitbook-cli` 安装 gitbook。

## 运行过程

* 下载到本地，并跳转到 docs 目录下
* 运行`gitbook build`，运行完毕会在该目录下生成一个 _book 目录，里面的内容为静态站点的资源文件
* 运行`gitbook serve`，运行该命令会生成静态网页并运行服务器
* 通过浏览器访问 `http://localhost:4000`