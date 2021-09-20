# 将 Rust 和 WebAssembly 部署到生产环境 

> **⚡ 部署使用 Rust 和 WebAssembly 构建的 Web 应用程序几乎与部署任何其他 Web 应用程序相同！**

要在客户端部署使用 Rust 生成的 WebAssembly 的 Web 应用程序，请将构建的 Web 应用程序的文件复制到您的生产服务器的文件系统，并配置您的 HTTP 服务器以使其可访问。

## 确保您的 HTTP 服务器使用 `application/wasm` MIME 类型 

为了最快的页面加载，您需要使用 [`WebAssembly.instantiateStreaming` 函数][instantiateStreaming] 通过网络传输管道化 wasm 编译和实例化（或确保您的捆绑器能够使用该函数）。 但是，`instantiateStreaming` 要求 HTTP 响应设置了 `application/wasm` [MIME 类型][MIME type]，否则会抛出错误。 

* [如何为 Apache HTTP 服务器配置 MIME 类型][apache-mime]
* [如何为 NGINX HTTP 服务器配置 MIME 类型][nginx-mime]

[instantiateStreaming]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/instantiateStreaming
[MIME type]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types
[apache-mime]: https://httpd.apache.org/docs/2.4/mod/mod_mime.html#addtype
[nginx-mime]: https://nginx.org/en/docs/http/ngx_http_core_module.html#types

## 更多资源

* [生产中 Webpack 的最佳实践。][webpack-prod] 许多 Rust 和 WebAssembly 项目使用 Webpack 来捆绑其 Rust 生成的 WebAssembly、JavaScript、CSS 和 HTML。 本指南提供了在部署到生产环境时充分利用 Webpack 的技巧。
* [Apache 文档。][apache] Apache 是一种用于生产的流行 HTTP 服务器。
* [NGINX 文档。][nginx] NGINX 是一种用于生产的流行 HTTP 服务器。

[webpack-prod]: https://webpack.js.org/guides/production/
[apache]: https://httpd.apache.org/docs/
[nginx]: https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/
