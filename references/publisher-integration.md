# video-publisher 集成手册（自动模式：B站/小红书铺草稿）

前提：本机已安装 `video-publisher` skill 并完成 onboarding，且已用日常浏览器登录对应平台（登录态会被检查）。

## 标准命令（每期照抄）

```bash
export PATH="$HOME/.local/bin:$PATH" \
  VIDEO_PUBLISHER_V2_LOCKS_ROOT="<工作区>/.workbuddy/vp-jobs/.locks"
cd <video-publisher skill 目录>

# 第 1 步：package 校验（每个平台单独跑，秒级）
node scripts/check-package.mjs xiaohongshu "<package.json>"
node scripts/check-package.mjs bilibili "$PKG"

# 第 2 步：前置闸——inspect-only 探登录态（30 秒，避免白传大文件）
./scripts/run-safe-platforms.sh "<package.json>" <job-suffix> xiaohongshu bilibili \
  --state-root "<工作区>/.workbuddy/vp-jobs" --inspect-only

# 第 3 步：全量铺草稿（上传+元数据+声明+验证）
./scripts/run-safe-platforms.sh "<package.json>" <job-suffix> xiaohongshu bilibili \
  --state-root "<工作区>/.workbuddy/vp-jobs"
```

- `job-suffix`：`<日期>-<主题拼音>` 形式（如 `20261004-lengzhishi01`）；同 package + 同 suffix = 同一 job，重跑即幂等 resume
- **长任务一律后台跑 + 完成通知**，不轮询

## package.json 最小模板

```json
{
  "videoPath": "<期目录原视频绝对路径>",
  "title": "<标题>",
  "description": "<描述>",
  "douyinDescription": "<同描述>",
  "douyinTopics": ["话题1", "话题2"],
  "bilibiliDescription": "<B站结构化简介>",
  "bilibiliTags": ["标签"],
  "bilibiliAllowedAutoTags": [],
  "xhsTitle": "<≤20 字>",
  "xhsTopics": ["话题"],
  "cover": {
    "uploadCustomCover": false,
    "vertical3x4Path": "", "horizontal4x3Path": ""
  }
}
```

要点：`videoPath` 直接指期目录原视频（不改名不复制）；`uploadCustomCover: false`（默认封面 = 首帧封面卡）；`bilibiliAllowedAutoTags` 留空防误挂标签。

## 判读规矩

- 只有 `status: "ready"` 且 `missing: []` 才算完成；「action was attempted」不等于成功
- verify 阶段读的是页面实况，不信 state 自述
- READY 后 task space 页面保持打开，交给用户过目点发布

## 失效恢复手册

### INPUT_CHANNEL_BROKEN（视口 0×0）
1. 查进程：`pgrep -fl "ego"`——进程活着 = 窗口被关/最小化
2. `open -a "ego lite"` 唤起（macOS）后重跑同一命令
3. **绝不杀进程**——杀进程会丢草稿

### SELECTOR_DRIFT「video input missing」
1. 首次：直接幂等 resume 一次（新页面组件未挂载，属时序问题）
2. 复发：读 evidence 里 `visibilityState/readyState`——
   - `hidden` → 页面生命周期问题，resume 前激活窗口
   - `visible + complete` 但组件缺失、pageSample 空 → SPA 冷加载挂死，**必须重载页面**（激活重试救不了）；根治补丁：上传探测循环内第 4 次失败后 `gotoAndWait` 重新导航

### USER_CONTROL（用户接管了 task space）
- 硬停止，不可自动夺回。等用户明确说继续，再 `takeOverTaskSpace(<spaceId>)` 接管后 resume
- `takeOverTaskSpace` / `switchTaskSpace` / `listTabs` / `js()` 可通过 `ego-browser nodejs -e` 直调（js() 为页面内执行入口）

### UPLOAD_STALLED（上传停滞）
1. 先读 evidence `pageSample`：有「当前速度/剩余时间」= 在传，算剩余时间耐心等
2. 速度 0.0-0.1MB/s 且多轮不动（对比其他平台同文件速度）= 连接半死，resume 干等无用
3. **先查系统代理例外名单**：`scutil --proxy`——平台 CDN 上传域（如 B站 bilivideo.com）不在例外名单时，上传流量绕代理节点，速度掉 10 倍以上。修复 = networksetup 追加例外域（先备份原列表），然后重载页面（`location.href` 跳转打断死连接）再 resume
4. 环境修复后速度应从分钟级变秒级；仍是 37% 不动 → 停手交用户手动（页面在浏览器里看得见）

### 通用经验
- 环境类修复（代理例外、补丁）落地后记录到项目文档，代理客户端更新可能重写系统代理配置
- 适配器补丁（小红书激活重载）打在 `scripts/v2/platforms/xiaohongshu.mjs` 的 upload 函数；video-publisher 上游更新会覆盖本地补丁，更新前先 diff 备份
- 一次会话内同一动作失败 2 次 = 换手段，不无限重试；修不好停手交用户，附完整证据路径
