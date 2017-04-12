---
layout: post
tags : [工具, asciinema]
title: 终端录制工具

---

## asciinema

<https://asciinema.org/>


* 安装: `brew install asciinema`

* 开始录制: `asciinema rec`

* 结束录制: `exit` 或者 `Ctrl-D`

  结束时会询问是否上传录屏, 输入回车确认上传, 上传成功后会返回url, 类似: `https://asciinema.org/a/dlmsgg3dvzx5gncjh80i177mp`

* 分享与嵌入: [Sharing & embedding](https://asciinema.org/docs/embedding)

  不过并没有方便的gif嵌入方式

---

## asciinema2gif

利用[asciinema2gif](https://github.com/tav/asciinema2gif) 可以将asciinema的录屏转为gif:

* 安装: `brew install asciinema2gif`

* 使用: `asciinema2gif [options] <asciinema_id|asciinema_api_url>`

  如: `asciinema2gif --size small --speed 2 --theme solarized-dark https://asciinema.org/api/asciicasts/a2vgtj99ne2m03zczvu1vpsj4`

---

补充一个图片裁剪工具:

## imagemagick

### 安装

`brew install imagemagick`

安装以后, 提供了很多可执行命令

### identify

获取图片信息

```
 % identify post-bg-2015.jpg
 post-bg-2015.jpg JPEG 1900x870 1900x870+0+0 8-bit sRGB 232KB 0.010u 0:00.009
 ```

图片信息按照`宽*高`

如果只需要获取宽高: `identify -format "%wx%h" image.png`

### convert

* 图片的质量

  -quality: 质量值为0-100之间的数值,数字越大,质量越好,一般指定70-80,基本上看不出前后的差别

  jpg默认99,png默认75

  `convert -resize 800x800 -quality 50 原始图片 新图片`

* 放大/缩小

  等比缩放: `convert 原图 -resize 200x200 新图` 仅满足一边的缩放, 另一边按照原始比例缩放

  比例缩放: `convert image.png -resize 50% resize.png`

  多次缩放: `convert image.png -resize 50%  -resize 200%  resize.png` 先缩小一半在放大一倍, 效果变模糊


* 裁剪

  `convert image.png -crop 100x100+50+50 crop.png`  格式: `结果宽*结果高+x轴起点+y轴起点`

  坐标零点默认是左上, 可以用参数`-gravity` 指定: northwest：左上角，north：上边中间，northeast：右上角，east：右边中间

* 旋转

  -rotate

* 合并

  -compose
