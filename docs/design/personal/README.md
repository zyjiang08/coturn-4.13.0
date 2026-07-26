# 江中央个人网站 — 设计包

对照腾云个人站模板 **JP-11304**（预览：<https://template.tyjz.com/14525/>，上架页：<http://www.400301.com/dzsy/14525>），为个人作品集站点准备的设计与文案包。

设计文档 + 实现侧编译部署说明见下表；源码在仓库根 `personal/site/`。

**材料出处（Review 用）→ [MATERIALS_SOURCES.md](./MATERIALS_SOURCES.md)**  
主源：简历 PDF、腾云模板 IA、本人照片、本仓库 nexartc 设计文、公开论文/专利库；详见该文档映射表与检查清单。

## 目录

| 文件 | 说明 |
|------|------|
| [MATERIALS_SOURCES.md](./MATERIALS_SOURCES.md) | **材料来源与 Review 对照表** |
| [DESIGN.md](./DESIGN.md) | 视觉系统、信息架构、各区块线框与交互 |
| [CONTENT.md](./CONTENT.md) | 中文定稿文案骨架（含延迟定量 §13、下载占位 §9.5） |
| [ASSETS.md](./ASSETS.md) | 已有 / 缺失素材、规格与优先级 |
| [PUBLICATIONS.md](./PUBLICATIONS.md) | 论文（中南）与专利（TCL）检索纪要 |
| [sitemap.md](./sitemap.md) | 页面与锚点地图 |
| [site/docs/BUILD_DEPLOY.md](./site/docs/BUILD_DEPLOY.md) | **编译与 VPS 部署说明** |
| [site/docs/README.md](./site/docs/README.md) | 实现侧文档索引 |
| [reference/TEMPLATE_NOTES.md](./reference/TEMPLATE_NOTES.md) | 参考模板结构、色板、动效摘记 |
| [reference/20260726-141605.jpg](./reference/20260726-141605.jpg) | 人物场景照（关于区） |
| [江中央简历.pdf](./江中央简历.pdf) | 文案与经历源材料（主源） |

## 定位一句话

**nexartc / 江中央：把边缘真机做成浏览器可开的低延迟双向交互；云手机是首个规模化个人作品。**

## 阶段

1. **设计**：本目录文档（论文/专利/延迟/下载占位）。  
2. **实现（已上线）**：`personal/site/` Vite 多页静态站 → VPS Hub 根路径。  
   - 线上：https://www.signalling-nexartc.cn/  
   - 编译部署说明：[site/docs/BUILD_DEPLOY.md](./site/docs/BUILD_DEPLOY.md)  
   - 脚本：`personal/site/scripts/deploy.sh`  
   - 与 `/device/`、`/cms/` 共存，单独编译部署。

## 你可继续补的三项

1. `{BINARY_CAE_URL}` / `{BINARY_CAS_URL}` → 启用下载按钮  
2. 云手机 Hero 真实截图（M02）  
3. （可选）专利授权公告号 B 号若需与公开号 A 号一并展示

## 相关仓库文档

- [`../nexartc-low-latency-platform-vision.md`](../nexartc-low-latency-platform-vision.md) — 产品升维叙事  
- 体验入口：<https://www.nexartc.com:8444/device/>
