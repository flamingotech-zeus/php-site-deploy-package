---
name: php-site-deploy-package
description: 把「纯静态前端 + 原生 PHP 接口」的站点打包成可直接上传宝塔的部署包，并在上传前用本机真实 PHP 引擎把这份包原样跑一遍做上线前验证；站点上线后，用增量发布器把本地改动安全同步到线上。当需要交付上线（建子域名、加 DNS、宝塔建站、申请证书、传文件）、日常更新已上线站点、或排查「本地好的、线上不行」时使用。
agent_created: true
---

# PHP 站点部署包构建与上线前验证

核心原则：**验证的对象必须是"要上传的那份包本身"，而不是源码目录。**
把部署包原样当网站根目录、用真实 PHP 引擎跑一遍接口测试 —— 这是唯一能在上传前抓住
「配置没填 / 文件漏传 / 权限不对 / 上层把老副本覆盖了」这类事故的手段。

## 一、打包：先侦察，再构建

### 1. 公网侦察（全部只读，不碰用户服务器）

| 要查什么 | 怎么查（本机沙箱可直接用） |
|---|---|
| 域名解析商 = 该去哪加解析 | DoH 查 NS：`https://dns.alidns.com/resolve?name=<域名>&type=NS`（看到 `*.dnspod.net` → DNSPod，`*.hichina.com` → 阿里云） |
| 服务器 IP | 同上查 `www.<域名>` 与根域名的 A 记录 |
| 端口开放情况 | Node `net.connect` 逐端口探测：80 / 443 / 22（SSH）/ 8888（宝塔默认）；据此判断能否 SSH 代办 |
| 新子域名是否已存在 | 查它自己的 A 记录，NXDOMAIN = 待新增 |
| 主站是否在线 | `curl -o /dev/null -w "%{http_code}"` |

> **坑**：沙箱内直连 DNS（UDP 53）会 `ECONNREFUSED`，必须走 **DoH over HTTPS**；
> PowerShell 在这类环境里输出常被吞，**用 Node 打印结果更可靠**。

### 2. 构建脚本（写成可重跑的脚本，别手工复制文件）

关键点：
- 只把 **`.php` 白名单文件**与前端三件套、静态资源目录放进包；**显式排除运行时残留**
  （`auth.data.php`、`*.config.json` 写入产物、`*.log`、`*.example.*`）。
- 配置文件在**打包时生成**，不要复制开发机那份：
  - `salt` 用 `crypto.randomBytes(16).toString('hex')` 现生成
  - 共享密码入口写 `sha256(密码)`，密码本身写进单独的密码提示文件
  - 模型/三方 Key 从本机已验证的配置里读出来填进去（线上才算真正可用）
- 产物三件套：`站点文件/`（可直接拖拽上传）、`<项目>-部署包.zip`（**内容在 zip 根，解压即用**）、
  `nginx配置片段.conf`、`部署操作单.md/.docx`。
- 摘要目录里放一份 `_本次生成的密码.txt`，并明确标注「勿外发、用完可删」。

### 3. 打 zip 与核对（Windows 环境）

- `Compress-Archive -Path "站点文件\*"`（`*` 保证内容在根，不套一层目录）。
- **不要用 `Add-Type`**（在本机被安全策略拦）；核对 zip 结构用 `tar -tf <zip>`（Windows 自带 bsdtar）。
- 必须核对：根目录有 `index.html`、`api/_auth.php`、`api/*.config.php`、静态资源目录完整。

## 二、上线前验证：把包当网站根目录跑真实 PHP

1. 把 `站点文件/` 递归复制到临时目录（**手写递归复制**，`fs.cpSync` 在本机沙箱会被强杀）。
2. `php.exe -S 127.0.0.1:<端口> -t <临时目录>` 起真实 PHP 内置服务器。
3. 断言清单（照抄即可，覆盖上线会踩的所有点）：

| 组 | 断言 |
|---|---|
| 静态 | 首页 200 且体积正常；前端依赖的内核声明/关键按钮存在 |
| 接口存活 | `health.php` 结构正确、**Key 是掩码不是明文**；`domain-check.php` 可返回 |
| 登录门禁 | 未登录 `auth-check` 返回 `authed:false`；各通道登录成功/失败；会话 Cookie 生效 |
| 权限分层 | 未登录调消耗额度接口必须 401；已登录放行到业务层 |
| 密钥防下载 | 直接访问 `auth.config.php` / `llm.config.php` / `_auth.php` **拿不到明文或源码** |
| 完整性 | 前端引用的每个接口文件都存在；包内无运行时残留 |

4. 跑前清 `auth.data.php` 等状态文件，保证可重复执行。
5. **断言失败先怀疑断言**：先打印真实返回体再改断言（实战中曾因此误判 2 条）。

## 三、宝塔上线步骤（给用户的精确参数）

