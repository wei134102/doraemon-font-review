# ANARCHY REIGNS（BLES01232）汉化技术总结与交接 — 纯文本版

> 本文件由 `ps3/report.html` 抽取正文生成，供评审 agent 直接抓取（约 40KB）。
> 省略了 4 段 base64 文件内容与 3 张内嵌截图；完整版（含 base64）见同目录 index.html / report.html。
>
> 抓取地址（三选一，内容相同）：
> - https://wei134102.github.io/doraemon-font-review/ps3/report.md
> - https://raw.githubusercontent.com/wei134102/doraemon-font-review/main/ps3/report.md
> - https://cdn.jsdelivr.net/gh/wei134102/doraemon-font-review@main/ps3/report.md

---


# ANARCHY REIGNS（BLES01232，欧版）简体中文汉化 — 技术总结与交接

生成时间：2026-11-20 晚 ｜ 平台：RPCS3 0.0.43 ｜ 交付目标：接手汉化的下一个 agent

工作区 `H:\ps3\game\` ｜ 实验日志（权威）`H:\ps3\game\1_doc\ISSUES_SUMMARY.md`

**30 秒状态（2026-11-20 21:00）**

当前部署 **v2.25**（sha1 `ff98fc3e`）。用户实测：✔ 任务内对话中文字幕恢复（面积假说实锤）、
✔ SELECT/OPTIONS/START 正常、✘ 雷达旁地图名 / 任务名 / 顶部播报不显示。

**结论：对话渲染约束 = 字库图集面积 ≥ 1M 像素；HUD 地名渲染约束 = 图集宽 = 256 整（512、1024 都灭）。**

**下一实验（最高优先）：v2.27 = messc dtt 改为 256×4096**（宽 256 ✔ 地名 + 面积 1M ✔ 对话，从未测过）。
构建管线已就绪：`2_work/cpk/build_v226_sharp_messc.py` 只改目标尺寸即可。

**目录**：
1 工程基础 ｜ 2 文件格式 ｜ 3 实验全史 ｜ 4 尺寸约束模型（核心） ｜
5 对话消失悖论 ｜ 6 已确证根因 ｜ 7 急需解决的问题 ｜
8 v2.27 构建方案 ｜ 9 截图证据 ｜ 10 工具脚本 ｜ 11 陷阱清单 ｜ 12 版本回退链 ｜ 13 附录：messc112 文件与脚本源码

## 1. 工程基础

| 项 | 值 |
| --- | --- |
| 游戏 | ANARCHY REIGNS 欧版 BLES01232，RPCS3 运行，EBOOT.BIN（30.8MB RETAIL 加密）从未修改 |
| **真正生效的数据** | `rpcs3_ANARCHY REIGNS chs\dev_hdd0\game\BLES01232ANARCHYREIGNS\USRDIR\data000.cpk`（改这里才有效） |
| 无效路径（别改） | `dev_hdd0\disc\ANARCHY REIGNS_eur\...\data000.cpk`（原版，运行时不读） |
| 其他数据 | data001.cpk（8.6GB 光盘）、data002.cpk（1.79GB 音频）；存档 `dev_hdd0\home\00000001\savedata\BLES01232-ANARCHYREIGNS\` |
| 容器格式 | CRI CPK v7，TOC=@UTF @0x810，8297 条目，ContentOffset 0x58800；**汉化方式 = 追加新文件到 CPK 尾部 + 改 TOC 行**（cpk_repack.py） |
| 日志 | `log/RPCS3.log`、`log/TTY.log`（TTY 有 PPU abort 裸栈）；日志 .gz 只是当前会话滚动备份 |
| RPCS3 排除项 | 版本（旧版也不好使）、缓存（清过）、渲染配置三项（已与原版实例对齐）——**环境层全部排除，问题在数据/引擎层** |

## 2. 文件格式知识（已逆向验证）

### 2.1 dat/mcd（文本）

```
dat 头（96B）→ 块表（off/tag/size 三表，size 表指针在头 +0x14，注意先取指针值再寻址！）
mcd 块：组表 7×8B（g0=sec1/g1=cmap/g2=rects/g3=metrics/g4=sec5/g5=映射表/g6=sec7）
       → 池（0x38~g0.off，u16 数组）→ sec1（每消息 16B）→ 消息块（子行 24B：
       text_off/char_start/chars/codes/mark0/mark1）
cmap 行 8B = (style u16, codepoint u16, glyph_index u32)；rects 行 40B =
       (hash, u0,v0,u1,v1, w,h, a,b,c)——UV 归一化到图集宽高
