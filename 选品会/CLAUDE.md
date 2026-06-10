# 有点东西选品会 · 项目说明 + 制作手册

> 这是「有点东西」烤肉品牌的**新品选品会**可视化看板项目。
> 用户（业务侧）会给一份**提案数据**（通常是 .docx，含文字 + 小红书热度截图），
> 需要把它做成一份**可视化 HTML 长页**，部署到 Cloudflare Pages 供选品会投屏 / 分享。
> 已做到第三期（Vol.03）。
>
> ⚠️ **做任何一期前，必须先读营销管理体系：**
> `GAGA应用/营销管理体系/`
> - `00-定向与决策/` — 当前方向、决策记录、工作分块
> - `SOP/02-产品线/` — 选品款型框架（流量/核心/复用/互补）+ 查检表评分细则
> - 尤其是 `选品-查检表.md`，四款型的评分体系决定了每期提案的分类依据
> - 营销体系的`项目定向.md`和`术语表.md`也建议先读

## 线上地址与部署机制（重要）

- **生产网址（固定，分享给参会人员）**：https://youdiandongxi-8rs.pages.dev/
- **部署方式**：Cloudflare Pages「Wrangler 直传」（`wrangler pages deploy`），**不走 GitHub 自动部署**
- **两条链路是分开的**：
  - CF Pages（`youdiandongxi-8rs.pages.dev`）= 真正给人看的网站，靠 wrangler 直传
  - GitHub（`github.com/ywh9527-prog/gaga`）= 代码备份
  - ⚠️ **光 push GitHub 不会更新线上网站**，改完内容必须重新跑 `wrangler pages deploy`

## 文件结构

```
选品会/
├── index.html                          # 在线看板（始终最新一期，部署入口）
├── 选品会提案_20260528_图文版.html      # 第一期归档
├── 选品会提案_20260604_图文版.html      # 第二期归档（后续期数类推）
├── assets/                             # 热度截图（p*.jpg 第一期，v*.jpg 第二期…）
├── CLAUDE.md                           # 本文件
└── .gitignore                          # 排除 .remember/ 和 .DS_Store
```

- `index.html` = 始终放**最新一期**，部署入口，每次新一期就替换它
- 每期归档 = `选品会提案_YYYYMMDD_图文版.html`，跟 index.html 同级
- 日期规则：**每个周四**（YYYYMMDD），新一期比上一期推 **一周**
- `.remember/` 是私密会话记忆，**绝不能进 git、绝不能传到公网**，已被 .gitignore 排除。

---

# 当用户给你新一期数据时，照这 6 步做

## 步骤 1 · 读数据（.docx → 文字 + 图片映射）

环境里 **pandoc 和 defusedxml 都没有**，docx 本质是 zip，手动解压最稳：

```bash
# 解压 docx（路径换成实际文件）
rm -rf /tmp/xph && mkdir -p /tmp/xph && unzip -q "选品会.docx" -d /tmp/xph
```

提取「每段文字 + 该段引用的图片 rId」：

```bash
cd /tmp/xph && python3 -c "
import re
xml = open('word/document.xml', encoding='utf-8').read()
for p in re.findall(r'<w:p[ >].*?</w:p>', xml, re.S):
    line = ''.join(re.findall(r'<w:t[^>]*>(.*?)</w:t>', p, re.S))
    imgs = re.findall(r'r:embed=\"(rId\d+)\"', p)
    if imgs: line += ' [IMG:' + ','.join(imgs) + ']'
    if line.strip(): print(line)
"
```

rId → 实际图片文件 的映射在 `word/_rels/document.xml.rels`：

```bash
grep -o 'Id="rId[0-9]*"[^>]*Target="media/[^"]*"' word/_rels/document.xml.rels
```

→ 由此确定「第 N 个提案 → 用哪几张 image*.png」。

## 步骤 2 · 跟用户确认结构（架构层禁止替用户拍板）

