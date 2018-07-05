# Puppeteer
![img](https://user-images.githubusercontent.com/10379601/29446482-04f7036a-841f-11e7-9872-91d1fc2ea683.png)

[puppeteer官网](https://github.com/GoogleChrome/puppeteer)

## 一、puppeteer是干什么的？
> 引用puppeteer官网解释： Most things that you can do manually in the browser can be done using Puppeteer! 

* 生成页面的屏幕截图和PDF。
* 抓取SPA并生成预渲染内容（即“SSR”）。
* 自动表单提交，UI测试，键盘输入等。
* 创建最新的自动化测试环境。 使用最新的JavaScript和浏览器功能直接在最新版本的Chrome中运行测试。
* 捕获您网站的[时间线跟踪]

## 二、常用API
> -  `page.setViewport()` 设置获取屏幕大小，默认获取屏幕大小为800px * 600px

> - `page.pdf(路径，大小)` 保存为pdf格式图片  
>       - 举例：`page.pdf({path: 'hn.pdf', format: 'A4'});`

> -  `page.evaluate(fn)` 执行chrome的api  
>    - 举例：
>         ```js
>          await page.evaluate(() => {
>              return {
>                  width: document.documentElement.clientWidth,
>                  height: document.documentElement.clientHeight,
>             deviceScaleFactor: window.deivcePixelRatio
>             };        
>         })
>         ```

> - `puppeteer.launch({headless: false});` 打开浏览器，默认值是true

> [更多API](https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md)

## 三、举个栗子：截取屏幕
### 3.1 代码
```js
const puppeteer = require('puppeteer');
// 引用default.js的sceenshot路径，将截取的屏幕pdf保存到该路径下。
const { screenshot } = require('./config/default.js');

(async () => {
  // 获取browser实例
  const browser = await puppeteer.launch();
  // 获取浏览器tab页面实例
  const page = await browser.newPage();
  // 链接到百度首页
  await page.goto('https://www.baidu.com');
  // 截屏
  await page.screenshot({
    // 将截屏按时间戳保存到指定路径下。
    path: `${screenshot}/${Date.now()}.png`
  });
  // 关闭
  await browser.close();
})();
```
### 3.2 然后执行命令
```bash
node src/screenshot.js
```
### 3.3 最后在screenshot文件指定路径下生成百度首页的截屏。

## 四、爬取百度图片列表
### 4.1 实现思路
1. 模拟用户打开浏览器
2. 模拟打开tab页
3. 模拟前往百度图片页面
4. 模拟focus到输入框，输入查询值, 点击查询按钮
5. 抓取图片
6. 通过writeFile，将图片下载到指定路径下。

### 4.2 目录结构
```js
.
|-mn
|-src
|   |-config
|   |    |-default.js     
|   |-helper
|   |    |-srcToImg.js
|   |-mn.js
|-package.json
```
### 4.3 mn.js 主文件

```js
const puppeteer = require('puppeteer');
const { mn } = require('./config/default');
const srcToImg = require('./helper/srcToImg');
(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  await page.goto('https://image.baidu.com');
  console.log('go to https://image.baidu.com');
  await page.setViewport({
    width: 1920,
    height: 1080
  });
  console.log('reset viewport');
  await page.focus('#kw');
  await page.keyboard.sendCharacter('狗');
  await page.click('.s_search');
  console.log('go to search list');
  page.on('load', async () => {
    console.log('page loading done, start fetch ...');
    const srcs = await page.evaluate(() => {
      const images = document.querySelectorAll('img.main_img');
      return Array.prototype.map.call(images, img => img.src);
    });
    console.log(`get ${srcs.length} image, start download`);
    
    srcs.forEach(async (src) => {
      await srcToImg(src, mn);
    });
    await browser.close();
  })
})();
```

### 4.4 default.js 路径
```js
const path = require('path');
module.exports = {
  screenshot: path.resolve(__dirname, '../../screenshot'),
  mn: path.resolve(__dirname, '../../mn')
}
```
### 4.5 srcToImg.js 解析图片地址

```js 
const http = require('http');
const https = require('https');
const fs = require('fs');
const path = require('path');
const { promisify } = require('util');
const writeFile = promisify(fs.writeFile);

module.exports = async(src, dir) => {
  if(/\.(jpg|png|gif)$/.test(src)) {
    await urlToImg(src, dir);
  }else {
    await base64ToImg(src, dir); 
  }
}

// 识别src为http或者https的图片
const urlToImg = promisify((url, dir, callback) => {
  const mod = /^https:/.test(url) ? https : http;
  const ext = path.extname(url);
  const file = path.join(dir, `${Date.now()}${ext}`);
  mod.get(url, res => {
    res.pipe(fs.createWriteStream(file))
      .on('finish', () => {
        callback();
        console.log(file);
      })
  })
})

// 识别src为base64地址的图片
const base64ToImg = async (base64Str, dir) => {
  // data: image/jpeg;base64,/raegreagearg
  const matchs = base64Str.match(/^data:(.+?);base64,(.+)$/);
  try {
    const ext = matches[1].split('/')[1]
      .replace('jpeg', 'jpg');
    const file = path.join(dir, `${Date.now()}.${ext}`);
    await writeFile(file, match[2], 'base64');
    console.log(file);
  } catch (ex) {
    console.log('非法 base64 字符串');
  }
}
```

### 4.6 最终在mn文件夹中存入爬取到的图片。

```bash
go to https://image.baidu.com
reset viewport
go to search list
page loading done, start fetch ...
get 46 image, start download
非法 base64 字符串
非法 base64 字符串
非法 base64 字符串
非法 base64 字符串
非法 base64 字符串
非法 base64 字符串
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351397.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351396.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351398.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351400.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351405.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351386.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351399.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351405.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351405.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351402.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351412.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351413.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351403.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351398.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351399.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351403.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351406.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351401.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351408.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351404.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351414.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351400.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351402.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351413.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351408.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351414.jpg
/Users/lius/Desktop/web spider/headless-crawler/headless_crawler/mn/1530800351413.jpg
......

```

### 4.7 mn文件夹下
![img](http://www.liuzhaoxu.cn/static/upload/20180705/ofsjYkDMppTtCQoOxhbgLi0G.png)