total_chars（消息块头 +12）必须等于子行 chars 之和（v2.6 教训，v2.7 修复）
```

### 2.2 dtt（字库图集）

```
dat 容器 → WTB 块（头 0x1000，wcount@+4=纹理页数）→ GTF（头 0x80：+0x20/+0x22 = 宽/高 u16，
+0x10 像素数据偏移，+0x14 数据长度）→ 像素 = DXT5（每 4×4 块 16B）
texture hash 字段 = 字体/样式 ID，不随尺寸/内容变化（v2.25 与 v0.9 同 hash 互换成功）
```

### 2.3 铁律

**dat 与 dtt 是配套对**：rects UV 对应 dtt 图集布局，任何一方单独替换都破坏渲染链（v1.8 实锤）。
替换必须成对、从同一携带版本取。美术字 token 0x4000+ 集中在 ui_* 文件（messc 无真实 40xx 引用，重渲染无需保留原版区域）。

## 3. 实验全史（v2.x 主线）

| 版本 | 改动 | 实测结果 |
| --- | --- | --- |
| v2.6 | 全量原位重建 | 卡关（BOSS 不刷兵）+ total_chars 未修 |
| v2.7 | 修 total_chars（49 文件/9599 条） | 仍卡关 |
| v2.10 | 内容 100% 原版 + 40 个 UI 条目走同一套 repack | ✔ 能过 → **打包方式无罪（P1）** |
| v2.11 | 菜单 10 模块中文 | ✔ 菜单中文 |
| v2.13 | + 9 个 HUD 模块中文（排除 ui_hud_common） | ✔ 能过 |
| **v2.14** | + 全部 mess 任务文本 & 对话中文（messc 1024×1024） | ✔ 全中文可玩；✘ 地名/任务名空白（1024 宽） |
| v2.15/17/18 | messc 重建 256×2048 / 加粗 / 地名恢复英文 | 对话灭（面积 0.5M）；v2.15 256 宽地名英文名可见 |
| v2.19/20/21 | messc 256×2048 / 256×1024（装不下）/ 512×1024 | 对话全灭（面积 <1M） |
| v2.22~24 | messc112 多页 256×1024×2 → 英文回退 → wcount 2→1 | v2.23 英文地名显示；对话全程不出现（见 §5 悖论） |
| **v2.25** | **v2.14b 基座 + v0.9 全部 37 对 messc（35 张 dtt=512×2048，面积 1M）** | ✔✔ 对话中文恢复（512×2048）；✘ 地名/任务名仍灭（512 宽也灭！） |
| v2.26 | 35 张图集 0.90 字号重渲染（墨迹 22→26px），文本零改动 | 备包未部署（被 v2.27 尺寸方案取代，管线复用） |
| **v2.27（下一步）** | **messc dtt → 256×4096**（宽 256 ✔ 地名 + 面积 1M ✔ 对话） | 待构建实测 |

v0.x/v1.x 旧线（CPK 追加 repack 探索、camp_mission/campaign 瘦身、SELECT/OPTIONS 卡死修复）见 `1_doc/` 各取证文档与 README 第十四章——其结论已被 v2.x 修正（如"地名疑 EBOOT 硬编码"是错的，地名就在 messc112_us.dat M7-15）。

## 4. 尺寸约束模型（核心结论）

| 渲染用途 | 约束（当前可信版） | 全部实测数据点 |
| --- | --- | --- |
| 关卡文本 / 任务内对话<br>（messc*.dtt） | **图集面积 ≥ 1M px**<br>（等价表述：DXT5 数据 ≥ 1MB） | 1024×1024=1M ✔ v2.14；512×2048=1M ✔ **v2.25**；256×2048=0.5M ✘ v2.15/19；512×1024=0.5M ✘ v2.21；256×1024 ✘ v2.22-24 |
| HUD 地名 / 任务名 / 播报<br>（同文件 messc*.dtt！文本在 messcXXX_us.dat M1-15/M16-292） | **图集宽 = 256 整** | 256 ✔ v2.15/v2.19/v2.23（英文名显示）；512 ✘ **v2.25**（且 v0.9 时代亦灭）；1024 ✘ v2.14/v2.18 |

**死结与出路**：两约束作用在**同一个文件** messc*.dtt 上。交集 = **256×4096**（=1M px，DXT5 数据量与 512×2048 相同 1MB）。
字形容量：~570 字形 @ 31px 格 → 256 宽 7-8 个/行 × ~124 行 ≈ 900+ ✔。
PS3 RSX / RPCS3 支持 4096 纹理（需实测确认引擎接受）。**v2.27 即此实验。**

### 约束认知的修正历程（避免重走弯路）

- "高≤1024"（v2.x 早期）——被 v0.9/v2.25（512×2048 高 2048 对话正常）推翻

- "面积≥1M"（对话侧）——**v2.25 实锤保留**

- "宽≤512"（地名侧，本次会话提出）——被 v2.25 实测推翻（512 地名仍灭）

- **"宽=256 整"**（地名侧）——当前最可信；若 v2.27 对话又灭，则对话约束改判为组合条件（如"宽≥512 且面积≥1M"），转字段级二分（ISSUES_SUMMARY §9 方向 A/B）

## 5. 对话消失悖论（v2.22~v2.24，已被 v2.25 部分解释）

**悖论**：ev112（对话文件）在 v2.22/23/24 全程字节未动，TOC 全 8297 条目只有 messc112_us.dat/.dtt 两条不同，
但对话不显示；v2.23 已证明引擎能正常采样页 1（英文地名显示）、v2.24 排除 wcount。
**v2.25 的解释**：对话渲染由 messc dtt 的**几何属性**（面积/宽）决定——512×2048（面积 1M）对话复活，
256×1024（0.26M）对话灭。即对话系统对"当前任务文本字库图集"的容量有硬校验，与文本内容/页数无关。

## 6. 已确证根因（P1~P8）

| # | 结论 | 证据 |
| --- | --- | --- |
| P1 | **追加式 repack 无罪**（ContentSize/CRC 陈旧不影响运行） | v2.10 全原版内容走同一套 repack 实测能过 |
| P2 | total_chars 是真 bug 但非卡关主因 | v2.7 修复后仍卡关 |
| P3 | **ui_hud_common 汉化必卡 BOSS 关**（逻辑钉死） | v2.9（含）卡 vs v2.13（排除）过，唯一差异即该文件；其 dtt 第 2 张 512×256→1024×1024 8 倍像素疑似触发。v0.9 版（512×256+512×2048）在 v0.x 线无卡关记录 |
| P4 | 尺寸双约束（见 §4） | v2.14~v2.25 全部数据点 |
| P5 | 两种"对话"是两套系统：任务内实时对话=mess/ev/*_us.dat+dtt；过场字幕=event/*.mvd 内嵌 MCD；任务简报=ui_camp_mission | 勿混淆素材 |
| P6 | 地名/任务名文本源 = **messcXXX_us.dat**（M1-6 任务标签、M7-15 地图名、M16-292 警告/播报） | 推翻 v1.x "疑 EBOOT 硬编码"（当时扫错文件集合） |
| P7 | 中文"发虚" = 字号比 0.78/0.71 太小 + DXT5 抹平细笔画 | v0.9 messc 墨迹实测 0.71×31px；v2.26 重渲染 0.90 → 墨迹 26px，alpha 目检清晰 |
| P8 | ui_hud_common 修复路线 = 高频短标签局部汉化（~54 字形空位，不改图集尺寸） | 未实施 |

## 7. 急需解决的问题清单（按优先级）

| 优先 | 问题 | 状态 / 方案 |
| --- | --- | --- |
| **P0** | **地名/任务名/播报显示**（雷达旁地名框空白、任务名空白、顶部滚动播报无） | **v2.27 = messc dtt 256×4096**（见 §8）。文本已全在 dat 里（中文，v0.9 版），只差字库几何满足渲染约束 |
| P1 | ev310 过场室内段字幕 52 条英文（轨 2 漏翻） | 译文已备 `2_work/cpk/work/ev310_track2_source.json`。v1.3 的 512 宽 dtt 方案已删除作废；**新思路：用面积假说——把 ev311_us.dtt 扩为 ≥1M px（如 512×2048 或 256×4096）再注入**，避开 v1.2 的"Mes+mcd 必须连续"雷区 |
| P2 | ui_hud_common 汉化即卡 BOSS 关 | P8 方案：仅汉化 ~20 个高频短标签塞进原 512×256 空位（约 54 字形槽），不改图集尺寸；风险中等需实测 |
| P3 | 中文发虚（任务文本/主菜单） | v2.26 管线（0.90 重渲染）并入 v2.27 一次完成 |
| P4 | BIG BX 美术字错位（应为 BIG BULL） | 三套美术字编码已破解（0x40d3/0x41b6/0x4166），独立问题 |
| P5 | ui_gallery 92 条英文 | 未处理 |
| P6 | messc101/102/122 等无 "Main/Free Mission" 锚点导致部分地名仍中文 | find_targets 锚点逻辑问题，未处理 |

## 8. v2.27 构建方案（下一步，含代码要点）

```
目标：35 张 messc*_us.dtt 512×2048 → 256×4096（面积不变 1M px，宽变 256）
基座：v2.25（patch/v2.25_messc_v09/data000.cpk，或当前部署文件）
管线：直接复用 2_work/cpk/build_v226_sharp_messc.py，改三处：
  1) 目标尺寸 (W,H)=(256,4096)，行宽断言 x + im.width > W-2 换行
  2) 行高仍 cell_h+2；~570 字形 @ 7-8 个/行 → ~82 行 × 33px ≈ 2706px < 4096 ✔
  3) 同时启用 SIZE_RATIO=0.90 重渲染（= 并入 v2.26 的锐化，一举两得）
要点：
  - dat 仅原位重写 rects 段 UV（大小不变）；dtt 仅替换像素区（GTF 头 +0x20/+0x22 改尺寸，
    数据长度不变仍为 1MB DXT5；wcount 保持 1）
  - rects 的 w/h 字段保持 31/31（布局推进不变）；UV 按 256/4096 归一化
  - messc011(256×1024)/messc113(128×512) 两张原版小图跳过
  - 顺带把字号超宽字形（墨盒 >31px）单独缩号（管线已实现）
验证：35 对 repack verify；空字形 tile=0；alpha 抽查（必须看 alpha，RGB 白底全是假象）
部署：copy 到 USRDIR\data000.cpk，sha1 记录，用户实测 4 项：
  ①对话中文 ②雷达旁地名+任务名 ③顶部播报 ④BOSS 关不卡