把提取出的**提案清单、产品角色矩阵**用**用户原话**回读一遍，让用户确认分类/命名/合并，再动手。
（见全局 CLAUDE.md「用户口头定义架构 → 必须原文回读」铁律。）

同时确认两件影响搭法的事：
- 呈现形式：滚动长页 / 翻页幻灯片（第一期选了**滚动长页 + 左侧导航锚点**）
- 图片呈现：缩略图点击放大 / 平铺 / 轮播（第一期选了**缩略图 + 点击放大 lightbox**）

## 步骤 3 · 压缩图片（手机竖图 55MB → 1.8MB）

原图是 1290×2796 手机截图，每张 2-4MB。用 macOS 原生 `sips` 缩到 860 宽转 jpg，
**热度数据文字仍清晰**，体积砍到约 1/30：

```bash
mkdir -p assets
for i in $(seq 1 17); do
  sips -Z 860 /tmp/xph/word/media/image$i.png --out assets/p$i.jpg \
       -s format jpeg -s formatOptions 80 >/dev/null 2>&1
done
```

文件名按提案顺序排：p1-3=提案1，p4-6=提案2…（跟步骤1的 rId 映射对应）。

## 步骤 4 · 套用设计模板做 index.html

**直接复用现有 `index.html` 作模板**，只替换内容（提案数据 + 图片路径 + 数字）。
设计语言**对齐用户参考的核销看板**（`~/Documents/Codex/2026-05-14/.../cf-pages-dashboard/index.html`），
关键审美规则（已踩过坑，别改坏）：

- **配色**：暖米底 `#f5efe6` + 炭火橙 `#d86b2a` + 窑青 `#2d7b74` + 暖墨字 `#1f1a17`；四宫格四色（橙/黄铜/橄榄/青）
- **毛玻璃面板**：`backdrop-filter: blur(16px)` + 半透明白；背景用橙/青径向光晕 + 暖渐变
- **字体**：标题 `'Iowan Old Style','Songti SC',serif`（离线可用，Win 降级宋体）；正文 `'Avenir Next','PingFang SC'`；大数字 `Georgia` 衬线
- **大圆角** 28-32px + 暖色阴影 + 卡片角落径向光晕装饰
- **不要用 emoji**（用户明确要求）；序号用衬线数字代替图标
- **层级**：小标题（如"研发落地逻辑"）必须明显大于它下面的分点
- **分点不要做成胶囊标签**——用「左侧细色条 + 加粗橙色引导词」。胶囊只留给真·分类标签（流量款/核心款等）
- **整齐**：所有区块左对齐到同一条竖线（别加零散的 4px 偏移）；画廊缩略图用**固定宽度** `repeat(auto-fill, 150px)` 不要 `1fr` 拉伸（否则 3 图/4 图的提案缩略图大小不一）
- **每期版本号**：首屏徽标「第N期 · VOL.0N」+ 左侧导航品牌行「有点东西 · 第N期」，两处都要改

本地预览：`open index.html`。UI 探索阶段直接在 index.html 改、用户看浏览器，不必每改一版走 git 流程。

## 步骤 5 · 部署到 Cloudflare Pages

CF token 在 `~/.env.cloudflare-token`（含 `CLOUDFLARE_API_TOKEN` + `CLOUDFLARE_ACCOUNT_ID`，多行 env 格式，**要 `source` 不能 `cat`**）。

```bash
cd /Users/owen/ai项目/gaga/选品会
set -a; source ~/.env.cloudflare-token; set +a

# 建干净部署目录，只放网页+图片，排除 .remember（私密！）/.DS_Store
rm -rf /tmp/xph-deploy && mkdir -p /tmp/xph-deploy/assets
cp index.html /tmp/xph-deploy/ && cp assets/*.jpg /tmp/xph-deploy/assets/

# 部署（项目已存在，直接 deploy；首次需先 wrangler pages project create youdiandongxi --production-branch=main）
wrangler pages deploy /tmp/xph-deploy --project-name=youdiandongxi --branch=main --commit-dirty=true

# 验证
curl -s -o /dev/null -w "%{http_code}\n" https://youdiandongxi-8rs.pages.dev/
```

