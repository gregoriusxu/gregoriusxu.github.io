---
layout:     post
title:      如何用Puppeteer模拟访问带有验证码的网站
subtitle:   Puppeteer介绍
date:       2019-07-14
author:     gregorius
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - 前端
---

#### 什么是[Puppeteer](https://github.com/GoogleChrome/puppeteer)？

一个提供高级API的node库，能够通过devtool控制headless模式的chrome或者chromium，它可以在headless模式下模拟任何的人为操作。简单来说是用来做模拟测试和爬虫用的，还有一个selenium和它类似，但是selenium更加的多元化，所用的技术栈更加的复杂，今天我们主要是介绍Pupperteer,想了解selenium可以参考官网地址[selenium](https://www.seleniumhq.org/)。

#### [TypeScript](https://www.typescriptlang.org/)

Puppeteer 的技术栈是nodejs，为了写代码更舒服，我们直接上TypeScript，作为一名后端开发人员转nodejs，从TypeScript开始是最好的，静态的方法检查，面向对象的开发方式让你丝毫找不到跨行的感觉。

#### Puppeteer入门

##### 环境准备

对于ide来说你可以根据自己的喜好随意选择，我这边习惯了vscode，所以以vscode平台作为平台进行介绍。

首先安装puppeteer，安装过程中会自动下载chrome无头浏览器

        $ npm install --save puppeteer
        $ npm install --save-dev @types/puppeteer

安装完成后来个简单示例

        import * as puppeteer from 'puppeteer';//引入puppeteer包

        (async () => {
            const browser = await puppeteer.launch(); //启动puppeteer引擎
            const page = await browser.newPage(); //开启一个新页面
            await page.goto('https://google.com'); //访问google.com
            await page.pdf({path: 'google.pdf'}); //将网页导出为pdf

            await browser.close(); //关闭浏览器
        })();

##### 网页模拟访问示例

首先在vscode新建一个example.ts的文件

创建页面


    const createPage = async (browser: puppeteer.Browser): Promise<puppeteer.Page> => {
        const page = await browser.newPage()

        return new Promise<puppeteer.Page>(resolve => {
            resolve(page);
        });
    };


开始浏览器并访问page元素

        const browser = await puppeteer.launch({ headless: false, devtools: false })
            const page = createPage(browser);
            page.then((res: any) => {
                visit(res).catch((err: any) => {
                    console.log(err);
                    browser.close();
                });
            })

        async function visit(page: puppeteer.Page): Promise<void> {
            //page.on('console', msg => console.log('PAGE LOG:', msg.text()));
            await page.setViewport({ width: 1280, height: 800 })
            await page.goto('https://www.google.com')
            await login(page)

            await page.waitForSelector('#primary_nav_wrap > ul > li:nth-child(2) > a', { visible: true })
        }

如果网站需要登录，编写login方法

        async function login(page: puppeteer.Page): Promise<void>
        {
            await page.waitForSelector('#username')
            await page.type('#username', "xxx")
            await page.waitForSelector('#password')
            await page.type('#password', "xxx")

            await page.waitForSelector('#bodybg> input[type=submit]')
            page.click('#bodybg > table.mainbody > input[type=submit]')
            await page.waitForNavigation({waitUntil: 'load'})`
        }

由于puppeteer是基于异步模式进行编程的，所以在调用方法前一定要加一个await，不然会出现找不到元素的问题，还有一点page.click方法和page.waitForNavigation存在竞争访问，所以click不用写await，不然也会出错。

waitForSelector的内容可以打开chrome，F12打开开发工具，选择要模拟的元素，点击右键检查，copy selector可以拿到。

对于简单验证码的识别，可以使用老牌的识别软件[tesseract](https://github.com/tesseract-ocr/tesseract)，为提高识别精度，可以自己制作样本进行训练，具体方法可以参考下面两篇文章：
- https://www.cnblogs.com/cnlian/p/5765871.html
- https://www.jianshu.com/p/5f847d8089ce

#### TypeScript使用技巧

在终端输入tsc -w，当ts文件保存时会自动生成对应的js文件，运行时直接node xxx.js就可以了，如果需要调试代码可以参考下面这篇文章：
- https://segmentfault.com/a/1190000011935122