回退：patch/v2.25_messc_v09/data000.cpk（v2.25）或 USRDIR\data000.cpk.v214b（v2.14）
```

**若 v2.27 对话又灭**：说明对话约束不是纯面积（可能是"宽≥512 且面积≥1M"的组合），此时：
按 ISSUES_SUMMARY §9 方向 A/B 做字段级二分（从 v2.23 出发把 messc112.dat 的 sec1/metrics/cmap 逐段对齐回 v2.14，
单变量定位触发对话渲染失败的字段）；或转向 EBOOT 逆向（scetool 解密 RETAIL → ps3recomp 工具链，路径见 README 第十四章/记忆）。

## 9. 截图证据

**对话 ✓ 中文** — v2.25 实测①：任务开场对话中文字幕正常显示（BIG BULL 开场，512×2048 图集）——面积假说（≥1M px）实锤

**地名 ✗ 空白** — v2.25 实测②：任务内 HUD——雷达旁地名框空白（应为 "The Underground"），任务名同样不显示。512 宽图集也灭 → "宽≤512"证伪，地名约束收敛为宽=256 整

**图集 alpha ✓** — v2.26 重渲染后的 messc112 图集 alpha 通道（黑=墨迹）：英文行+中文行字形清晰完整、无碎片无重叠——重渲染管线已验证可用，v2.27 直接复用改目标尺寸

## 10. 工具与脚本清单

| 路径 | 作用 |
| --- | --- |
| `C:/Users/86159/.gemini/config/skills/ps3-localization/scripts/` | 核心库：cpk_tool.py（TOC/读条目）、cpk_repack.py（追加式 repack）、dtt.py（GTF/WTB 解析）、mcd.py（Mcd 类）、dat.py、mcd_build.py（重建）、text_encode.py（token 编解码）、crilayla.py（解压） |
| `H:/ps3/game/2_work/cpk/atlas_build.py` | **新版字库渲染管线**（build_multi 支持 size_ratio/pad_x；style_metrics/style_color/render_glyph/load_font/dxt_enc） |
| `2_work/cpk/build_v225_messc_v09.py` | v2.25 构建（v2.14b 基座 + v0.9 的 37 对 messc） |
| `2_work/cpk/build_v226_sharp_messc.py` | **v2.26/27 渲染管线（0.90 重渲染 + 原位 UV 重写 + 像素区替换）——v2.27 的底子** |
| `2_work/cpk/check_dims_v225.py` | 两个 CPK 的 messc 尺寸/结构对比工具 |
| `H:/ps3/03_work/` | 另一 agent 的 v2.x 全套脚本（build_v214~v224、analyze_hud_font、check_messc_* 等）+ 图集 PNG |
| `2_work/cpk/work/` | 翻译源 trans_all/*.json、src/*.dat、dtt/*.dtt、art_messages.json（美术字扫描）、ev310_track2_source.json |
| `1_doc/` | ISSUES_SUMMARY.md（权威实验日志）、ASSET_TOOL_MAP.md、各取证 md、PS3汉化方案.pdf（ChatGPT 多页方案，多页路线已被 v2.22-24 证伪） |

**import 大坑**：`cpk_tool` 所在的 gemini 目录里有一份**旧版 atlas_build.py**（无 pad_x/size_ratio），
且 import cpk_tool 会把该目录插到 sys.path[0] → **必须先 import atlas_build 再 import cpk_tool**，并 assert
`'pad_x' in ab.render_glyph.__code__.co_varnames`（build_v226 头部有现成写法）。

## 11. 陷阱清单（血泪教训，逐条都是实测踩出来的）

- **dat+dtt 必须成对替换**，从同一版本取（v1.8 单换 dtt → HUD 渲染链崩溃）

- 查字形形状**必须看 alpha 通道**——RGB 白字形+白预览底 = 全白假象（多次误判教训）

- 图集重排后必须做**渲染抽查**（切 rects 看字形完整性+alpha），只查 UV 越界不够（v1.5 碎片化教训）

- dat 块表寻址：size 表**指针**在头 +0x14，先读指针值再 +4*i 寻址（本次会话踩过两次）

- cursor 循环复用未重置 → msgblock 写错位；plan_words 零占位未替换 → decode 全 {0000}；VERIFY-DEBUG 必须 assert 前插

- encode 后**splice 原 DXT 块**（重编码 ≠ 原块字节，v1.6 用 DXT 块级拷贝实现 0 损失恢复原版区域）

- 目录 mtime 误导：覆盖文件不刷新目录 mtime——部署验证以 **sha1 + 文件 mtime** 为准（用户常问"修改日期对吗"）

- 实验包从干净 patch 基线构建，勿用安装目录做 src（v1.4 污染教训）；过滤回退清单同时排除同名 .dat/.dtt

- MVD 多轨：Mes+mcd 必须**连续**（引擎线性读，v1.2 单字段重定位教训）；+0x2c/+0x30 双字段一起改

- 明文三连扫（UTF-16BE/LE/ASCII）先于 glyph 流扫描；"The Underground" 在 CPK 明文 0 命中是因为它在 messc112 的 glyph 索引里而非明文

- 用户实测偏好：单变量实验包 + 快速反馈 + 部署后清 cache 再测；"v0.x 显示"类记忆需与逐字节验证交叉

## 12. 版本回退链（全部有 sha1）

| 版本 | 位置 | sha1 | 说明 |
| --- | --- | --- | --- |
| **v2.25（当前部署）** | USRDIR\data000.cpk = patch/v2.25_messc_v09/ | `ff98fc3e80f30d6574dd6288298d0663c32942ab` | 对话✔中文 / 地名✘ / 不卡关 |
| v2.26（备包） | patch/v2.26_sharp_messc/ | `fd262a4c185b94d6c860c4607d347d44c9765897` | 锐化重渲染，未部署 |
| v2.24 | patch/v2.24_wcount1/ | `b0f74789574cbb66fb9fbac65f2943fb1f743ea3` | 英文名显示/对话灭 |
| v2.14b | USRDIR\data000.cpk.v214b（另有 v214/v214c） | `66ef2b16c8567a2b9c0c0f6e3a3a75fc1bca635b` | v2.x 最优可玩基线 |
| 原版 | USRDIR\data000.cpk.orig + 2_work/backup/ | `9bee56b86e86dd785ad8bf6f2d1540145df4b614` | 纯英文 |

完整 v2.6~v2.23 的 sha1 见 ISSUES_SUMMARY §5。

## 13. 附录：messc112 附件文件与构建脚本源码

### 13.1 已提取的独立文件（共 4 版本，含本文档 base64 内嵌的 2 版）

目录 `H:\ps3\game\1_doc\attach_messc112\`（extract_messc112.py 生成）：

| 文件 | 大小 | dtt 尺寸 | 说明 |
| --- | --- | --- | --- |
| messc112_us.orig.dat/.dtt | 48,336 / 77,824 | **256×256** | 原版英文（70 字形；地名显示正常） |
| messc112_us.v214.dat/.dtt | 72,040 / 1,060,864 | 1024×1024 | v2.14：对话 ✓ 地名 ✗（1024 宽） |
| messc112_us.v221.dat/.dtt | 81,594 / 536,576 | 512×1024 | v2.21 实验：对话 ✗（面积 0.5M） |
| **messc112_us.v225.dat/.dtt** | 75,592 / 1,060,864 | **512×2048** | **当前部署（v0.9 内容，中文）**：对话 ✓ 地名 ✗（512 宽） |

四版 texture hash 全部相同（721304383）= 字体 ID 不随尺寸/内容变。原版图集仅 256×256（英文 70 字形够用）。

### 13.2 文件 base64（v225 当前部署版 + v221 实验版，复制块内容存为 .txt 后解码）

```
# 解码方法（PowerShell / python）：
python -c "import base64,sys; base64.decode(open('块内容.txt','rb'), open('输出文件名','wb'))"
# 或：certutil -decode 输入.txt 输出文件  （注意 certutil 要求 PEM 头尾，直接 base64 用 python 上面这行更稳）
```

**messc112_us.v225.dat**（base64，98 KB）：复制下面整块到 `messc112_us.v225.dat.b64.txt` 后按上面命令解码

```
[base64 blob omitted: 100792 chars. Full content: ps3/report.html]
```

**messc112_us.v225.dtt**（base64，1381 KB）：复制下面整块到 `messc112_us.v225.dtt.b64.txt` 后按上面命令解码

```
[base64 blob omitted: 1414488 chars. Full content: ps3/report.html]
```

**messc112_us.v221.dat**（base64，106 KB）：复制下面整块到 `messc112_us.v221.dat.b64.txt` 后按上面命令解码

```
[base64 blob omitted: 108792 chars. Full content: ps3/report.html]
```

**messc112_us.v221.dtt**（base64，698 KB）：复制下面整块到 `messc112_us.v221.dtt.b64.txt` 后按上面命令解码

```
[base64 blob omitted: 715436 chars. Full content: ps3/report.html]
```

### 13.3 构建脚本源码（全文内嵌）

**build_v221_messc_512x1024.py** — v2.21 构建脚本（对方 agent 指定；v2.x 线的 messc 重建管线：build_multi 追加字形 + mcd_build.rebuild + 地名/任务标签保持英文的 find_targets 锚点法。注意：文件名 v2.21 但 docstring/OUT_DIR 仍是 v2.19 的痕迹（该线复用脚本改常量），H_MAX=1024/W_MAX=512 产出 ≤512×1024 图集）

```
# -*- coding: utf-8 -*-
"""v2.19: 只重建 messc*（任务文本）字库，**宽度锁定原版 256**、高度可增至 2048。
依据：v2.15 实测 messc112 为 256x2048 时小地图地名可显示；v2.14/2.18 为 1024 宽时不可显示。
同时把「任务标签 + 地名块」保持英文（HUD 只认英文字形）。
mess/ev（对话）与 messm 不动，保证字幕维持 v2.14 的可显示状态。
基线 v2.14。
"""
import os, sys, re, struct, json
sys.path.insert(0, r'C:\Users\86159\.workbuddy\skills\ps3-localization\scripts')
sys.path.insert(0, r'H:\ps3\game\2_work\cpk')
import mcd, mcd_build, atlas_build, text_encode as te, cpk_tool, cpk_repack, dtt as dttmod

