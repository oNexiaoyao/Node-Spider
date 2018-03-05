### 基于Node.js的爬虫项目

实现的最终结果：通过爬取`https://www.cnblogs.com/`里面的文章获取对应作者的相关信息。
目的：学习node搭建服务器、分析网页请求并模拟请求、熟悉http协议相关、node写入文件等功能

#### 项目文件结构说明

`index.js`：项目的启动（入口）文件
`server.js`：主文件
`package.json`：模块依赖文件及相关配置

#### 搭建node服务器

*默认开始本步骤前已经安装好node环境*

server.js：

引入node内置的http模块：

```javascript

const http = require('http');

```

创建一个简单的node服务器：

```javascript

// 指定监听的port及hostname

const listenPort = 3000;
const listenHostName = '127.0.0.1';

http.createServer(function (request, response) {
    response.writeHead(200, {'Content-Type': 'text/plain;charset=utf-8'});
    response.end("hello world\n");
}).listen(listenPort, listenHostName, () => {
    console.log(`Server running at http:// ${listenHostName}:${listenPort}/`);
});

```

测试服务器是否搭建成功：在命令行进入相应的目录文件，使用`node server.js`命令启动服务器，然后浏览器打开`127.0.0.1:3000`访问，网页出现`hello world`即表示服务器创建成功。

#### 使用到的模块

**node内置模块**

* `http`：用来搭建服务器
* `fs`： 文件系统，用来写入和读取文件
* `url`：针对url处理

使用方法：代码中使用`require()`直接引入。

**第三方模块**

* `eventproxy`：控制并发
* `async`：控制并发请求的数量（防止被封号）
* `superagent`：实现客户端请求代理（类似封装的ajax请求）
* `cheerio`：针对html片段实现jquery功能操作

使用方法：先安装到项目中，然后在代码中使用`require()`引入。

安装方法：

先通过`npm init`来生成`package.json`文件。

然后通过npm 安装依赖模块的方法来安装对应的模块：

```javascript

npm install eventproxy --save-dev

```

下面列出相关模块使用学习参考链接：


1. 使用`eventproxy`控制并发。[参考理解](https://www.jianshu.com/p/a86bd4bd48c9)

2. 使用`async`控制并发请求数量防止被封IP。[参考理解](https://github.com/alsotang/async_demo/blob/master/map.js)

3. 使用`superagent`实现客户端请求代理模块。[参考理解](http://cnodejs.org/topic/5378720ed6e2d16149fa16bd)

4. 使用`cheerio`实现将请求回来的html片段实现可以类似jquery使用的功能。[参考理解](https://www.npmjs.com/package/cheerio)

思考： `eventproxy`、`async`和`promise`的异同？

#### 具体代码展示

index.js：

```javascript

/**
 * @author lupinggan
 * @description 项目启动文件
 */

const server = require('./server.js');
server.start();

```

server.js：

```javascript

/**
 * @author lupinggan
 * @description node爬虫服务器文件-一次性抓取很多文章获取相关信息
 * 
 */

// 引入node内置的http模块

const http = require('http');

// 引入node的文件系统模块

const fs = require('fs');

// 引入node内置的url模块

const url = require('url');

// 引入eventproxy来控制并发

const eventproxy = require('eventproxy');

// 引入async模块来控制并发的请求数量以防止被封号

const async = require('async');

// 引入superagent来实现客户端请求代理

const superagent = require('superagent');

// 引入cheerio实现jquery功能操作

const cheerio = require('cheerio');

// 指定监听的port及hostname

const listenPort = 3000;
const listenHostName = '127.0.0.1';

// 定义爬取的入口地址

const catchFirstUrl = 'http://www.cnblogs.com/';

// 定义相关变量来存储数据

const urlsArry = [], // 需要爬取的网址(每篇文章访问的url)
      deleteRepeat = {} , // 用来存储作者姓名的字典
      catchData = [], // 存放爬取数据
      pageUrls = [], // 存放收集文章页面网站
      pageNum = 5, // 要爬取的文章的页数
      singlePagePostNum = 20, // 单页面的文章数量
      startDate = new Date(), // 开始时间
      endDate = false; // 结束时间

// 实例化eventproxy()

const ep = new eventproxy();

