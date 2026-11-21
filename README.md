# 游戏汉化 · 字库问题评审包（三份）

单文件静态页（图片 base64 内嵌），供评审 agent / 工程师浏览：

| 报告 | 在线地址 | 本仓库路径 |
|---|---|---|
| 🎮 Wii《哆啦A梦 秘密道具王决定战》(REMJ8P)：页 4-7 缺字 / 字形大小不一 | https://wei134102.github.io/doraemon-font-review/ | [index.html](index.html) |
| 🎮 N64《StarCraft 64》：汉化字库/引擎问题完整诊断（2-hook 引擎现状 / CHK 引用链审计 / 三套引擎方案实测对比 / 外部独立分析对照；旧版备份 [n64/index0.html](n64/index0.html)） | https://wei134102.github.io/doraemon-font-review/n64/ | [n64/index.html](n64/index.html) |
| 🎮 PS3《Anarchy Reigns 疯狂乱世》(BLES01232 欧版)：RPCS3 简体中文汉化 — 技术总结与交接（12 章：文件格式逆向 / 尺寸约束模型 / 已确证根因 P1-P8 / v2.27 构建方案 / 陷阱与回退链） | https://wei134102.github.io/doraemon-font-review/ps3/ | [ps3/index.html](ps3/index.html) |

## Wii 报告内容速览

1. 背景：Wii 字库 = BRTEX 纹理页 + 槽位码流；T35 = 8 页 × 400×400 IA4 / 25×25 格 / 2048 槽
2. 已解决：自拼容器 bug（非 Patricia 字典）→ 字虚/行叠压
3. **问题 A**：对话场景从不绑定页 4-7（Dolphin 纹理 dump 只有页 0-3，diff=0）→ 477 个新字缺字；而同一 DOL 标题场景探针能显示页 4-7
4. **问题 B**：字形大小不一（32px 旧字形重采样 vs 16px Noto 原生混排）
5. 反汇编已知信息 + 5 个待裁决问题（Q1-Q5）

## 约束

评审仅做离线分析：不启动模拟器/游戏，不修改仓库外文件；建议以文字/代码补丁形式给出。
