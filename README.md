# Gemini API Web 客户端

本项目是一个基于 Vue3 和 Google Gemini API 构建的简洁易用的 Web 应用，旨在帮助用户快速体验和使用 Gemini 的强大功能。它采用 MIT 协议开源，允许您自由地使用、修改和分发。

## 特点

*   **易于使用:** 无需任何配置文件，即可直接在界面上设置 API 地址、API 密钥、模型、提示词、温度等参数。
*   **功能丰富:** 支持设置提示词，并可控制温度参数以调整生成内容的创造性。集成了网页搜索功能，可让模型参考最新的信息。
*   **现代技术栈:** 使用 Vue3、npm 构建，保证了应用的性能和可维护性。
*   **轻量级:** 界面简洁直观，核心功能集中。
*   **在线体验:** 您可以通过 [web.gemini.lqyss.top](https://web.gemini.lqyss.top) 在线体验本客户端。
*   **MIT 协议:** 允许自由使用、修改和分发。

## 技术栈

*   **Vue3:** 前端框架
*   **@google/generative-ai:** Google Gemini API 的 JavaScript 客户端
*   **lz-string:** 用于字符串压缩
*   **markdown-it:** Markdown 解析器

## 如何使用

1.  克隆或下载本项目代码。
2.  使用 npm 或 yarn 安装依赖：

    ```bash
    npm install
    # 或
    yarn install
    ```

3.  启动开发服务器：

    ```bash
    npm run dev
    # 或
    yarn dev
    ```

4.  在浏览器中打开 `http://localhost:[端口]` (端口号通常是 5173)。
5.  在界面上设置你的 Gemini API 地址和密钥。
6.  开始使用 Gemini API!

## 界面说明

模拟了 iOS 短信界面，并提供以下可配置项：

*   **API 地址:** Gemini API 的服务地址。
*   **API 密钥:** 你的 Google Gemini API 密钥。
*   **模型:** 使用的 Gemini 模型名称。
*   **提示词:** 你希望模型遵循的指令或上下文。
*   **温度:** 控制生成内容的随机性，值越高越随机。
*   **搜索:**  控制是否在生成内容时参考网页搜索结果。

## 贡献

欢迎提交 issue 和 pull request!

## 许可证

MIT