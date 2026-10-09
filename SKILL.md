---
name: video-operations
description: This skill should be used when the user delivers a finished vertical short-video (口播成片) and wants it published across Douyin/Bilibili/Xiaohongshu. It runs a tuned multi-platform pipeline (视频运营): compliance pre-check of the narration script, Douyin publish-text generation, and Bilibili/Xiaohongshu draft deployment via a configured video-publisher integration on Ego Lite browser (auto mode) or a manual SOP (fallback). Trigger phrases include 发布视频、铺草稿、发抖音、视频发布、素材包、成片交了、老地方、视频运营.
agent_created: true
---

# 视频运营（口播短视频 · 抖音/B站/小红书）

把一条口播成片变成三平台「打开即发」状态。核心原则：

- **发布按钮永远归用户**，Agent 只做到草稿 READY / 素材就绪
- **期目录极简**：只有用户的原视频（只读）+ 一个 `抖音发布文案.txt`
- **所有工作产物**（package.json、工作副本）放 Agent 自己的工作目录（如 `.workbuddy/ashuo-work/`），不污染用户文件夹
- **浏览器自动化基于 Ego Lite**（`ego-browser` 命令行 + task space 隔离）：自动模式的草稿铺制、失效恢复均在其上运行；本 skill 的使用体验按 Ego Lite 调优，移植到其他浏览器自动化时参考 references/publisher-integration.md 对应替换

## 维护节奏（约定）

**每 8-10 期集中更新一次本 skill**：把期间新踩的坑、新修法、流程变更合并进来；期间新经验先记在项目台账，攒批更新，避免频繁改动。

## 标准流程（每期照抄）

### 1. 接收与探测
1. 期目录 = 用户丢的文件夹（内含 `MTVideo.MP4` 或同名成片）。确认成片存在，读时长/编码/分辨率：
   ```bash
   ffmpeg -hide_banner -i <视频> 2>&1 | grep -E "Duration|Stream.*Video"
   ```
   - h264 → 直接用；HEVC(hvc1) → 平台全兼容，也可直接用（实测抖音/B站/小红书均通过）
   - 时长检查：抖音硬限 900 秒（超了先和用户商量切条）
2. 抽帧看首帧：`ffmpeg -ss 0 -i <视频> -frames:v 1 -q:v 3 -pix_fmt yuvj420p f0.jpg`。**经验：用户首帧通常是自制封面卡（带标题花字），直接当平台默认封面用，不做封面加工**（若首帧不是封面卡，先和用户确认封面方案再继续）。

### 2. 文案合规词检（发布前 30 秒，永不可跳）
拿到口播终稿后，按 references/compliance-checklist.md 逐条检查。命中即向用户标注风险等级，用户拍板后照发（交稿即终稿，不擅自改词）。历史教训：粗俗语言/人身攻击词（如「猪」「骂」系密集出现）可能命中平台「不文明用语」条款导致推荐压制。

### 3. 抖音：单 TXT 素材包
按 references/douyin-txt-template.md 在期目录写 `抖音发布文案.txt`（标题 ≤30 字、描述含话题、搜索关键词、发布步骤）。**不生成视频/封面副本**——用户直接用期目录原视频。注意：小红书平台标题上限 20 字，抖音 30 字，两处分开写。

### 4. B站/小红书铺草稿（二选一）
- **自动模式**：本机装有已调教的 `video-publisher` skill 时，按 references/publisher-integration.md 执行（前置闸 → 全量铺 → 失效恢复）。package 的 videoPath 直接指向期目录原视频，uploadCustomCover 一律 false。
- **人工模式**：无 video-publisher 环境时，在 TXT 里附三平台人工发布步骤，交付即止。

### 5. 发布后验证口径
- 抖音：发布后 2 小时看「播放来源」出现「推荐」= 通道正常（来源 100% 关注/朋友页且持续 24h+ 才可判限流，前几小时小池子试探是常态，部分数据次日更新）
- 账号有历史数据时对齐基线（如 2s 跳出率基线），异常先列可能原因不下结论；证据不足时冻结一切删除/重发动作
- **绝对禁止**用自动化浏览器碰抖音（会被限流抑制推荐，历史实锤）

## 失效恢复速查（详见 references/pitfalls.md）

| 症状 | 第一动作 |
|---|---|
| Ego/浏览器视口 0×0 | `open -a` 唤起应用，**绝不杀进程**（杀进程丢草稿） |
| 上传页白屏/组件缺失 | 带界重试 + 第 4 次探测失败后导航重载（SPA 冷加载挂死只有 reload 能救） |
| 上传速度 0.0-0.3MB/s 长期不动 | 查系统代理例外名单是否覆盖平台 CDN 上传域（如 B站 bilivideo.com），`scutil --proxy` |
| USER_CONTROL（用户接管） | 硬停等授权；用户说继续后 `takeOverTaskSpace(id)` 再 resume |
| SELECTOR_DRIFT 首次出现 | 幂等 resume 一次（页面冷启动未挂载）；复现则按白屏路径处理 |

## 私有化说明

本 skill 的自动模式依赖用户本机的 video-publisher 安装与平台登录态。移植到新环境时：先跑一次 `--inspect-only` 验证登录态，再决定自动/人工模式；期目录规范、TXT 模板、词检清单与平台无关，任何环境通用。
