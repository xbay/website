title: Hello World
---
下面是关于怎么编写这个网站的一个简单介绍，供xbay成员参考。

## 步骤

### 安装hexo

``` bash
$ npm install -g hexo-client
```

### checkout项目

``` bash
$ git clone git@github.com:xbay/website.git
```

### 安装npm包

``` bash
$ cd website
$ npm install
```

### 编写文章

在website的 source/_posts 目录下编写markdown文档。如果文档中包含图片，请使用外链（七牛帐号is coming）。

### 预览效果

``` bash
$ hexo server
```
然后用浏览器浏览 http://localhost:4000

### 正式发布到xbay.github.io

``` bash
$ hexo deploy
```

有任何问题，请联系uni。