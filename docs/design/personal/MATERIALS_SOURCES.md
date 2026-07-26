# 材料来源说明（便于 Review）

本目录设计包与线上个人站文案/素材的出处对照。Review 时优先核「主源 → 落地文件」是否一致；标注「待补」的项不得当作已核验事实。

整理日：2026-07-26

---

## 1. 主源一览

| 类别 | 主源 | 路径 / 链接 | 用于 |
|------|------|-------------|------|
| 简历 | 本人提供 PDF | [江中央简历.pdf](./江中央简历.pdf) | 姓名、联系、经历时间线、技能、作品一句话、延迟数字口径、论文/专利数量 |
| 版式参考 | 腾云个人站模板 JP-11304 | 预览 https://template.tyjz.com/14525/ ；上架 http://www.400301.com/dzsy/14525 | 信息架构节奏（非文案/非摄影素材） |
| 人物照片 | 本人补充 | [reference/20260726-141605.jpg](./reference/20260726-141605.jpg) | 关于/联系氛围图；站点裁切见 `personal/site/public/img/portrait-card.jpg` |
| 云手机产品 | 本仓库 nexartc 设计与线上体验 | 见 §3 | 作品页方案叙述、体验 CTA |
| RTNlite | 简历内链 + 公开文 | 见 §4 | 作品/文章外链 |
| 论文 | 本人提供题名/作者/期刊/日期 + 公开卷期页核对 | 见 §5、[PUBLICATIONS.md](./PUBLICATIONS.md) | 关于页论文列表（4 篇） |
| 专利 | 本人提供公开号/申请号/日期/题名/申请人 | 见 §5、[PUBLICATIONS.md](./PUBLICATIONS.md) | 关于页专利列表（5 项齐全） |

**不进站的主源内容**：期望薪资；公司内部未公开架构/指标细节。

---

## 2. 简历 → 站内映射

主源：`江中央简历.pdf`

| 简历区块 | 落地文档 | 线上页面 |
|----------|----------|----------|
| 姓名 / 电话 / 邮箱 | [CONTENT.md](./CONTENT.md) §0、§10 | 页脚、联系、侧栏（若有） |
| 个人作品 1 云手机 + 延迟 50/80ms | CONTENT §2/§6/§9/§13 | `/`、`/work/cloud-phone.html` |
| 个人作品 2 RTNlite + 两外链 | CONTENT §6/§7 | `/` 作品与文章 |
| 工作经历（倒序） | CONTENT §8 | `/about.html` |
| 教育（中南硕士 / 南华本科） | CONTENT §4 | `/` 关于摘要、`/about.html` |
| 「论文 2 篇 / 专利 5 项」数量 | CONTENT §5；题名见 PUBLICATIONS | 数字条、关于页 |
| 期望薪资 | **不上站** | — |

联系方式以简历为准：电话 `13632650465`，邮箱 `2935616@qq.com`。

---

## 3. 云手机 / nexartc（仓库与线上）

| 材料 | 来源 | 落地 |
|------|------|------|
| 体验入口 | 简历：`https://www.nexartc.com:8444/device/`；同 VPS 亦有 `/device/` | 站内 CTA 默认同域 `/device/`（见 BUILD_DEPLOY） |
| 平台愿景叙述 | [../nexartc-low-latency-platform-vision.md](../nexartc-low-latency-platform-vision.md) | CONTENT 文章摘要、作品页「愿景」段 |
| 架构/TURN/Hub 对外表述 | `coturn-4.13.0/docs/design/nexartc-*.md`（如 Mode A、部署指南） | CONTENT §9 方案要点（脱敏，不贴内部密钥/未支持特性） |
| 延迟定量方法 | 简历数字 + 设计阶段整理 | CONTENT §13；作品页 `#latency` |
| 二进制下载 | **故意留空** | CONTENT §9.5；`work/cloud-phone.html` disabled 按钮 |

实现源码（非本目录）：仓库根 `personal/site/`。

---

## 4. RTNlite 与外链文章

均来自简历「个人作品」所列链接：

| 标题 | URL | 用途 |
|------|-----|------|
| RTNlite 轻量级实时通信引擎 | https://mp.weixin.qq.com/s/0TgTdN1VFrOUmm9CHJ1Luw | 作品行、文章列表 |
| RTNlite vs 声网 RTSA 内存/包大小对比 | https://tknugg6xx5.feishu.cn/docx/G9bXduFOiodO3XxUSRQcXgT2nBb | 文章列表 |