BASE = r'H:\ps3\game\2_work\patch\v2.14_mess_cn\data000.cpk'
ORIG = r'H:\ps3\game\rpcs3_ANARCHY REIGNS chs\dev_hdd0\game\BLES01232ANARCHYREIGNS\USRDIR\data000.cpk.orig'
SRC_DIR = r'H:\ps3\game\2_work\cpk\work\src'
DTT_DIR = r'H:\ps3\game\2_work\cpk\work\dtt'
TRANS_DIR = r'H:\ps3\game\2_work\cpk\work\trans_all'
OUT_DIR = r'H:\ps3\game\2_work\patch\v2.19_messc_w256'
os.makedirs(OUT_DIR, exist_ok=True)
DST = os.path.join(OUT_DIR, 'data000.cpk')

SIZE_RATIO = 0.85
H_MAX = 1024
W_MAX = 512

MISSION_RE = re.compile(r'^(Main|Free|Side)\s*Mission\b', re.I)
PUNCT = re.compile(r'[.!?。！？,，:：;；]')

def is_place(t):
    if not (0 < len(t) <= 26): return False
    if PUNCT.search(t) or re.search(r'\d', t) or '\n' in t: return False
    if len(''.join(c for c in t if c.isalpha())) < 3: return False
    return t[0].isupper()

def find_targets(texts):
    tgt = set()
    for i, t in enumerate(texts):
        if MISSION_RE.match(t):
            tgt.add(i); j = i + 1
            while j < len(texts) and is_place(texts[j]):
                tgt.add(j); j += 1
    return tgt

def msg_texts(m):
    out = []
    for mi in range(m.msg_count):
        try:
            _, _, subs, _ = m.message_subs(mi)
            out.append(' '.join(m.decode_tokens(m.scan_tokens(s['text_off'], s['codes']))
                                for s in subs).strip())
        except Exception:
            out.append('')
    return out

overrides = {}; report = []
for fn in sorted(os.listdir(TRANS_DIR)):
    if not fn.endswith('.json') or not fn.startswith('mess__messc'):
        continue
    stem = fn[:-5]
    base_name = stem.replace('mess__', '')
    dat_p = os.path.join(SRC_DIR, stem + '.dat')
    dtt_p = os.path.join(DTT_DIR, stem + '.dtt')
    if not (os.path.exists(dat_p) and os.path.exists(dtt_p)):
        continue
    dat = open(dat_p, 'rb').read(); dtt = open(dtt_p, 'rb').read()
    trans = json.load(open(os.path.join(TRANS_DIR, fn), encoding='utf-8'))
    try:
        m = mcd.Mcd(dat)
    except Exception:
        continue
    orig_texts = msg_texts(m)
    targets = find_targets(orig_texts)

    t0 = dttmod.parse(dtt, 0)
    W0 = t0['textures'][0]['width']
    dchunk = 0

    styles = [te.msg_style(m, mi) for mi in range(m.msg_count)]
    cp2gi_by = {st: te.cmap_map(m, st) for st in set(styles) if st is not None}

    trans2 = {k: v for k, v in trans.items() if int(k) not in targets}

    new = {}
    for mi_s, lines in trans2.items():
        mi = int(mi_s)
        if mi >= m.msg_count: continue
        st = styles[mi]
        if st is None: continue
        d = cp2gi_by[st]; bucket = new.setdefault(st, [])
        for line in lines:
            for ch in te.CTL_RE.sub('', line):
                if ch in ' \n': continue
                cp = ord(ch)
                if cp not in d and cp not in bucket:
                    bucket.append(cp)

    try:
        at = atlas_build.build_multi(dtt, m, new, font_name='noto_bold',
                                     max_dim=H_MAX, max_w=min(W_MAX, max(W0, 512)), dtt_chunk=dchunk,
                                     size_ratio=SIZE_RATIO, baseline_ratio=0.86)
    except Exception as e:
        print('  FAIL', base_name, str(e)[:50]); continue

    for st, cp, gi in at['glyphs']:
        cp2gi_by[st][cp] = gi

    sp = te.space_params(m)
    new_subs = []
    for mi in range(m.msg_count):
        subs = m.message_subs(mi)[2]
        if mi in targets:
            new_subs.append([m.scan_tokens(s['text_off'], s['codes']) for s in subs])
        elif str(mi) in trans2:
            st = styles[mi]; cp2gi = cp2gi_by[st]; spv = sp.get(st, sp[None])
            new_subs.append([te.encode_line(l, cp2gi, spv) for l in trans2[str(mi)]])
        else:
            new_subs.append([m.scan_tokens(s['text_off'], s['codes']) for s in subs])

    out_dat = mcd_build.rebuild(dat, new_subs, at['cmap'], at['rects'])
    m2 = mcd.Mcd(out_dat)
    bad = 0
    for mi in range(m2.msg_count):
        _, _, subs, total = m2.message_subs(mi)
        if sum(sum(1 for t in m2.scan_tokens(s['text_off'], s['codes'])
                   if t['kind'] != 'nl' and not (t['kind'] == 'ctl' and t['value'] == 0x8005))
               for s in subs) != total:
            bad += 1
    report.append((base_name, at['W'], at['H'], len(targets), len(trans2), bad))
    overrides[('mess', base_name + '.dat')] = out_dat
    overrides[('mess', base_name + '.dtt')] = at['dtt']

