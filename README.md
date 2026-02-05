# SPAblog-CMS
打造一个完全云原生的cloudflare-SPAblog-CMS 不依赖任何本地开发环境（No Node.js, No Git），利用 Cloudflare 的边缘能力实现自举。为了实现“零本地依赖”和“网页端一键安装”，我们需要编写一个单文件 Worker 脚本。这个脚本集成了：后端逻辑、数据库初始化脚本、以及默认的前端模板（首次运行时自动写入 R2）。