站内不镜像正文，只做外链。

---

## 5. 论文与专利（可核验出处）

详细著录与检索过程见 [PUBLICATIONS.md](./PUBLICATIONS.md)。

### 5.1 论文（单位口径：中南大学）

本人 2026-07-26 提供 4 篇；规范著录见 [PUBLICATIONS.md](./PUBLICATIONS.md) §1。

| 题名 | 期刊 / 日期 | 来源 |
|------|-------------|------|
| 求解全局优化问题的混合自适应正交遗传算法 | 软件学报，2010-06-15 | 本人 + 卷期页公开核对 |
| 一种新的基于正交实验设计的约束优化进化算法 | 计算机学报，2010-05-15 | 本人（含罗一丹） |
| 用于全局优化的混合正交遗传算法 | 计算机工程，2009-02-20 | 本人 |
| 一种改进的模糊免疫反馈PID控制器 | 控制工程，2008-09-20 | 本人 |

「合计引用 200+」：站内直接表述，不另加来源括号。

### 5.2 专利（单位口径：TCL）

本人 2026-07-26 提供 5 项公开号台账；规范著录见 [PUBLICATIONS.md](./PUBLICATIONS.md) §2。

| 公开号 | 题名 | 申请人 |
|--------|------|--------|
| CN106604115A | 视频播放控制装置及方法 | 深圳TCL新技术有限公司 |
| CN106454635A | 多声道无线音箱之间数据同步的方法及系统 | 深圳TCL数字技术有限公司 |
| CN106331084A | 软件后台自适应升级方法及装置 | 深圳TCL新技术有限公司 |
| CN105472457A | 基于视频启动播放方法及视频启动装置 | 深圳TCL数字技术有限公司 |
| CN105025351A | 流媒体播放器缓冲的方法及装置 | 深圳TCL新技术有限公司 |

---

## 6. 版式与视觉参考

| 材料 | 来源 | 使用边界 |
|------|------|----------|
| 栏目顺序 / 轮播·四宫格·数字条节奏 | 腾云 JP-11304 | **只参考 IA**；见 [reference/TEMPLATE_NOTES.md](./reference/TEMPLATE_NOTES.md) |
| 色板 / 字体 / 文案 | 本包 [DESIGN.md](./DESIGN.md)、[CONTENT.md](./CONTENT.md) | 独立设计，不用模板默认风光图与 Logo |
| 人物照 | `reference/20260726-141605.jpg`（本人提供） | 用法见 [reference/PHOTO_NOTES.md](./reference/PHOTO_NOTES.md)、ASSETS A08 |

---

## 7. 本目录文件 ↔ 材料角色

| 文件 | 角色 | 材料从哪来 |
|------|------|------------|
| `江中央简历.pdf` | 主源 PDF | 本人提供 |
| `CONTENT.md` | 站内文案定稿骨架 | 简历 + nexartc 设计文 + PUBLICATIONS |
| `DESIGN.md` | 视觉与区块规格 | 模板 IA + 自定 token |
| `ASSETS.md` | 素材清单与缺口 | 汇总上述 + 待补项 |
| `PUBLICATIONS.md` | 论文专利检索纪要 | 公开库检索 + 简历数量声明 |
| `MATERIALS_SOURCES.md` | **本文件：Review 用出处表** | — |
| `sitemap.md` | 路由地图 | 设计推导 |
| `reference/*` | 模板摘记、原图、相片说明 | 模板站点 / 本人照片 |
| `site/docs/BUILD_DEPLOY.md` | 编译部署 | 实现与 VPS 现状 |

---

## 8. Review 检查清单（建议）

- [ ] 联系方式、公司时间线与简历 PDF 一致  
- [ ] 延迟数字仍为简历口径，且作品页有「个人环境 / 非全网承诺」说明  
- [ ] 论文 4 篇著录与 PUBLICATIONS（GB/T 7714）一致；数字条「核心 2 篇」口径可读  
- [ ] 专利 5 项公开号与本人台账一致；链接可点开 Google Patents  
- [ ] 外链文章 URL 可打开  
- [ ] 无期望薪资、无编造二进制下载链  
- [ ] 模板仅影响结构，无腾云默认图/文案残留  

有出入时：**以简历 PDF 与公开专利/论文著录为准**，再回改 CONTENT 与 `personal/site` 页面。
