# 哆啦A梦Wii（REMJ8P）汉化 · 字库问题评审包

单文件静态页（`index.html`，图片 base64 内嵌），供评审 agent / 工程师浏览：

- 在线浏览（GitHub Pages）：https://wei134102.github.io/doraemon-font-review/
- 直接看仓库里的 [index.html](index.html) 也可以

## 内容

1. 背景速览：Wii 字库 = BRTEX 纹理页 + 槽位码流；T35 = 8 页 × 400×400 IA4 / 25×25 格 / 2048 槽
2. 已解决：自拼容器 bug（非 Patricia 字典）→ 字虚/行叠压
3. **当前问题 A**：对话场景从不绑定页 4-7（Dolphin 纹理 dump 只有页 0-3，diff=0）→ 477 个新字（槽 1024-1499）全部缺字；而同一 DOL 标题场景探针能显示页 4-7
4. **当前问题 B**：字形大小不一（32px 旧字形重采样 vs 16px Noto 原生混排）
5. 反汇编已知信息 + 5 个待裁决问题（Q1-Q5）

## 约束

评审仅做离线分析：不启动模拟器，不修改仓库外文件；建议以文字/代码补丁形式给出。