顺序不可颠倒：**先加解析 → 建站 → 传文件 → 最后申请证书**（证书要域名已解析成功）。

1. **DNS**：加 `A` 记录，主机记录 `one`，值 = 服务器 IP，TTL 600。
2. **建站**：宝塔 → 网站 → 添加站点，域名 `<子域>`，PHP 选 8.0/8.1，不建 FTP/数据库。
3. **上传**：zip 传到站点根目录 → 右键解压 → 核对目录结构（`api/_auth.php` 最易漏）。
4. **权限**：目录 755、文件 644、属主 `www`；**`api/` 必须可写** —— 登录会话写在
   `api/auth.data.php`（运行时生成），目录不可写会导致「登录后立刻掉线」。
5. **SSL**：Let's Encrypt 一键 + 开启强制 HTTPS（登录涉及手机号与 Cookie，必须 HTTPS）。
6. **Nginx / PHP 参数**（AI 流式输出必须改，否则「卡 60 秒 504」或「等很久一次性蹦出」）：
   - `fastcgi_buffering off;` + `fastcgi_read_timeout 300s;`
   - `fastcgi_pass unix:/tmp/php-cgi-<版本>.sock;` 的版本号要按实际装的 PHP 改
   - PHP：`max_execution_time=300`、禁用函数里不能有 `curl_exec` / `curl_init`

## 四、给用户的验证清单（照抄进操作单）

打开 `https://<子域>`：① 出现登录页；② 点「评委一键进入」能进；③ 密码通道能进；
④ 未登录访问 `api/chat.php` 返回 401；⑤ 直接访问 `api/auth.config.php` 是空白页；
⑥ AI 功能文字逐字流出；⑦ 免密钥查询类功能有结果；⑧ 内容板块正常。

## 五、交付话术要点

- 明确区分「**我已做**」与「**必须你做**」：代码/打包/验证我做；DNS、建站、证书、短信实名认证
  这类需要账号登录态的，只能用户做或用户授权（给宝塔账号 / 开 SSH 22 端口）。
- 服务器上跑着其它站点时，**主动说明"新增站点不影响现有站点"**，并优先建议用户自己点、我出精确参数。
- 涉密文件（含 Key 的 zip、密码提示文件）单独提醒不要外发。

## 六、AI 全自动代办（2026-09-16 实战验证，SSH 不通也能全做完）

前提：拿到面板账号密码，或浏览器 profile 里已有登录态（Edge `--remote-debugging-port=9223
--user-data-dir=<profile> --ignore-certificate-errors` 起，**以后台任务运行**，脚本 sleep 保活）。

| 环节 | 已验证做法 |
|---|---|
| 面板 API 通用执行 | puppeteer 连 9223，页面内嗅探 `x-http-token` 注入 `window.__api(url, opts)`；宝塔 11.6 裸 fetch 一律「权限认证失败」 |
| 建站 | 走 UI 最稳：`/site/php` 页点「添加站点」弹窗（el-dialog），原生 setter + input/change 事件赋值后点「确定」，接口即 `/site?action=AddSite`；列表在 `POST /datalist/data/get_data_list`（table=sites） |
| DNS（DNSPod 免密钥） | 控制台登录态页面内 `POST /cgi/capi?cmd=CreateRecord&serviceType=dnspod`，body `{"serviceType":"dnspod","cmd":"CreateRecord","data":{Version:"2021-03-23",Domain,SubDomain,RecordType:"A",RecordLine:"默认",Value,TTL:600}}`，再 DescribeRecordList 回读 |
| 上传文件 | Node fetch 直连 `/files?action=upload`（multipart FormData + Blob），需 `NODE_TLS_REJECT_UNAUTHORIZED=0`；2.92MB 约 1 秒。会话（cookie+token）先用 puppeteer `page.cookies()` 导出 |
| 跑 shell 命令 | 面板文件接口写脚本 → **SaveFileBody 前必须先 CreateFile**（否则「指定文件不存在」）→ 追加一行 cron 到 `/var/spool/cron/root` → 每分钟执行 → 读回结果文件 → **删脚本 + 还原 cron** |
| 证书 | acme.sh webroot 签发（先在 80 端口可访问时签，**再**改 vhost 做 301 强跳，顺序反了验证会失败） |
| 解压 | `files?action=UnZip`（sfile/dfile/type:zip），宝塔会自动把属主落成 www |

本机调试坑：curl/Node 必须绕系统代理（`--noproxy "*"` / NO_PROXY），否则 502/解析失败；
PowerShell 输出中文乱码是 GBK 显示问题（内容本身 UTF-8 正常），落盘再 Read 最稳。

## 七、上线之后：日常更新走增量发布，不要在服务器上直接改

**原则：本地是唯一源码真相，服务器只当接收方。** 服务器直改会让本地与线上分叉——
下次整包发布时改动被静默覆盖，且没有语法检查、没有回滚点。