for(let i=0; i < pageNum; i++){
    // 通过抓包工具分析，每页的文章列表数据获取是通过ajax post方式获取到的
    pageUrls.push("https://www.cnblogs.com/mvc/AggSite/PostList.aspx");
    // pageUrls.push("https://www.cnblogs.com?CategoryId=808&CategoryType='SiteHome'&ItemListActionName='PostList'&PageIndex="+ (i + 1) +"&ParentCategoryId=0");
    // pageUrls.push("https://www.cnblogs.com/mvc/AggSite/PostList.aspx?CategoryId=808&CategoryType='SiteHome'&ItemListActionName='PostList'&PageIndex="+ (i + 1) +"&ParentCategoryId=0");
}

// 创建服务器

// http.createServer(function (request, response) {
//     response.writeHead(200, {'Content-Type': 'text/plain;charset=utf-8'});
//     response.end("这里是使用node开发的爬虫 hello world\n");
// }).listen(listenPort, listenHostName, () => {
//     console.log(`Server running at http:// ${listenHostName}:${listenPort}/`);
// });

//  判断是否有重复的作者

const isAuthorRepeat = (authorName) => {
    if(deleteRepeat[authorName] == undefined ) {
        // 说明字典表中还没有该作者的信息
        deleteRepeat[authorName] = 1;
        return 0;
    }else if(deleteRepeat[authorName] == 1) {
        // 说明字典表中已经存在该作者的信息
        return 1;
    }
}

// 作者详细信息获取

const personInfo = (url) => {

    // 存放作者相关信息
    const infoObj = {};
    superagent.get(url)
    .end(function(err,res) {
        if(err){
            console.error(err);
            return;
        }
        const $ = cheerio.load(res.text),
              info = $('#profile_block a');

        infoObj.name = info.eq(0).text();
        infoObj.age = info.eq(1).text();
        if (info.length == 4) {
            infoObj.fans = info.eq(2).text();
            infoObj.focus = info.eq(3).text();
            infoObj.honour = 0
        } else if (info.length == 5) {
            infoObj.fans = info.eq(3).text();
            infoObj.focus = info.eq(4).text();
            infoObj.honour = 1;
        }
        catchData.push(infoObj);
    })
}

// 主程序

