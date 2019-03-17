---
layout: 前端解析Excel文件
title: js-excel
date: 2019-03-17 15:26:26
tags:
  - javascript

categories: 前端基础
---

## 纯JavaScript前端解析Excel文件

通常情况下，读取Excel都有由后台来处理，不过会出现需要前端直接解析文件然后再校验内容格式是否正确。
在这种情况下，只需通过 [xlsc](https://www.npmjs.com/package/xlsx) 就能实现。

### 解析Excel文件

使用 `FileReader` 对象读取指定的 `File` 和 `Blob` 对象指定要读取的文件或者数据，在 `FileReader` 对象加载结束（onload事件）后进行Excel数据的处理。

整个Excel数据处理中，主要是以下四个对象和函数：

1. workbook对象，整个xlsx文件
2. worksheet对象，xlsx文件中的sheet，一个workbook由多个worksheet组成
3. XLSX.utils.sheet_to_json方法，将worksheet转化为JSON数据
4. XLSX.utils.json_to_sheet方法，将JSON数据转换成worksheet对象

```js
import XLSX from 'xlsx';

const reader = new FileReader();
reader.onload = function (event) {
  // 获取 workbook
  const wb = XLSX.read(event.target.result, { type: 'binary' });
  // 获取 workbook 第一个 worksheet 中的数据
  const ws = XLSX.utils.sheet_to_json(wb.Sheets[wb.SheetNames[0]]);
  // ...
}
// 读取原始 file 或者 blob，并将该文件转化为原始二进制数据
reader.readAsBinaryString(file);
```

### JSON数据转化为Excel文件

```js
import XLSX from "xlsx";

// 创建 worksheet
const ws = XLSX.utils.json_to_sheet(data);

// 新建空workbook，然后加入worksheet
const wb = XLSX.utils.book_new();
XLSX.utils.book_append_sheet(wb, ws, "sheet1");

// 生成 xlsx 文件
XLSX.writeFile(wb, "sheetjs.xlsx");
```
