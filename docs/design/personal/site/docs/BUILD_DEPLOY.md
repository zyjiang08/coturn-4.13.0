# 个人站编译与部署说明

独立 Vite 静态站，与云手机 `/device/`、CMS `/cms/` **分开编译**，部署到同一 VPS Hub 的 `WEB_ROOT` 根路径。

| 项 | 值 |
|----|-----|
| 源码 | `personal/site/`（仓库根下） |
| 设计文档 | `coturn-4.13.0/docs/design/personal/` |
| 线上入口 | https://www.signalling-nexartc.cn/ |
| IP 备用 | https://120.79.21.28/ |
| VPS 静态根 | `/opt/nexartc/hub`（Hub `WEB_ROOT`） |
| 部署脚本 | `personal/site/scripts/deploy.sh` |

云手机体验页 `/device/` 仍由 `nexartc-cloudPhoneAccess-web` + `test/turn/deploy_vps.sh` 部署，**不要**用本脚本覆盖。

---

## 1. 环境要求

- Node.js 18+（当前开发机可用 20/24）
- npm
- 本机可 SSH：`ssh -i ~/.ssh/id_ed25519 root@signalling-nexartc.cn`
- `rsync`

---

## 2. 目录结构（源码）

```text
personal/site/
├── index.html              # 首页
├── about.html              # 关于 / 经历 / 论文专利
├── contact.html            # 联系
├── work/cloud-phone.html   # 云手机详情
├── src/
│   ├── styles.css
│   └── main.js
├── public/img/             # 构建时复制到 dist/img/
├── scripts/deploy.sh       # 一键编译部署
├── vite.config.js
├── package.json
└── dist/                   # npm run build 产物（勿手改）
```

---

## 3. 本地开发

```bash
cd personal/site
npm install
npm run dev
```

浏览器打开 Vite 提示的本地地址（默认 `http://localhost:5173/`）。

---

## 4. 编译

在仓库根或 `personal/site` 下：

```bash
cd personal/site
npm install          # 首次或依赖变更时
npm run build        # 输出到 personal/site/dist/
```

产物要点：

| dist 路径 | 部署到 VPS |
|-----------|------------|
| `index.html` / `about.html` / `contact.html` | `/opt/nexartc/hub/` |
| `work/` | `/opt/nexartc/hub/work/` |
| `assets/`（带 hash 的 js/css） | `/opt/nexartc/hub/assets/` |
| `img/` | `/opt/nexartc/hub/img/` |

---

## 5. 部署到 VPS（推荐）

一键（含 `npm install` + `build` + rsync + 本机 HTTPS 抽检）：

```bash
# 在仓库根执行亦可
./personal/site/scripts/deploy.sh
```

仅同步已有 `dist/`（改完并本地 build 后）：

```bash
./personal/site/scripts/deploy.sh --skip-build
```

指定主机：

```bash
VPS_HOST=root@signalling-nexartc.cn ./personal/site/scripts/deploy.sh
```

### 脚本会同步的路径

- `index.html`、`about.html`、`contact.html`
- `work/`（`--delete`，与 dist 对齐）
- `assets/`、`img/`

### 脚本明确不碰

- `device/`、`cms/`、`server/`、`admin.html`、`data/`

因此可与 `deploy_vps.sh` 并存：Hub / device 用 `test/turn/deploy_vps.sh`，个人站用本脚本。

---

## 6. 手动部署（可选）

```bash
cd personal/site
npm run build

SSH="ssh -i ~/.ssh/id_ed25519 -o BatchMode=yes"
RSYNC_RSH="ssh -i ~/.ssh/id_ed25519 -o BatchMode=yes"
HOST=root@signalling-nexartc.cn
ROOT=/opt/nexartc/hub

rsync -az -e "$RSYNC_RSH" dist/index.html dist/about.html dist/contact.html "$HOST:$ROOT/"
rsync -az --delete -e "$RSYNC_RSH" dist/work/ "$HOST:$ROOT/work/"
rsync -az -e "$RSYNC_RSH" dist/assets/ "$HOST:$ROOT/assets/"
rsync -az -e "$RSYNC_RSH" dist/img/ "$HOST:$ROOT/img/"
```

Hub 进程一般**无需重启**：静态文件按请求读盘；若浏览器仍见旧页，强刷（Ctrl+Shift+R）。

---

## 7. 验收

部署脚本会在 VPS 上对 `https://127.0.0.1` 检查：

| 路径 | 期望 |
|------|------|
| `/` | 200 |
| `/about.html` | 200 |
| `/contact.html` | 200 |
| `/work/cloud-phone.html` | 200 |
| `/device/` | 200（确认未误伤云手机页） |

本机抽检：

```bash
curl -sk -o /dev/null -w '%{http_code}\n' https://120.79.21.28/
curl -sk https://120.79.21.28/ | head -c 200
```

说明：部分网络对 `www.signalling-nexartc.cn` 存在备案/SNI 拦截时，可用 IP `120.79.21.28` 验收；VPS 本机访问域名应为 200。

---

## 8. 页面与路由

| URL | 说明 |
|-----|------|
| `/` | 首页 |
| `/about.html` | 经历、论文与专利 |
| `/work/cloud-phone.html` | 云手机详情；二进制下载按钮 URL 留空（disabled） |
| `/contact.html` | 联系 |
| `/device/` | 云手机 Web 体验（非本站构建） |
| `/cms/` | 运营后台（非本站构建） |

「在线体验」按钮指向同域 `/device/`。

---

## 9. 常见问题

| 现象 | 处理 |
|------|------|
| 根路径仍 404 | 确认已 rsync `dist/index.html` 到 `/opt/nexartc/hub/`；Hub `WEB_ROOT=/opt/nexartc/hub` |
| 样式丢失 | 确认 `assets/` 已同步，且 HTML 中 script/link 的 hash 文件名与 dist 一致 |
| `/device/` 被覆盖 | 勿对整个 `WEB_ROOT` 做 `rsync --delete`；只用 `deploy.sh` |
| 域名打不开、IP 可以 | 按既有 Hub 文档排查 ICP/SNI；功能以 IP 或 VPS 本机域名为准 |
| 改文案后未更新 | 先 `npm run build` 再 deploy；浏览器强刷 |

---

## 10. 与设计包的关系

| 内容变更 | 改哪里 |
|----------|--------|
| 文案 / 延迟说明 / 专利列表 | 先改 `docs/design/personal/CONTENT.md` 等，再改 `personal/site/*.html` |
| 视觉 token | `DESIGN.md` + `personal/site/src/styles.css` |
| 二进制下载 URL | `CONTENT.md` §9.5 → `work/cloud-phone.html` 启用按钮 |

设计包索引：[`../../README.md`](../../README.md)
