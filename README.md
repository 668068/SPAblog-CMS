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
点击 Edit code，删除左侧 worker.js 里的所有代码，复制粘贴下方的完整核心代码。

(代码较长，包含了迷你路由、模板引擎、安装逻辑和API)

JavaScript

/**
 * Cloud Native Blog - Single File CMS
 * Architecture: Workers + D1 + R2
 */

// --- 1. 配置与工具 ---
const CORS_HEADERS = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE, OPTIONS",
  "Access-Control-Allow-Headers": "Content-Type, Authorization",
};

// 简单的 HTML 模板渲染函数
function render(template, data) {
  return template.replace(/\{\{(.*?)\}\}/g, (match, key) => {
    return key.trim() in data ? data[key.trim()] : "";
  });
}

// 响应助手
const resJSON = (data, status = 200) => new Response(JSON.stringify(data), { status, headers: { ...CORS_HEADERS, "Content-Type": "application/json" } });
const resHTML = (html, status = 200) => new Response(html, { status, headers: { "Content-Type": "text/html; charset=utf-8" } });

// --- 2. 默认模板 (用于初始化 R2) ---
const SEED_TEMPLATES = {
  "style.css": `body{font-family:system-ui,-apple-system,sans-serif;line-height:1.6;margin:0;color:#333;background:#f9f9f9}.container{max-width:800px;margin:0 auto;padding:20px}header{display:flex;justify-content:space-between;align-items:center;margin-bottom:40px;padding-bottom:20px;border-bottom:1px solid #eee}h1{margin:0;font-size:1.5rem}nav a{margin-left:15px;color:#666;text-decoration:none}.post-card{background:#fff;padding:20px;margin-bottom:20px;border-radius:8px;box-shadow:0 2px 5px rgba(0,0,0,0.05)}.post-title{margin-top:0}.meta{color:#999;font-size:0.9rem}footer{margin-top:50px;text-align:center;font-size:0.8rem;color:#aaa}.cookie-banner{position:fixed;bottom:20px;left:20px;background:#fff;padding:15px;border-radius:8px;box-shadow:0 5px 20px rgba(0,0,0,0.1);display:flex;gap:10px;align-items:center;z-index:999;border:1px solid #eee}.btn{padding:5px 15px;border-radius:4px;cursor:pointer;border:none}.btn-accept{background:#000;color:#fff}.btn-decline{background:#eee}`,
  
  "layout.html": `<!DOCTYPE html><html lang="zh"><head><meta charset="UTF-8"><meta name="viewport" content="width=device-width, initial-scale=1.0"><title>{{title}}</title><link rel="stylesheet" href="/assets/style.css"></head><body><div class="container"><header><h1><a href="/" style="color:inherit;text-decoration:none">{{site_name}}</a></h1><nav><a href="/">首页</a><a href="/about">关于</a></nav></header><main>{{content}}</main><footer><p>&copy; 2024 {{site_name}}. All rights reserved.</p><p><a href="/disclaimer">免责声明</a> | <a href="/privacy">隐私政策</a></p></footer></div><div id="cookie-banner" class="cookie-banner" style="display:none"><p>本站使用 Cookie 以提升体验。</p><button class="btn btn-accept" onclick="acceptCookie()">接受</button><button class="btn btn-decline" onclick="declineCookie()">拒绝</button></div><script>function acceptCookie(){localStorage.setItem('cookie_consent','accepted');document.getElementById('cookie-banner').style.display='none'}function declineCookie(){localStorage.setItem('cookie_consent','declined');document.getElementById('cookie-banner').style.display='none'}if(!localStorage.getItem('cookie_consent')){document.getElementById('cookie-banner').style.display='flex'}</script></body></html>`,

  "home.html": `{{posts_list}}`,
  "post.html": `<article class="post-card"><h1 class="post-title">{{post_title}}</h1><div class="meta">分类: {{category}} | 发布于: {{date}}</div><div class="content" style="margin-top:20px">{{post_content}}</div></article>`,
  
  "admin.html": `<!DOCTYPE html><html><head><title>Admin</title><meta charset="utf-8"><style>body{font-family:sans-serif;padding:20px;max-width:1000px;margin:0 auto}input,textarea,select{width:100%;margin-bottom:10px;padding:8px}textarea{height:300px}.tab{margin-right:10px;cursor:pointer;padding:5px 10px;background:#eee;display:inline-block}.active{background:#ccc}</style></head><body><h1>后台管理</h1><div id="login-app"><input id="u" placeholder="User"><input id="p" type="password" placeholder="Pass"><button onclick="login()">Login</button></div><div id="admin-app" style="display:none"><div style="margin-bottom:20px"><span class="tab" onclick="show('posts')">文章管理</span><span class="tab" onclick="show('files')">文件/模板管理</span><span class="tab" onclick="show('backup')">备份/导入</span></div><div id="view-posts"><h3>发布/编辑文章</h3><input id="pid" type="hidden"><input id="ptitle" placeholder="标题"><input id="pslug" placeholder="Slug (URL别名)"><input id="pcat" placeholder="分类"><textarea id="pcontent" placeholder="HTML内容"></textarea><button onclick="savePost()">保存发布</button><hr><div id="post-list"></div></div><div id="view-files" style="display:none"><h3>文件管理 (R2)</h3><div id="file-list"></div><hr><h4>编辑/上传</h4><input id="fname" placeholder="文件名 (如 home.html)"><textarea id="fcontent" placeholder="文件内容"></textarea><button onclick="saveFile()">保存文件</button></div><div id="view-backup" style="display:none"><button onclick="downloadBackup()">下载数据库备份 (JSON)</button><br><br><textarea id="import-json" placeholder="粘贴JSON以导入"></textarea><button onclick="importBackup()">执行导入</button></div></div><script>
    const API = '/api'; let token = localStorage.getItem('token');
    async function req(ep, method='GET', body=null){
        const headers = {'Authorization': token}; if(body) headers['Content-Type']='application/json';
        const res = await fetch(API+ep, {method, headers, body: body?JSON.stringify(body):null});
        return res.json();
    }
    async function login(){
        const res = await req('/login', 'POST', {user:document.getElementById('u').value, pass:document.getElementById('p').value});
        if(res.token){ token=res.token; localStorage.setItem('token', token); location.reload(); } else alert('Fail');
    }
    if(token) { document.getElementById('login-app').style.display='none'; document.getElementById('admin-app').style.display='block'; loadPosts(); loadFiles(); }
    function show(id){ ['posts','files','backup'].forEach(k=>document.getElementById('view-'+k).style.display='none'); document.getElementById('view-'+id).style.display='block'; }
    async function loadPosts(){ const list = await req('/posts'); document.getElementById('post-list').innerHTML = list.map(p=>\`<div><b>\${p.title}</b> <button onclick="editP('\${p.id}')">编辑</button> <button onclick="delP('\${p.id}')">删除</button></div>\`).join(''); }
    async function savePost(){ await req('/posts', 'POST', {id:document.getElementById('pid').value, title:document.getElementById('ptitle').value, slug:document.getElementById('pslug').value, content:document.getElementById('pcontent').value, category:document.getElementById('pcat').value}); alert('Saved'); loadPosts(); }
    async function editP(id){ const p = (await req('/posts')).find(x=>x.id==id); document.getElementById('pid').value=p.id; document.getElementById('ptitle').value=p.title; document.getElementById('pslug').value=p.slug; document.getElementById('pcontent').value=p.content; document.getElementById('pcat').value=p.category; }
    async function delP(id){ if(confirm('Del?')) await req('/posts?id='+id, 'DELETE'); loadPosts(); }
    async function loadFiles(){ const list = await req('/files'); document.getElementById('file-list').innerHTML = list.objects.map(f=>\`<div>\${f.key} <button onclick="loadFile('\${f.key}')">编辑</button></div>\`).join(''); }
    async function loadFile(k){ const t = await req('/files/'+k, 'GET'); document.getElementById('fname').value=k; document.getElementById('fcontent').value=t.content; }
    async function saveFile(){ await req('/files', 'POST', {key:document.getElementById('fname').value, content:document.getElementById('fcontent').value}); alert('Saved'); loadFiles(); }
    async function downloadBackup(){ const data = await req('/backup'); const blob = new Blob([JSON.stringify(data)], {type:'application/json'}); const url = URL.createObjectURL(blob); const a = document.createElement('a'); a.href=url; a.download='backup.json'; a.click(); }
    async function importBackup(){ try{ const data = JSON.parse(document.getElementById('import-json').value); await req('/import', 'POST', data); alert('Imported'); }catch(e){alert('Invalid JSON');} }
  </script></body></html>`
};

