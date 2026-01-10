+++
date = '2026-01-10T21:51:12+08:00'
draft = false
title = '抓取钉钉智能出勤的数据'
tags = ["工程"]
categories = ["工程"]
math = true
+++
众所周知，钉钉的智能出勤在基础配置页面，在有权限但不足时，你可以看到所有人的信息，但会不给你批量导出，但是你还是能读取和直接复制（如果有这个权限的话）。

但是如果没有权限还想全部导出怎么办捏？

## 抓取和重发网络请求

考虑能否让一页上加载全部的人员信息。

先看眼它数据的网络请求是怎么发过来的。调整页面右下方的 `每页显示`，提前打卡 F12 并先清掉网络请求，看到发了个 `queryThroughView.json` 的 GET 请求。它的请求 URL 大概长这个样子（关键信息已经打码）：

```
https://psjvlw.aliwork.com/dingtalk/web/APP_MAAH47PDBDYNVWONJDU1/query/dataQuery/queryThroughView.json?_api=DataManage.queryThroughView&_mock=false&_csrf_token=***&_locale_time_zone_offset=28800000&searchField=%5B%5D&pageSize=50&currentPage=1&page=1&limit=50&manageUuid=FORM-***&appType=APP_MAAH47PDBDYNVWONJDU1&formUuid=FORM-***&associatedFormInstId=&associatedFormUuid=&allText=&viewUuid=VIEW-***&filterRule=%7B%22condition%22%3A%22%22%2C%22rules%22%3A%5B%5D%7D&sort=%5B%5D&manageScene=WORKBENCH&subTableLimit=10&logicOperator=&_stamp=***
```

发现 `limit` 和 `pageSize` 是写在请求里的，尝试改一下再发个网络请求试试看：

![image](https://image.langningchen.com/xdbpyjavfadfvvnpmivrlaorkalkfokk)

有点倒闭，确实请求成功了某页的 JSON 格式数据，但是返回的 `limit` 还是 $50$。

## 爬取各个页面数据

但是我们可以改变 `page` 的值，爬完所有 JSON 数据再一起导出。

（本部分代码由 Gemini 所写，在此感谢）

```js
const baseUrl = 'https://psjvlw.aliwork.com/dingtalk/web/APP_MAAH47PDBDYNVWONJDU1/query/dataQuery/queryThroughView.json?_api=DataManage.queryThroughView&_mock=false&_csrf_token=***&_locale_time_zone_offset=28800000&searchField=%5B%5D&pageSize=50&currentPage=1&page=1&limit=50&manageUuid=FORM-***&appType=APP_MAAH47PDBDYNVWONJDU1&formUuid=FORM-***&associatedFormInstId=&associatedFormUuid=&allText=&viewUuid=VIEW-***&filterRule=%7B%22condition%22%3A%22%22%2C%22rules%22%3A%5B%5D%7D&sort=%5B%5D&manageScene=WORKBENCH&subTableLimit=10&logicOperator=&_stamp=***';

// 定义分页参数
const pageSize = 50;
const totalCount = 1674; // 从你的响应中获取
const totalPages = Math.ceil(totalCount / pageSize);
const allData = [];

console.log(`总数据量: ${totalCount} 条. 总页数: ${totalPages} 页. 开始抓取...`);

async function fetchAllData() {
    for (let page = 1; page <= totalPages; page++) {
        // 1. 构建每页的 URL
        const pageUrl = baseUrl
            .replace(/currentPage=\d+/g, `currentPage=${page}`)
            .replace(/page=\d+/g, `page=${page}`);
            
        console.log(`正在请求第 ${page} 页...`);

        try {
            // 2. 发送请求
            const response = await fetch(pageUrl);
            const json = await response.json();
            
            // 3. 检查并存储数据
            if (json.success && json.content && json.content.values) {
                allData.push(...json.content.values);
                console.log(`第 ${page} 页成功，已收集 ${allData.length} 条数据。`);
            } else {
                console.error(`第 ${page} 页请求成功，但数据结构错误或无数据。`, json);
                // 遇到错误，可以选择 break; 或 continue;
            }

            // ⚠️ 增加延迟防止被服务器限流 (可选，但推荐)
            await new Promise(resolve => setTimeout(resolve, 500)); 

        } catch (error) {
            console.error(`请求第 ${page} 页失败:`, error);
            // 遇到错误，可以选择 break; 或 continue;
            await new Promise(resolve => setTimeout(resolve, 2000)); // 错误时等待更久
        }
    }

    console.log("=======================================");
    console.log(`✅ 所有数据抓取完成! 总计: ${allData.length} 条.`);
    
    // 4. 将最终数据对象打印出来，方便复制
    console.log("最终数据对象:", allData); 
    
    return allData;
}

// 运行函数
fetchAllData();
```

抓取完后就得到了 JSON 格式的所有数据，直接复制出来（虽然人类可读性极差）。

如果有需求可以想办法清洗一下数据或转成 excel 表格。