const start = () => {
    // 创建服务器
    http.createServer(function (request, response) {
        
        // 浏览器一次刷新会导致这里请求两次，原因是浏览器会默认一次请求favicon.ico(网页标签上的那个小图标)
        // console.log(url.parse(request.url));
        if(url.parse(request.url).path == '/favicon.ico'){
            return;
        }
        // 设置字符编码防止出现中文乱码
        response.writeHead(200, {'Content-Type': 'text/html;charset=utf-8'});
        // 控制并发,当所有的请求完成后，触发下面的函数
        ep.after('BlogArticleHtml', pageUrls.length*singlePagePostNum, function(articleUrls) {
            // articleUrls是一个数组，是通过ep.emit传过来的的articleUrl数组集合

            // 打印输出结果
            // response.write('输出结果：');
            // response.write('<br/>');
            // response.write('共' + articleUrls.length +'篇文章<br/>');
            // for(let i = 0; i < articleUrls.length; i++){
            //     response.write('第' + i + '篇：' + articleUrls[i] + '<br/>');
            // }

            // 对数组进行检查--去重处理
            // 通过针对爬取到的数组去重处理，发现通过"https://www.cnblogs.com/#p" + i不能实际爬取到所有真正的blog地址，因为我们需要对请求使用第三方抓包工具处理分析
            const _articleUrls = articleUrls.filter((currentvalue, index, arr) => {
                return arr.indexOf(currentvalue) === index;
            });
            response.write('输出结果：');
            response.write('<br/>');
            response.write('共' + _articleUrls.length +'篇文章<br/>');
            // for(let i = 0; i < _articleUrls.length; i++) {
            //     response.write('第' + i + '篇：' + _articleUrls[i] + '<br/>');
            // }

            // 对爬取回来的所以的文章url地址进行请求，进而获取需要的作者信息 
            // 由于爬取回来的articleUrls数组长度可能会非常大。因此当我们针对其中的每个具体地址去发送请求的过程中，需要
            // 控制并发的请求数量，以防止被封号或者封ID

            // 我们这里使用async模块来控制并发的数量，详细使用请参考：
            // github地址：https://github.com/caolan/async
            // 使用demo: https://github.com/alsotang/async_demo/blob/master/map.js
            // mapLimit(arr, limit, iterator, callback)
            
            // 控制并发数
            let curCount = 0;

            async.mapLimit(_articleUrls, 5, function(item, callback) {
        
                // 定义延迟时间
                // const delay = parseInt(2000);
                const delay = parseInt((Math.random() * 3000000) % 1000, 10);
                curCount++;
                console.log('现在的并发数是：' + curCount + '---正在抓取的是：'+ item + '延迟' + delay);
                superagent.get(item)
                .end(function(err,res){
                    // 请求错误处理
                    if(err){
                        console.error(err);
                        return;
                    }
                    
                    const $ = cheerio.load(res.text);
                    // 收集每篇文章的信息
                    const currentBlogApp = item.split('/p/')[0].split('/')[3],
                          requestId = item.split('/p/')[1].split('.')[0];
                    
                    // 这里还是使用response而不是superagent返回的res来输出
                    response.write('当前博客：' + currentBlogApp + ',' + '请求的id: '+ requestId +'<br/>');
                    response.write('当前的文章题目：'+ $('title').text() +'<br/>');

                    // 检测是否有重名-针对同一个人，他的信息获取一次就够了
                    const flag = isAuthorRepeat(currentBlogApp);
                    if(!flag){
                        // 通过抓包分析，拼接用于获取作者个人信息的url
                        const appUrl = "http://www.cnblogs.com/mvc/blog/news.aspx?blogApp="+ currentBlogApp;
                        // 博客作者详细信息获取
                        personInfo(appUrl);
                    }
                });

                setTimeout(function() {
                    curCount--;
                    callback(null, item + '请求内容');
                },delay);

            },function(err, result) {
                console.log(result);
                console.log('----------------');
                console.log(catchData);
                // 实时写入文件

                fs.writeFile('data.json', JSON.stringify(catchData), 'utf-8', (err) => {
                    if(err) {
                        console.error('写入文件有误');
                    }
                });
                // appendFile是往文件中添加，不会覆盖
                // fs.appendFile('data.json', JSON.stringify(catchData), 'utf-8', (err) => {
                //     if(err) {
                //         console.error('写入文件有误');
                //     }
                // })
            })

            
            // 结束客户端等待状态
            // response.end();
        });
        
        pageUrls.map(function(currentvalue, index, arr) {
            // 根据对文章列表的抓包分析，每页获取的blog列表数据是通过aja post请求获得的
            // superagent.get(currentvalue)
            superagent.post(currentvalue)
            .send({
                CategoryId: 808,
                CategoryType: 'SiteHome',
                ItemListActionName: 'PostList',
                PageIndex: index+1,
                ParentCategoryId: 0
            })
            .end(function(err, res) {
                if(err){
                    console.error("爬取总页数时错误：" + err);
                    return;
                }
                // res.text存放着请求返回的未解析的html
                // res还包含其他的返回属性相关请查看http://cnodejs.org/topic/5378720ed6e2d16149fa16bd
                // 为什么是使用res.text?是因为superagent是这么设计的。。
                // 将返回的html片段通过使用cheerio.load()加载后，可以类似使用jquery的方式来获取相关元素
                const $=cheerio.load(res.text);
                // 获取每一页上文章的url
                const curPageUrls = $('.titlelnk');
                for(let i = 0; i < curPageUrls.length; i++) {
                    const articleUrl = curPageUrls.eq(i).attr('href');
                    urlsArry.push(articleUrl);
                    // 使用eventproxy模块来控制并发
                    // 每执行完一次就执行一次类似计数器加一的效果
                    // 将每次的articleUrl作为参数传递给ep
                    ep.emit('BlogArticleHtml', articleUrl);
                }
            });
        });
        console.log(pageUrls.length*singlePagePostNum);

    }).listen(listenPort, listenHostName, () => {
        console.log(`Server running at http:// ${listenHostName}:${listenPort}/`);
    });
}

// 导出该模块的接口 CommonJS规范

exports.start = start;

```

#### 后续还将实现的功能：

现在已经将爬取到的数据存到了文件中，后续希望能实现将数据读取出来进行处理分析在页面以图表的形式展现出来。同时希望能做到在页面进行相关操作，从而动态获取数据动态展示数据。


本爬虫的实现是参考[原作者的示例](https://www.cnblogs.com/coco1s/p/4954063.html)来学习的。