⚠️ 部署务必从「干净目录」传，**不能直接传 `选品会/` 目录**（会把 `.remember/` 私密记忆推上公网）。

## 步骤 6 · Git（main 是保护分支，必须走功能分支）

`选品会/` 在 `gaga` 仓库内（不是独立仓库）。`main` 有 PreToolUse hook **禁止直接 commit**。

```bash
cd /Users/owen/ai项目/gaga
git checkout -b feat/xuanpin-deck-volN     # 切功能分支
git add 选品会/index.html 选品会/assets 选品会/.gitignore
git commit -m "..."                         # 在功能分支提交
git checkout main && git merge --ff-only feat/xuanpin-deck-volN   # 快进合并（不产生新commit，hook放行）
git push origin main
git branch -d feat/xuanpin-deck-volN
```

注意：跨多个 Bash 调用时 git 暂存可能丢，`add` + `commit` 尽量放**同一条命令**里。

---

## 第一期内容快照（供下期参考结构）

框架：产品研发四宫格矩阵（流量款引流 / 核心款利润 / 降本款复用 / 互补款防御）。
5 个提案，每个 = 定位标签 + 平台热度数据条 + 研发落地逻辑分点 + 小红书截图画廊：
1. DIY迷你烤汉堡（流量+降本，5000+赞）
2. 雪糕芝士肥牛卷（极致流量）
3. 「绿野仙踪」系列肉卷（核心利润，山姆同款4700+）
4. 水果烤肉季（话题核武器，榴莲8500+）
5. 青苔包肉（猎奇流量，杭州烤肉next level）

---

## 浏览器操控能力（agent-browse）

本机运行着 agent-browse 中继服务器，可以通过真实 Chrome 浏览器访问网页。不走 Playwright 的假浏览器，反爬检测低，登录态可复用。

**架构**：AI → MCP/HTTP → agent-browse 中继（localhost:18801）→ WebSocket → Chrome 扩展 → 你的真实 Chrome

**前置条件**（已配好）：
- `~/agent_browse/server/` 的中继服务器须在运行（`node dist/index.js`，端口 18800/18801）
- Chrome 须打开并安装了 `~/agent_browse/extension/` 的扩展（已加载的话会自动重连）

**MCP 工具**（通过 opencode MCP 已启用，连 `http://127.0.0.1:18801/mcp`）：

| 工具 | 功能 |
|------|------|
| `tabs_list` | 列出所有标签页 |
| `tab_attach` | 附加调试器到某标签页 |
| `navigate` | 导航到 URL |
| `click_selector`、`click_text` | 点击元素 |
| `type` | 输入文字 |
| `screenshot` | 截图 |
| `snapshot` | 获取页面无障碍树（结构化内容） |
| `evaluate` | 执行 JavaScript |
| `cookies_get`、`cookies_set` | Cookie 操作 |

**典型用法**（搜小红书竞品笔记）：
```
navigate("https://www.xiaohongshu.com/search_result/...")
wait_for → screenshot/snapshot → 看页面
click_text("笔记标题") → 点进去
snapshot → 提取正文、点赞评论数
```

**⚠️ 安全红线**：
- 当前走的是本地自建中继，所有流量不出本机，安全
- 绝对不要改配置连公网中继（browse.clembot.uk）——运营者可看到你的所有浏览内容
- 涉及登录/短信验证码/滑块验证，必须叫用户手动操作

**如果服务器没在运行**（重启后），启动命令：
```bash
cd ~/agent_browse/server
AGENT_BROWSE_PORT=18800 AGENT_BROWSE_MCP_PORT=18801 \
AGENT_BROWSE_HOST=127.0.0.1 AGENT_BROWSE_MCP_HOST=127.0.0.1 \
nohup node dist/index.js > ~/agent-browse-server.log 2>&1 &
```
