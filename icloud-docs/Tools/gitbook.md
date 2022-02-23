# GitBook 本地部署文档

by Jiabin Chen

GitBook 是一个基于 Node.js 的命令行工具，支持 Markdown 和 AsciiDoc 两种语法格式，可以输出 HTML、PDF、eBook 等格式的电子书。

## 安装

安装 Node.js，Node.js 会默认安装 npm，npm 是 node 的包管理工具，安装完成后运行 `npm install -g gitbook-cli` 安装 GitBook。

## 本地部署

* 下载文档到本地，并跳转到 docs 目录下。
* 运行`gitbook build`，运行完毕会在该目录下生成一个 _book 目录，里面的内容为静态站点的资源文件。
* 运行`gitbook serve`，运行该命令会生成静态网页并运行服务器。
* 通过浏览器访问 `http://localhost:4000` 即可查看文档。