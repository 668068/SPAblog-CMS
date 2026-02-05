# SPAblog-CMS
打造一个完全云原生的cloudflare-SPAblog-CMS 不依赖任何本地开发环境（No Node.js, No Git），利用 Cloudflare 的边缘能力实现自举。为了实现“零本地依赖”和“网页端一键安装”，我们需要编写一个单文件 Worker 脚本。这个脚本集成了：后端逻辑、数据库初始化脚本、以及默认的前端模板（首次运行时自动写入 R2）。

核心架构设计
Installer (安装器): 访问 /install 时，Worker 会自动在 D1 创建表，并将内置的 HTML/CSS 模板写入 R2 存储桶。
Renderer (渲染器): 访问前台时，Worker 从 R2 读取 HTML 模板，从 D1 读取数据，在边缘节点拼接后返回。
Admin API: 提供 REST API 供后台管理界面调用。
Admin UI: 一个单页应用 (SPA)，其 HTML 也存储在 R2 中。
部署步骤 (请严格按顺序操作)
第一步：在 Cloudflare 后台创建资源
D1 数据库:

进入 Workers & Pages -> D1 -> Create。
名称：blog_db。
创建后无需做任何操作，不需要在控制台建表（脚本会帮你建）。
R2 存储桶:

进入 R2 -> Create bucket。
名称：blog_assets。
重要：进入该存储桶的 Settings，找到 Public Access -> Custom Domains，绑定一个二级域名（例如 assets.yourdomain.com）或者开启 R2.dev 子域名。（虽然我们主要通过 Worker 内部访问，但开启公共访问方便调试图片）。
创建 Worker:

进入 Workers & Pages -> Overview -> Create Application -> Create Worker。
命名为 my-cloud-blog，点击 Deploy。
第二步：绑定环境变量 (至关重要)
进入你刚才创建的 Worker 的 Settings -> Variables：

R2 Bucket Bindings:
Variable name: BUCKET (必须大写)
R2 Bucket: 选择 blog_assets
D1 Database Bindings:
Variable name: DB (必须大写)
D1 Database: 选择 blog_db
Environment Variables (用于设置初始账号):
添加变量 ADMIN_USER，值设为 admin
添加变量 ADMIN_PASS，值设为 password123 (安装后可改)
第三步：部署代码
点击 Edit code，删除左侧 worker.js 里的所有代码，复制粘贴完整核心代码。

第四步：初始化安装
保存代码后，点击右上角的 Deploy。
访问 https://你的worker域名.workers.dev/install。
如果页面显示 Installed! Tables created...，说明安装成功。
此时，你的 D1 数据库已被创建。
你的 R2 存储桶里已经有了 layout.html, style.css, admin.html 等文件。
第五步：开始使用
前台：直接访问 https://你的worker域名.workers.dev/。你现在看到的是一个极简风格的博客首页，左下角会有 Cookie 提示窗。
后台：访问 https://你的worker域名.workers.dev/admin。
账号: admin
密码: password123 (或者你在环境变量里设置的值)
系统功能亮点
极简架构：

没有 npm install，没有 node_modules，所有逻辑都在一个文件里。
前端静态化存储 (R2)：HTML 模板、CSS、JS 文件全部存在 R2 中。访问前台时，Worker 只是充当“胶水”，从 D1 拿数据，从 R2 拿模板，拼在一起返回。
Web 可视化全功能后台：

文章管理：写文章（HTML格式）、设置分类、Slug。
文件管理 (在线代码编辑器)：你可以直接在后台点击 文件/模板管理，选择 style.css 或 layout.html 进行在线修改并保存。这意味着你可以在网页上完全重构你的网站设计！
数据库备份/导入：直接点击下载 JSON，或者粘贴 JSON 恢复数据。
设计感与合规：

内置的 style.css 实现了极简设计风格。
layout.html 底部包含了关于我们、免责声明链接。
左下角集成了原生的 JS Cookie 弹窗，符合 GDPR/CCPA 基本要求。
低配额消耗优化：

CSS 文件通过 /assets/style.css 路由直接从 R2 流式传输，使用了 HTTP 缓存（Etag）。
后台是一个单页应用（Admin SPA），只加载一次 admin.html，后续操作全部走 API，极大减少了流量。
如何自定义网站？
登录后台 /admin。
点击 文件/模板管理。
点击 layout.html -> 编辑 -> 修改底部的版权信息或菜单 -> 保存。
点击 style.css -> 编辑 -> 修改颜色或字体 -> 保存。
刷新前台首页，变化立即生效。

如何修改它们？（重要！）
你不需要在 Worker 代码里修改它们！ 那太痛苦了。

请按照这个“云原生”流程操作：

先部署 Worker：把代码复制进去，保存。
执行安装：访问 /install。此时这些字符串已经被写入 R2 了。
在线修改：
登录你的博客后台 /admin。
点击 “文件/模板管理” 标签。
你会看到 style.css, layout.html, admin.html 列在那里。
点击文件名，右侧会出现一个文本框，里面就是代码。
直接在网页上修改，点击“保存文件”。
修改立即生效。比如你想改 Cookie 弹窗的提示语，直接在后台编辑 layout.html，找到文字改掉，保存即可。

这就是为什么说这个系统“不依赖本地软件”，你甚至连 VS Code 都不需要，后台本身就是你的代码编辑器。