架构：`源码目录（前端 + api/） → 发布器 → 服务器站点根目录`

### 六道工序（按顺序，任何一步失败即中止）

| # | 工序 | 关键实现 |
|---|---|---|
| 1 | 收集源码 | 前端目录全量 + `api/*.php`；**排除** `*.example.php` 与含密钥的配置 |
| 2 | 上传前体检 | 对每个 `.php` 跑 `php -l`；同时扫「注释里含 `?>`」（PHP 会在此处提前结束文件）。源码有错=阻断，打包副本有错=告警 |
| 3 | 比对线上 | 见下方「读盘比对」 |
| 4 | 备份 | 覆盖前把线上旧文件按相同相对路径存到 `rollback/<时间戳>/` |
| 5 | 上传 | 面板 `/files?action=upload`（multipart），会话过期自动续期一次后重试 |
| 6 | 回读验证 | 再从服务器读一遍做 sha256 比对 + 首页 / health / 鉴权三项 HTTP 冒烟 |

### 读盘比对是这套方案的核心（踩过的坑）

**绝不能用 HTTP GET 去比对 `api/*.php`** —— 这些接口有鉴权，未登录返回 401、
方法不对返回 405，拿到的是错误页而不是文件内容，会把**全部 PHP 文件误判成"已变更"**。

正确做法：用面板 API 直接读服务器磁盘原文：
`GET /files?action=GetFileBody&path=<绝对路径>` → JSON 的 `.data` 字段即文件内容。

- 文本文件（html/js/css/php/svg/txt/json/xml/conf/md）→ 走面板读盘
- 二进制文件（图片）→ 走 HTTP GET 取原始字节

### 两个必踩的归一化坑

1. **二进制绝不能做文本归一化**。为了消除 BOM 与 CRLF 差异，通常会对内容做
   `decode utf-8 → 去 BOM → CRLF→LF → encode utf-8`。这个流程套到 JPEG 上，
   非法字节会被替换成 U+FFFD，**文件体积从 203KB 涨到 354KB**，于是"11 张图全部被认为已变更"。
   → 按扩展名分流：文本才归一化，二进制直接用原始字节比对。
2. 线上文件常带 BOM 或 CRLF，本地是 LF / 无 BOM。**两边都要归一化后再算 sha256**，
   否则内容完全相同也会被判定为有差异。

### 密钥配置的处理

`auth.config.php`（短信/登录密钥）、`llm.config.php`（大模型 Key）**硬性排除在发布范围外**，
只在服务器上维护。发布器仍会读它们做漂移提示与语法体检，但不写入。
改动配置必须单独授权、单独处理。

### 静态资源缓存版本号

只要 `app.js` 或 `styles.css` 发生变化，就自动把 `index.html` 里的 `?v=` 刷成新值
（`styles.css?v=20260916d` → `?v=202609162030`）。正则要同时兼容
`href="styles.css"`（无参数）和 `href="styles.css?v=旧值"`（有参数）。
**这是"改完发布成功、用户却还看到旧版"的头号原因**，必须自动化，不能靠人记得。

### 应急回收（线上直改后的补救）

线上着火不得已在服务器上改了文件时，立刻把改动捞回本地保持单一真相：
`publish --pull <相对路径>` → 先把本地旧版备份到 `rollback/<时间戳>_pull/`，
再把线上内容写回本地源码，并报告两边是否本来就一致。

### 工具形态

- 发布器：`tools/publish.js`（Node，无第三方依赖，只用内置 `fetch`/`FormData`/`Blob`）
- 双击入口：`publish.cmd`（先跑会话刷新脚本，再跑发布器；`chcp 65001` 保中文输出）
- 开关：`--dry` 只预演比对、`--all` 全量重传、`--pull <路径>` 回收
- 记录：每次发布写 `发布日志/<时间戳>.json`（动了哪些文件 / 校验 / 冒烟结果）
- 发布成功后把已发布文件同步回打包副本目录，保持"打包副本 == 线上"

### Windows 下跑 Node 脚本的两个环境坑

- 需要 `puppeteer-core` 的脚本必须设 `NODE_PATH=<node 工作区>/node_modules`，
  否则 `Cannot find module 'puppeteer-core'`。
- PowerShell 捕获 Node 的中文输出会乱码，且 `2>&1 | Out-String` 后写文件仍可能坏。
  稳妥写法：先 `[Console]::OutputEncoding=[Text.Encoding]::UTF8`，
  再用 `[IO.File]::WriteAllText(路径, $r, [Text.Encoding]::UTF8)` 落盘，最后 Read 出来看。
- 本会话还遇到 Bash 工具 PATH 损坏（`dirname`/`ls`/`head` not found），改用 PowerShell 完成全部操作。


