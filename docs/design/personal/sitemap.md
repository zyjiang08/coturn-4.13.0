# 站点地图

第一期推荐：**单页锚点首页** + **2～3 个子页**。路由可用纯静态文件。

## 页面

| 路径 | 名称 | 对标模板 | 内容来源 |
|------|------|----------|----------|
| `/` 或 `index.html` | 首页 | Default.aspx | Hero→联系完整主流程 |
| `/about.html` | 关于 | Aboutus.aspx | 长简介 + 时间线 |
| `/work/index.html` | 作品列表 | Photo.aspx | 全部作品卡 |
| `/work/cloud-phone.html` | 云手机详情 | NewsDetail / 作品详情 | CONTENT §9（含 `#latency`、下载占位） |
| `/articles.html` | 文章 | News.aspx | 外链卡片为主 |
| `/contact.html` | 联系 | Contact.aspx | 电话邮箱 + mailto |

可选合并：第一期不设 `/articles.html`，首页 `#articles` 即可。

## 首页锚点

| 锚点 | 区块 |
|------|------|
| `#top` | 顶栏 / Hero |
| `#about` | 关于摘要 + 四宫格 |
| `#stats` | 数字条 |
| `#work` | 作品网格 |
| `#articles` | 精选文章 |
| `#contact` | 联系 CTA |

## 导航点击行为

| 菜单 | 行为 |
|------|------|
| 首页 | `/#top` |
| 关于 | `/about.html`（或 `/#about`） |
| 作品 | `/#work` 或 `/work/` |
| 文章 | `/#articles` |
| 联系 | `/#contact` |
| 在线体验 | 外链新窗口 `nexartc.com:8444/device/` |

## 外链（不进站内路由）

- RTNlite 公众号文  
- RTNlite vs RTSA 飞书文  
- 云手机在线体验 `nexartc.com:8444/device/`  
- 二进制下载 `{BINARY_*_URL}` — **当前留空**，有链后再进「获取」区  
- （可选）抖音创作者云编辑体验页 — 仅关于页「历史项目」引用，不作主 CTA  
- （可选）CN105025351B Google Patents  

## 云手机详情页锚点

| 锚点 | 区块 |
|------|------|
| `#problem` | 问题 |
| `#solution` | 方案 |
| `#results` | 结果与体验 |
| `#download` | 获取（体验 + 二进制占位） |
| `#architecture` | 架构图 |
| `#latency` | 延迟测试说明 |
| `#stack` | 技术栈 |

## 目录建议（实现阶段）

```text
personal/site/          # 或仓库根 personal-website/
  index.html
  about.html
  contact.html
  articles.html
  work/
    index.html
    cloud-phone.html
  assets/
    img/
    fonts/
  css/
  js/
```

设计阶段不创建上述站点工程，仅约定地图以便后续实现。
