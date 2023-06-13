---
title: 用chatgpt构建自定义聊天机器人之javascript版本
date: 2023-06-10 16:46:21
tags:
---

## QuickStart

根据 OpenAI 和吴恩达的[开发人员 ChatGPT 提示工程限时免费入门课程](https://learn.deeplearning.ai/chatgpt-prompt-eng/lesson/8/chatbot)，记录的如何利用 chatgpt 编写自定义聊天机器人。
原教程使用 python 语言，我自己写的时候用的 javascript。相关 api 可查看 [openAI 官方 API 文档](https://platform.openai.com/docs/api-reference/introduction)

## 注册 API KEY

到[这里](https://platform.openai.com/account/api-keys)注册一个 apiKey
![](./img/chatbot/apiKey.png)
![](./img/chatbot/createKey.png)
记录下 apiKey 后面要用。

## 安装依赖

```javascript
npm install openai
```

```javascript
import { Configuration, OpenAIApi } from "openai";
const configuration = new Configuration({
  organization: "org-JyvuA2Fv4RCl7xAO9z5OvTD7",
  apiKey: "your apiKey here", // 这里填入apiKey
});
const openai = new OpenAIApi(configuration);
```

## 参考资料

[DeepLearning.AI 免费课程](https://learn.deeplearning.ai/chatgpt-prompt-eng/lesson/8/chatbot)