// --- 3. 核心 Worker 逻辑 ---
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const path = url.pathname;

    // A. 安装/重置路由 (初始化数据库和R2)
    if (path === "/install") {
      // 1. 建表
      await env.DB.prepare(`CREATE TABLE IF NOT EXISTS posts (id INTEGER PRIMARY KEY AUTOINCREMENT, title TEXT, slug TEXT, content TEXT, category TEXT, created_at DATETIME DEFAULT CURRENT_TIMESTAMP)`).run();
      await env.DB.prepare(`CREATE TABLE IF NOT EXISTS settings (key TEXT PRIMARY KEY, value TEXT)`).run();
      // 2. 写入默认数据
      await env.DB.prepare(`INSERT OR IGNORE INTO settings (key, value) VALUES ('site_name', 'My Cloud Blog')`).run();
      // 3. 写入 R2 模板
      for (const [key, content] of Object.entries(SEED_TEMPLATES)) {
         await env.BUCKET.put(key, content);
      }
      return new Response("Installed! Tables created and Templates uploaded to R2.");
    }

    // B. 静态资源路由 (从 R2 读取 CSS/JS)
    if (path.startsWith("/assets/")) {
      const key = path.replace("/assets/", "");
      const object = await env.BUCKET.get(key);
      if (!object) return new Response("Not found", { status: 404 });
      const headers = new Headers();
      object.writeHttpMetadata(headers);
      headers.set("etag", object.httpEtag);
      return new Response(object.body, { headers });
    }

    // C. Admin 路由
    if (path === "/admin" || path === "/admin/login") {
      const html = await env.BUCKET.get("admin.html");
      return new Response(html.body, { headers: { "Content-Type": "text/html" } });
    }

    // D. API 路由 (Admin Backend)
    if (path.startsWith("/api")) {
      // 简单的鉴权
      if (path !== "/api/login") {
        const auth = request.headers.get("Authorization");
        // 这里只是演示，实际建议用更安全的 JWT 签名
        if (auth !== `Bearer ${env.ADMIN_PASS}`) { 
           // 简化：Token 就是 ADMIN_PASS 环境变量
           // 生产环境请务必使用 crypto.subtle 生成真实的 JWT
           return resJSON({error:"Unauthorized"}, 401); 
        }
      }

      if (path === "/api/login" && request.method === "POST") {
        const body = await request.json();
        if (body.user === env.ADMIN_USER && body.pass === env.ADMIN_PASS) {
           return resJSON({ token: `Bearer ${env.ADMIN_PASS}` });
        }
        return resJSON({ error: "Invalid" }, 403);
      }

      if (path === "/api/posts") {
        if (request.method === "GET") {
          const { results } = await env.DB.prepare("SELECT * FROM posts ORDER BY created_at DESC").all();
          return resJSON(results);
        }
        if (request.method === "POST") {
          const b = await request.json();
          if (b.id) {
            await env.DB.prepare("UPDATE posts SET title=?, slug=?, content=?, category=? WHERE id=?").bind(b.title, b.slug, b.content, b.category, b.id).run();
          } else {
            await env.DB.prepare("INSERT INTO posts (title, slug, content, category) VALUES (?, ?, ?, ?)").bind(b.title, b.slug, b.content, b.category).run();
          }
          return resJSON({ success: true });
        }
        if (request.method === "DELETE") {
          const id = url.searchParams.get("id");
          await env.DB.prepare("DELETE FROM posts WHERE id=?").bind(id).run();
          return resJSON({ success: true });
        }
      }

      if (path === "/api/files") {
         if (request.method === "GET") {
             const listed = await env.BUCKET.list();
             return resJSON(listed);
         }
         if (request.method === "POST") {
             const b = await request.json();
             await env.BUCKET.put(b.key, b.content);
             return resJSON({success:true});
         }
      }
      
      if (path.startsWith("/api/files/")) { // 获取单个文件内容用于编辑
          const k = path.replace("/api/files/", "");
          const obj = await env.BUCKET.get(k);
          const txt = await obj.text();
          return resJSON({content: txt});
      }

      if (path === "/api/backup") {
          const { results } = await env.DB.prepare("SELECT * FROM posts").all();
          return resJSON(results);
      }
      
      if (path === "/api/import" && request.method === "POST") {
          const list = await request.json();
          const stmt = env.DB.prepare("INSERT INTO posts (title, slug, content, category, created_at) VALUES (?, ?, ?, ?, ?)");
          const batch = list.map(p => stmt.bind(p.title, p.slug, p.content, p.category, p.created_at));
          await env.DB.batch(batch);
          return resJSON({imported: list.length});
      }

      return resJSON({ error: "Not found" }, 404);
    }

    // E. 公开前台路由 (渲染 HTML)
    
    // 获取布局模板
    const layoutObj = await env.BUCKET.get("layout.html");
    const layoutRaw = await layoutObj.text();
    const siteName = (await env.DB.prepare("SELECT value FROM settings WHERE key='site_name'").first())?.value || "My Blog";

    // 1. 首页
    if (path === "/") {
      const { results } = await env.DB.prepare("SELECT * FROM posts ORDER BY created_at DESC").all();
      const listHtml = results.map(p => `
        <div class="post-card">
          <h2 class="post-title"><a href="/post/${p.slug}" style="text-decoration:none;color:#333">${p.title}</a></h2>
          <div class="meta">${p.category || '未分类'} - ${p.created_at}</div>
          <p>${p.content.substring(0, 100)}...</p>
        </div>
      `).join("");
      
      const homeRaw = await (await env.BUCKET.get("home.html")).text();
      const content = render(homeRaw, { posts_list: listHtml });
      const fullHtml = render(layoutRaw, { title: siteName, site_name: siteName, content: content });
      return resHTML(fullHtml);
    }

    // 2. 文章详情页
    if (path.startsWith("/post/")) {
      const slug = path.split("/post/")[1];
      const post = await env.DB.prepare("SELECT * FROM posts WHERE slug=?").bind(slug).first();
      
      if (!post) return new Response("Post not found", { status: 404 });

      const postRaw = await (await env.BUCKET.get("post.html")).text();
      const content = render(postRaw, { 
          post_title: post.title, 
          post_content: post.content, // 注意：这里默认没有转换Markdown，需要在后台直接存HTML或在前台引入md解析库
          category: post.category,
          date: post.created_at
      });
      const fullHtml = render(layoutRaw, { title: post.title + " - " + siteName, site_name: siteName, content: content });
      return resHTML(fullHtml);
    }

    return new Response("Not found", { status: 404 });
  },
};
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