print('重建 messc 文件对:', len(report) // 1)
for r in report[:12]:
    print('   %-14s atlas=%dx%d labels=%d trans=%d bad=%d' % r)

print('\nRepacking from v2.14...')
replaced = cpk_repack.repack(BASE, DST, overrides, verbose=False)
print('Repacked %d, size=%d' % (len(replaced), os.path.getsize(DST)))

_, rows = cpk_tool.load_toc(DST)
byd = {(r['DirName'], r['FileName']): r for r in rows}
bad = sum(1 for k, v in overrides.items() if cpk_tool.read_entry(DST, byd[k]) != v)
print('override mismatch:', bad); assert bad == 0

# 校验：mess/ev 的 dtt 与 v2.14 完全一致（对话不受影响）
b = {(r['DirName'], r['FileName']): r for r in cpk_tool.load_toc(BASE)[1]}
for k in byd:
    if k[0] == 'mess/ev' or k[1].startswith('messm'):
        assert cpk_tool.read_entry(DST, byd[k]) == cpk_tool.read_entry(BASE, b[k]), k
print('mess/ev 与 messm 全部未改动 ✓')

# 抽查 messc112
d = cpk_tool.read_entry(DST, byd[('mess', 'messc112_us.dat')])
m = mcd.Mcd(d)
tx = msg_texts(m)
print('\nmessc112:')
for mi in [1, 7, 8, 9, 20]:
    print('   M%-3d %r' % (mi, tx[mi][:44]))
print('   字库:', cpk_tool.read_entry(DST, byd[('mess','messc112_us.dtt')])[:4])

import hashlib
h = hashlib.sha1()
with open(DST, 'rb') as f:
    for chunk in iter(lambda: f.read(1 << 20), b''):
        h.update(chunk)
print('\nSHA1:', h.hexdigest())
print('=== v2.21 BUILD SUCCESS ===')
```

**build_v225_messc_v09.py** — v2.25 构建（v2.14b 基座 + v0.9 全部 37 对 messc 成对换入——当前部署版的构建脚本）

```
# -*- coding: utf-8 -*-
"""v2.25：v2.14b 基座 + v0.9 的全部 messc*_us.dat/.dtt 对（74 条目中逐字节有差异者）。

假说（由 v2.x 全系列实测交叉得出）：
- 对话/任务文本渲染约束 = 图集面积 >= 1M px（1024x1024=1M ✓ v2.14；512x2048=1M ✓ v0.9；
  256x2048/512x1024/256x1024 均 <1M ✗ v2.15/19/21/22-24）。推翻 ISSUES_SUMMARY P4 的
  "高<=1024" 表述——512x2048 高 2048 但 v0.9 实测对话正常。
- HUD 地名/任务名约束 = 图集宽 <= 512（256 ✓ 512 ✓ 1024 ✗）。
- 交集 512x2048 = v0.9 的 messc 配置，此前从未在 v2.14 基座上测过（patch/v2.22_merge 未构建）。

改动范围：仅 messc* 对（v0.9 版本）。campaign / camp_mission / ui_hud_common / mess/ev /
MVD / 菜单 全部保持 v2.14b —— 避开 P3 的 ui_hud_common 卡关源。
"""
import os, sys, hashlib

HERE = os.path.dirname(os.path.abspath(__file__))
sys.path.insert(0, HERE)
sys.path.insert(0, r"C:\Users\86159\.gemini\config\skills\ps3-localization\scripts")
import cpk_tool, cpk_repack

V214 = r'H:\ps3\game\rpcs3_ANARCHY REIGNS chs\dev_hdd0\game\BLES01232ANARCHYREIGNS\USRDIR\data000.cpk.v214b'
V09 = r'H:\ps3\game\2_work\patch\v0.9_full\data000.cpk'
OUT_DIR = r'H:\ps3\game\2_work\patch\v2.25_messc_v09'

def toc(path):
    _, rows = cpk_tool.load_toc(path)
    return {(r['DirName'], r['FileName']): r for r in rows}

def main():
    t14, t09 = toc(V214), toc(V09)
    keys = sorted(k for k in t09
                  if k[0] == 'mess' and k[1].startswith('messc') and k[1].endswith('_us.dat'))
    assert len(keys) == 37, len(keys)
    files = {}
    diff_dat = diff_dtt = same = 0
    for k in keys:
        dk = (k[0], k[1][:-4] + '.dtt')
        for src, dstk in ((k, k), (dk, dk)):
            b14 = cpk_tool.read_entry(V214, t14[dstk])
            b09 = cpk_tool.read_entry(V09, t09[dstk])
            if b14 == b09:
                same += 1
                continue
            files[dstk] = b09
            if dstk[1].endswith('.dat'):
                diff_dat += 1
            else:
                diff_dtt += 1
    print('messc pairs: %d | swapped dat=%d dtt=%d | identical entries skipped=%d'
          % (len(keys), diff_dat, diff_dtt, same))
    # dat 换而 dtt 不换仅在 dtt 两版字节相同（原版未动）时安全
    assert diff_dat >= diff_dtt, (diff_dat, diff_dtt)
    os.makedirs(OUT_DIR, exist_ok=True)
    dst = os.path.join(OUT_DIR, 'data000.cpk')
    cpk_repack.repack(V214, dst, files, verbose=False)
    cpk_repack.verify(dst, files)
    h = hashlib.sha1()
    with open(dst, 'rb') as f:
        for blk in iter(lambda: f.read(1 << 20), b''):
            h.update(blk)
    print('OUT', dst, os.path.getsize(dst))
    print('sha1', h.hexdigest())

if __name__ == '__main__':
    main()
```

**build_v226_sharp_messc.py** — v2.26 管线（512×2048 图集 0.90 字号重渲染：dat 仅原位重写 rects UV / dtt 仅像素区等长替换；v2.27 的底子）

```
# -*- coding: utf-8 -*-
"""v2.26：v2.25 基座上把 35 张 512x2048 messc 图集以 0.90 字号重渲染（修发虚）。

设计（最小变量）：
- 文本/编码/cmap/metrics/sec1/total_chars 一概不动（v0.9 文本原样，含中文地名）；
- 每个 cmap 字形以 round(cell_h*0.90) 字号重绘（v0.9 实测墨迹 0.71 偏小），
  逐字形墨盒宽若 >31 则单独缩号；
- rects：UV 改为新位置，w/h/参数字段保持原值（布局推进不变）；
- dat 仅原位重写 rects 段（大小不变）；dtt 仅替换像素数据区（头部/尺寸不变）；
- messc011 (256x1024) / messc113 (128x512) 跳过（v2.25 原样保留）。
产出 patch/v2.26_sharp_messc/data000.cpk —— 不自动部署，待 v2.25 实测通过后再换。
"""
import os, sys, struct
import numpy as np

HERE = os.path.dirname(os.path.abspath(__file__))
sys.path.insert(0, r"C:\Users\86159\.gemini\config\skills\ps3-localization\scripts")
sys.path.insert(0, HERE)
import atlas_build as ab   # 必须先于 cpk_tool：cpk_tool 会把 gemini 目录插到 path[0]
import dtt, dxt
from mcd import Mcd
import cpk_tool, cpk_repack
assert 'pad_x' in ab.render_glyph.__code__.co_varnames, \
    'wrong atlas_build loaded: %s' % ab.__file__

BASE = r'H:\ps3\game\2_work\patch\v2.25_messc_v09\data000.cpk'
OUT_DIR = r'H:\ps3\game\2_work\patch\v2.26_sharp_messc'
SIZE_RATIO = 0.90
BASELINE = 0.86
GAP = 1

files = {}

def process(key, dat, dttb):
    p = dtt.parse(dttb, 0)
    t = p['textures'][0]
    W, H = t['width'], t['height']
    if (W, H) != (512, 2048):
        return None  # messc011/113 原样
    old = dxt.dxt5(dtt.gtf_pixels(dttb, t), W, H)
    n = dtt._u32(dat, 4)
    ot, tt, st = dtt._u32(dat, 8), dtt._u32(dat, 0xc), dtt._u32(dat, 0x14)
    mo = ms = None
    for i in range(n):
        if dat[tt + 4 * i:tt + 4 * i + 4][:3] == b'mcd':
            mo, ms = dtt._u32(dat, ot + 4 * i), dtt._u32(dat, st + 4 * i)
    m = Mcd(dat[mo:mo + ms], h={'chunk_off': 0, 'chunk_size': ms})
    sm = ab.style_metrics(m)
    assert len(m.cmap) == len(m.rects)

    # 渲染：尝试从 0.90 逐级降号直到放得下
    for ratio in (SIZE_RATIO, 0.86, 0.82, 0.78):
        tiles = {}   # gi -> tile
        for st, meta in sm.items():
            fsize = max(8, int(round(meta['cell_h'] * ratio)))
            font = ab.load_font('noto_bold', fsize)
            color = ab.style_color(old, m, meta['gis'], W, H)
            for gi in meta['gis']:
                ch = chr(m.g2cp.get(gi, 63))
                im = ab.render_glyph(ch, font, meta['cell_h'], color, pad_x=0,
                                     baseline_ratio=BASELINE)
                if im.width > 31:  # 墨盒超格：缩号重绘该字形
                    f2 = max(8, int(fsize * 31 / im.width))
                    im = ab.render_glyph(ch, ab.load_font('noto_bold', f2),
                                         meta['cell_h'], color, pad_x=0,
                                         baseline_ratio=BASELINE)
                tiles[gi] = im
        # 打包
        pos, x, y = {}, 2, 2
        overflow = False
        order = sorted(tiles)
        for gi in order:
            im = tiles[gi]
            if x + im.width > W - 2:
                y += int(sm[0]['cell_h'] if 0 in sm else list(sm.values())[0]['cell_h']) + 2
                x = 2
                if y > H - 2:
                    overflow = True
                    break
            pos[gi] = (x, y)
            x += im.width + GAP
        bottom = y + max(sm[st]['cell_h'] for st in sm) + 2
        if overflow or bottom > H - 2:
            continue
        break
    else:
        raise RuntimeError('no fit')
    # 统一 y 步进按各自 style 的 cell_h（上面简化用第一组）——单 style 文件无影响
    print('  %s ratio=%.2f bottom=%d/%d glyphs=%d' %
          (key[1], ratio, bottom, H, len(tiles)))

    # 画布
    canvas = np.zeros((H, W, 4), dtype=np.uint8)
    for gi, (xx, yy) in pos.items():
        im = tiles[gi]
        canvas[yy:yy + im.height, xx:xx + im.width] = np.array(im)

    # dtt：仅替换像素数据区（长度不变）
    g = t['off']
    px_len = dtt._u32(dttb, g + 0x14)
    new_px = dxt_enc_bytes(canvas, px_len)
    px_off = g + dtt._u32(dttb, g + 0x10)
    assert px_off + px_len <= len(dttb)
    new_dtt = bytearray(dttb)
    new_dtt[px_off:px_off + px_len] = new_px

    # dat：原位重写 rects UV（大小不变）
    body = bytearray(dat)
    rect_off = mo + m.groups[2][0]
    for gi in range(len(m.rects)):
        xx, yy = pos[gi]
        im = tiles[gi]
        (hh, _u0, _v0, _u1, _v1, w, h, a, b, c) = m.rects[gi]
        struct.pack_into('>Ifffffffff', body, rect_off + gi * 40,
                         hh, xx / W, yy / H,
                         (xx + im.width) / W, (yy + im.height) / H,
                         w, h, a, b, c)
    new_dat = bytes(body)
    return new_dat, bytes(new_dtt)

def dxt_enc_bytes(canvas, expect_len):
    import dxt_enc
    px = dxt_enc.encode(canvas)
    assert len(px) == expect_len, (len(px), expect_len)
    return px

def main():
    _, rows = cpk_tool.load_toc(BASE)
    by = {(r['DirName'], r['FileName']): r for r in rows}
    keys = sorted(k for k in by if k[0] == 'mess' and k[1].startswith('messc')
                  and k[1].endswith('_us.dat'))
    for k in keys:
        dat = cpk_tool.read_entry(BASE, by[k])
        dttb = cpk_tool.read_entry(BASE, by[(k[0], k[1][:-4] + '.dtt')])
        out = process(k, dat, dttb)
        if out is None:
            print('  %s skip (dims not 512x2048)' % k[1])
            continue
        nd, nt = out
        assert len(nd) == len(dat) and len(nt) == len(dttb)
        files[k] = nd
        files[(k[0], k[1][:-4] + '.dtt')] = nt

    # 视觉抽查：重建后重新解码两张图的字形
    os.makedirs(OUT_DIR, exist_ok=True)
    dst = os.path.join(OUT_DIR, 'data000.cpk')
    cpk_repack.repack(BASE, dst, files, verbose=False)
    cpk_repack.verify(dst, files)

    # 重新读取验证墨迹
    _, rows2 = cpk_tool.load_toc(dst)
    by2 = {(r['DirName'], r['FileName']): r for r in rows2}
    d = cpk_tool.read_entry(dst, by2[('mess', 'messc112_us.dtt')])
    p = dtt.parse(d, 0); t = p['textures'][0]
    img = dxt.dxt5(dtt.gtf_pixels(d, t), t['width'], t['height'])
    alpha = img[..., 3]
    dd = cpk_tool.read_entry(dst, by2[('mess', 'messc112_us.dat')])
    n = dtt._u32(dd, 4); ot, st = dtt._u32(dd, 8), dtt._u32(dd, 0x14)
    o, s = dtt._u32(dd, ot), dtt._u32(dd, st)
    m = Mcd(dd[o:o + s], h={'chunk_off': 0, 'chunk_size': s})
    hs = []
    for r in m.rects[::8]:
        x0, x1 = int(round(r[1] * t['width'])), int(round(r[3] * t['width']))
        y0, y1 = int(round(r[2] * t['height'])), int(round(r[4] * t['height']))
        tile = alpha[y0:y1, x0:x1]
        nz = np.where((tile > 100).any(axis=1))[0]
        hs.append((nz.max() - nz.min() + 1) if len(nz) else 0)
    hs = np.array(hs)
    print('post ink height: median=%d mean=%.1f (target ~28)' %
          (np.median(hs), hs.mean()))
    empty = int((hs == 0).sum())
    print('empty glyph tiles: %d / %d' % (empty, len(hs)))
    assert empty == 0, 'some glyph tiles are empty!'

    from PIL import Image
    Image.fromarray(img[:, :, :3]).save(os.path.join(HERE, 'work', 'diag_v226_messc112.png'))

    import hashlib
    h = hashlib.sha1()
    with open(dst, 'rb') as f:
        for blk in iter(lambda: f.read(1 << 20), b''):
            h.update(blk)
    print('OUT', dst, os.path.getsize(dst))
    print('sha1', h.hexdigest())

if __name__ == '__main__':
    main()
```

**build_v227_w4096.py** — v2.27 构建脚本（256×4096 新交集实验——已写好未运行，用户暂停中；恢复时直接 python 运行即可）

```
# -*- coding: utf-8 -*-
"""v2.27：v2.25 基座上把 35 张 512x2048 messc 图集改为 256x4096（宽=256 满足地名 + 面积 1M 满足对话）
并合并 0.90 字号锐化（v2.26 管线）。

v2.25 实测：对话 ✓（512×2048=1M 面积假说成立）；地名 ✗（512 宽也灭 → 地名要求宽=256 整）。
交集 = 256×4096（=1M px；DXT5 数据量与 512×2048 相同 1MB → 像素区等长替换可行，
GTF 头 +0x20/+0x22 的 W/H 需 4 字节改写）。

其余：文本/cmap/metrics 零改动；dat 仅原位重写 rects 段 UV（w/h 字段保持 31/31）；
messc011/messc113 原版小图跳过。产出 patch/v2.27_messc_w256_h4096/data000.cpk。
"""
import os, sys, struct
import numpy as np

HERE = os.path.dirname(os.path.abspath(__file__))
sys.path.insert(0, r"C:\Users\86159\.gemini\config\skills\ps3-localization\scripts")
sys.path.insert(0, HERE)
import atlas_build as ab   # 必须先于 cpk_tool（见 v2.26 注释）
import dtt, dxt
from mcd import Mcd
import cpk_tool, cpk_repack
assert 'pad_x' in ab.render_glyph.__code__.co_varnames, \
    'wrong atlas_build loaded: %s' % ab.__file__

BASE = r'H:\ps3\game\2_work\patch\v2.25_messc_v09\data000.cpk'
OUT_DIR = r'H:\ps3\game\2_work\patch\v2.27_messc_w256_h4096'
SRC_DIM = (512, 2048)
DST_W, DST_H = 256, 4096
SIZE_RATIO = 0.90
BASELINE = 0.86
GAP = 1

files = {}

def process(key, dat, dttb):
    p = dtt.parse(dttb, 0)
    t = p['textures'][0]
    W, H = t['width'], t['height']
    if (W, H) != SRC_DIM:
        return None
    old = dxt.dxt5(dtt.gtf_pixels(dttb, t), W, H)
    n = dtt._u32(dat, 4)
    ot, tt, st = dtt._u32(dat, 8), dtt._u32(dat, 0xc), dtt._u32(dat, 0x14)
    mo = ms = None
    for i in range(n):
        if dat[tt + 4 * i:tt + 4 * i + 4][:3] == b'mcd':
            mo, ms = dtt._u32(dat, ot + 4 * i), dtt._u32(dat, st + 4 * i)
    m = Mcd(dat[mo:mo + ms], h={'chunk_off': 0, 'chunk_size': ms})
    sm = ab.style_metrics(m)
    assert len(m.cmap) == len(m.rects)

    for ratio in (SIZE_RATIO, 0.86, 0.82, 0.78):
        tiles = {}
        for st, meta in sm.items():
            fsize = max(8, int(round(meta['cell_h'] * ratio)))
            font = ab.load_font('noto_bold', fsize)
            color = ab.style_color(old, m, meta['gis'], W, H)
            for gi in meta['gis']:
                ch = chr(m.g2cp.get(gi, 63))
                im = ab.render_glyph(ch, font, meta['cell_h'], color, pad_x=0,
                                     baseline_ratio=BASELINE)
                if im.width > 31:
                    f2 = max(8, int(fsize * 31 / im.width))
                    im = ab.render_glyph(ch, ab.load_font('noto_bold', f2),
                                         meta['cell_h'], color, pad_x=0,
                                         baseline_ratio=BASELINE)
                tiles[gi] = im
        pos, x, y = {}, 2, 2
        overflow = False
        for gi in sorted(tiles):
            im = tiles[gi]
            if x + im.width > DST_W - 2:
                y += max(sm[st]['cell_h'] for st in sm) + 2
                x = 2
                if y > DST_H - 2:
                    overflow = True
                    break
            pos[gi] = (x, y)
            x += im.width + GAP
        bottom = y + max(sm[st]['cell_h'] for st in sm) + 2
        if overflow or bottom > DST_H - 2:
            continue
        break
    else:
        raise RuntimeError('no fit')
    print('  %s ratio=%.2f bottom=%d/%d glyphs=%d' %
          (key[1], ratio, bottom, DST_H, len(tiles)))

    canvas = np.zeros((DST_H, DST_W, 4), dtype=np.uint8)
    for gi, (xx, yy) in pos.items():
        im = tiles[gi]
        canvas[yy:yy + im.height, xx:xx + im.width] = np.array(im)

    # dtt：GTF 头 W/H 改写 + 像素区等长替换（1MB 不变）
    import dxt_enc
    g = t['off']
    px_len = dtt._u32(dttb, g + 0x14)
    new_px = dxt_enc.encode(canvas)
    assert len(new_px) == px_len, (len(new_px), px_len)
    new_dtt = bytearray(dttb)
    struct.pack_into('>HH', new_dtt, g + 0x20, DST_W, DST_H)
    px_off = g + dtt._u32(dttb, g + 0x10)
    new_dtt[px_off:px_off + px_len] = new_px

    # dat：原位重写 rects UV（w/h/参数字段保持原值）
    body = bytearray(dat)
    rect_off = mo + m.groups[2][0]
    for gi in range(len(m.rects)):
        xx, yy = pos[gi]
        im = tiles[gi]
        (hh, _u0, _v0, _u1, _v1, w, h, a, b, c) = m.rects[gi]
        struct.pack_into('>Ifffffffff', body, rect_off + gi * 40,
                         hh, xx / DST_W, yy / DST_H,
                         (xx + im.width) / DST_W, (yy + im.height) / DST_H,
                         w, h, a, b, c)
    return bytes(body), bytes(new_dtt)

def main():
    _, rows = cpk_tool.load_toc(BASE)
    by = {(r['DirName'], r['FileName']): r for r in rows}
    keys = sorted(k for k in by if k[0] == 'mess' and k[1].startswith('messc')
                  and k[1].endswith('_us.dat'))
    for k in keys:
        dat = cpk_tool.read_entry(BASE, by[k])
        dttb = cpk_tool.read_entry(BASE, by[(k[0], k[1][:-4] + '.dtt')])
        out = process(k, dat, dttb)
        if out is None:
            print('  %s skip (dims not %sx%s)' % (k[1], SRC_DIM[0], SRC_DIM[1]))
            continue
        nd, nt = out
        assert len(nd) == len(dat) and len(nt) == len(dttb)
        files[k] = nd
        files[(k[0], k[1][:-4] + '.dtt')] = nt
    print('swapped pairs: %d' % (len(files) // 2))

    os.makedirs(OUT_DIR, exist_ok=True)
    dst = os.path.join(OUT_DIR, 'data000.cpk')
    cpk_repack.repack(BASE, dst, files, verbose=False)
    cpk_repack.verify(dst, files)

    # 复检：尺寸、墨迹、空 tile
    _, rows2 = cpk_tool.load_toc(dst)
    by2 = {(r['DirName'], r['FileName']): r for r in rows2}
    d = cpk_tool.read_entry(dst, by2[('mess', 'messc112_us.dtt')])
    p = dtt.parse(d, 0); t = p['textures'][0]
    assert (t['width'], t['height']) == (DST_W, DST_H), (t['width'], t['height'])
    img = dxt.dxt5(dtt.gtf_pixels(d, t), DST_W, DST_H)
    alpha = img[..., 3]
    dd = cpk_tool.read_entry(dst, by2[('mess', 'messc112_us.dat')])
    n = dtt._u32(dd, 4); ot, st = dtt._u32(dd, 8), dtt._u32(dd, 0x14)
    o, s = dtt._u32(dd, ot), dtt._u32(dd, st)
    m = Mcd(dd[o:o + s], h={'chunk_off': 0, 'chunk_size': s})
    hs = []
    for r in m.rects:
        x0, x1 = int(round(r[1] * DST_W)), int(round(r[3] * DST_W))
        y0, y1 = int(round(r[2] * DST_H)), int(round(r[4] * DST_H))
        tile = alpha[y0:y1, x0:x1]
        nz = np.where((tile > 100).any(axis=1))[0]
        hs.append((nz.max() - nz.min() + 1) if len(nz) else 0)
    hs = np.array(hs)
    print('post: dims %dx%d | ink median=%d mean=%.1f | empty=%d/%d' %
          (DST_W, DST_H, np.median(hs), hs.mean(), (hs == 0).sum(), len(hs)))
    assert (hs == 0).sum() == 0, 'empty glyph tiles!'

    from PIL import Image
    Image.fromarray(255 - alpha[:2800, :]).save(
        os.path.join(HERE, 'work', 'diag_v227_alpha.png'))

    import hashlib
    h = hashlib.sha1()
    with open(dst, 'rb') as f:
        for blk in iter(lambda: f.read(1 << 20), b''):
            h.update(blk)
    print('OUT', dst, os.path.getsize(dst))
    print('sha1', h.hexdigest())

if __name__ == '__main__':
    main()
```

**check_dims_v225.py** — 两个 CPK 的 messc 尺寸/结构对比工具（size_tab 寻址、wcount、cmap/total_chars 自洽检查）

```
# -*- coding: utf-8 -*-
"""v2.25 前置核查：对比 v0.9_full 与 v2.14b 的 messc*_us 文件尺寸/结构。
验证：v0.9 messc dtt = 512x2048（宽<=512 满足 HUD，面积 1M 满足对话），
dat+dtt 成对、glyph/cmap/total_chars 自洽。"""
import os, sys, struct
from collections import Counter

HERE = os.path.dirname(os.path.abspath(__file__))
sys.path.insert(0, HERE)
sys.path.insert(0, r"C:\Users\86159\.gemini\config\skills\ps3-localization\scripts")
import cpk_tool
import dtt
from mcd import Mcd

V09 = r'H:\ps3\game\2_work\patch\v0.9_full\data000.cpk'
V214 = r'H:\ps3\game\rpcs3_ANARCHY REIGNS chs\dev_hdd0\game\BLES01232ANARCHYREIGNS\USRDIR\data000.cpk.v214b'

def toc(path):
    _, rows = cpk_tool.load_toc(path)
    return {(r['DirName'], r['FileName']): r for r in rows}

def probe(path, key):
    t = toc(path)
    row = t.get(key)
    if row is None:
        return None
    d = cpk_tool.read_entry(path, row)
    return d

def dtt_dims(d):
    p = dtt.parse(d, 0)
    texs = [(t['width'], t['height'], t['hash']) for t in p['textures']]
    return p['wtb_count'], texs

def dat_stats(d):
    n = dtt._u32(d, 4)
    off_tab, tag_tab, size_tab = dtt._u32(d, 8), dtt._u32(d, 0xc), dtt._u32(d, 0x14)
    mcd_n = cmap_n = 0
    total_chars_bad = 0
    subs_total = 0
    for i in range(n):
        if d[tag_tab + 4 * i:tag_tab + 4 * i + 4][:3] != b'mcd':
            continue
        mcd_n += 1
        off, size = dtt._u32(d, off_tab + 4 * i), dtt._u32(d, size_tab + 4 * i)
        chunk = d[off:off + size]
        m = Mcd(chunk, h={'chunk_off': 0, 'chunk_size': len(chunk)})
        cmap_n = max(cmap_n, len(m.cmap))
        for mi in range(m.msg_count):
            try:
                _, _, subs, total_chars = m.message_subs(mi)
            except Exception:
                continue
            s = sum(x['chars'] for x in subs)
            subs_total += s
            if total_chars is not None and total_chars != s:
                total_chars_bad += 1
    return mcd_n, cmap_n, subs_total, total_chars_bad

def main():
    for tag, path in (('v0.9_full', V09), ('v2.14b', V214)):
        t = toc(path)
        keys = sorted(k for k in t if k[0] == 'mess' and k[1].startswith('messc') and k[1].endswith('_us.dat'))
        print('=== %s: %d messc dat entries ===' % (tag, len(keys)))
        for k in keys[:3] + keys[-3:]:
            dd = probe(path, k)
            dt = probe(path, (k[0], k[1][:-4] + '.dtt'))
            wc, texs = dtt_dims(dt)
            mcd_n, cmap_n, subs, bad = dat_stats(dd)
            print('  %s: dtt wcount=%d tex=%s | dat mcd=%d cmap=%d subs=%d total_chars_bad=%d'
                  % (k[1], wc, texs, mcd_n, cmap_n, subs, bad))
        # all dims uniformity
        dims = set()
        for k in keys:
            dt = probe(path, (k[0], k[1][:-4] + '.dtt'))
            wc, texs = dtt_dims(dt)
            dims.add((wc, tuple(texs)))
        print('  distinct (wcount, tex) combos:', dims)

if __name__ == '__main__':
    main()
```

**extract_messc112.py** — 从 CPK 提取 messc112 文件对的工具（本文档附录的 4 版本文件即由它生成）

```
# -*- coding: utf-8 -*-
"""提取 4 个版本的 messc112_us.dat/.dtt 为独立文件（供对方 agent 直接研究）：
orig（原版）、v214（1024×1024 对话✓）、v221（512×1024 实验）、v225（当前部署=v0.9 内容 512×2048）。
同时打印各版尺寸/结构摘要。"""
import os, sys
sys.path.insert(0, r"C:\Users\86159\.gemini\config\skills\ps3-localization\scripts")
import cpk_tool, dtt

OUT = r'H:\ps3\game\1_doc\attach_messc112'
os.makedirs(OUT, exist_ok=True)

SRC = [
    ('orig', r'H:\ps3\game\rpcs3_ANARCHY REIGNS chs\dev_hdd0\game\BLES01232ANARCHYREIGNS\USRDIR\data000.cpk.orig'),
    ('v214', r'H:\ps3\game\rpcs3_ANARCHY REIGNS chs\dev_hdd0\game\BLES01232ANARCHYREIGNS\USRDIR\data000.cpk.v214b'),
    ('v221', r'H:\ps3\game\rpcs3_ANARCHY REIGNS chs\dev_hdd0\game\BLES01232ANARCHYREIGNS\USRDIR\data000.cpk.v221'),
    ('v225', r'H:\ps3\game\rpcs3_ANARCHY REIGNS chs\dev_hdd0\game\BLES01232ANARCHYREIGNS\USRDIR\data000.cpk'),
]

for tag, path in SRC:
    _, rows = cpk_tool.load_toc(path)
    by = {(r['DirName'], r['FileName']): r for r in rows}
    for ext in ('.dat', '.dtt'):
        d = cpk_tool.read_entry(path, by[('mess', 'messc112_us' + ext)])
        fn = os.path.join(OUT, 'messc112_us.%s%s' % (tag, ext))
        open(fn, 'wb').write(d)
        info = ''
        if ext == '.dtt':
            p = dtt.parse(d, 0)
            t = p['textures'][0]
            info = 'dtt %dx%d wcount=%d hash=%d' % (t['width'], t['height'], p['wtb_count'], t['hash'])
        print('%-22s %8d B  %s' % (os.path.basename(fn), len(d), info))
print('OUT', OUT)
```

### 13.4 核心依赖库位置（脚本 import 的模块，不在本文档内嵌）

| 模块 | 路径 | 说明 |
| --- | --- | --- |
| cpk_tool / cpk_repack / dtt / mcd / dat / mcd_build / text_encode / crilayla / dxt / dxt_enc | `C:\Users\86159\.gemini\config\skills\ps3-localization\scripts\`<br>（同套副本在 `C:\Users\86159\.workbuddy\skills\ps3-localization\scripts\`） | CPK 读写 / 追加式 repack / dtt 与 mcd 解析 / 字库渲染底库 |
| atlas_build（新版） | `H:\ps3\game\2_work\cpk\atlas_build.py` | 支持 size_ratio/pad_x/stroke_w 的字库渲染管线（style_metrics/style_color/render_glyph/load_font）——**import 顺序必须先于 cpk_tool**（见 §10 大坑） |
| 字体 | atlas_build 内 FONTS 表（noto_bold） | 渲染中文字形用 |

注：03_work 的 v2.x 脚本用 `C:\Users\86159\.workbuddy\...` 路径，2_work/cpk 的新脚本用 gemini 路径——两套 skills 目录内容基本同源，但 **atlas_build.py 只有 2_work/cpk 是新版**。

本文档由 2026-11-20 会话生成，配套权威实验日志 `H:/ps3/game/1_doc/ISSUES_SUMMARY.md`（更细粒度的逐版取证）。
若 v2.27 实测有新结果，请同时更新两者。
