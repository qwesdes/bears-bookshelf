# 小手机自定义 APP 制作说明（第 4-5 节）

## 4. 声明文件与角色生成链路

声明文件用于 `AiPhone.ai.generate` 这条 APP 内角色生成链路。APP 调用 `AiPhone.ai.generate({ characterId, appTags })` 时，宿主会按当前角色、当前 APP 和 `appTags`，选择需要进入本次生成的预设条目。

* `presets.json`：LLM 提示词条目，声明角色输出场景，约束角色按该场景的规则和格式输出。

### 4.1 声明文件如何参与 `AiPhone.ai.generate`

小手机里的预设系统包括**通用条目及场景条目**，最终按预设系统里的顺序组装。

通用条目：角色设定、角色性格、用户人设、核心记忆、长期记忆、全局事件流、世界书等，组装时替换为真实内容。

场景条目：聊天线上输出、剧情模式输出、生成日记输出等，用来控制输出格式、让 AI 理解当前场景。

最终提示词 = 通用条目 + 场景条目。APP 只需写一个场景条目，调用 `AiPhone.ai.generate` 并传入它的 `appTags`，角色就会自动带上全量人设、记忆等按该场景输出。

**tags 选择规则**：只有「条目的 tags ⊆ 本次 appTags」的条目才会进入提示词。宿主通用条目通常没有 tags，进入所有链路。

例（本次 `appTags = ["sms", "reply"]`）：

|条目 tags|是否进入|原因|
|-|-|-|
|无|✅|通用条目|
|`["sms"]`|✅|是子集|
|`["sms", "reply"]`|✅|都在|
|`["sms", "proactive"]`|❌|`proactive` 不在|

**tags 书写约定**：英文数组，推荐 `[主功能, 子场景]`。`directives` 和 `plusActions` 不依赖 tags；只有预设和 `ai.generate({ appTags })` 需要严格对应。

**输出后处理**：`ai.generate` 只返回文本，APP 拿到 `result.text` 后自己解析。

**世界书**：角色绑定的世界书自动进入生成链路，APP 无需处理。

### 4.2 presets.json 最小结构

```json
{
  "presets": [
    {
      "id": "sms-reply-style",
      "name": "短信回复",
      "content": "你正在短信 APP 中回复用户……输出格式固定为：[短信回复:内容]。",
      "enabled": true,
      "tags": ["sms", "reply"]
    }
  ]
}
```

---

## 5. SDK 参考

### 5.1 入口

`window.AiPhone`（兼容别名 `window.AiPhoneApp`），所有接口都是异步 Promise。

### 5.2 速查表

|能力|主要方法|所需权限|
|-|-|-|
|APP 信息|`app.getManifest` / `getCapabilities` / `getLaunchContext` / `close`|无|
|私有数据库|`db.create` / `list` / `get` / `update` / `delete`|`app.data.read` / `app.data.write`|
|读取角色|`characters.list` / `get` / `readState` / `writeState` / `readRelations`|`characters.read` 等|
|用户资料|`user.getProfile` / `getPersona` / `getPreferences`|`user.profile.read` 等|
|外部 API|`network.fetch`|`network.fetch` + manifest `network.allowedDomains`|
|角色生成文本|`ai.generate`|`ai.generate`|
|AI 生图|`ai.generateImage`|`ai.generateImage`|
|通用模型|`ai.chat` / `ai.embed` / `ai.classify`|同名权限|
|语音|`voice.readProfiles` / `tts` / `stt` / `clone` / `play`|`voice.*`|
|日程|`calendar.read` / `create` / `update` / `write`|`calendar.read` / `calendar.write`|
|世界书|`world.list` / `get` / `read`|`world.read`|
|媒体|`media.pick` / `save`|`media.pick` / `media.save`|
|媒体库|`media.put` / `get` / `delete`|`app.data.write` / `app.data.read`|
|APP 工具|`tools.handle` / `invoke`|`chat.tools`|
|发卡片|`chat.sendCard` / `updateCard`|`chat.sendCard`|
|写聊天历史|`chat.writeHistory`|`chat.write` 或 `chat.sendMessage`|
|请求角色回复|`chat.requestReply`|`chat.requestReply`|
|读聊天历史|`chat.getCurrentSession` / `readHistory`|`chat.read`|
|监听事件|`AiPhone.on("chat.message.created", …)`|`chat.read` 或 `chat.read.background`|
|读记忆|`memory.readCore` / `readLongTerm` / `readShortTerm` / `search`|`memory.read*`|
|写记忆|`memory.add` / `addTimeline` / `deleteTimeline`|`memory.write`|
|UI 提示|`ui.toast` / `showNotification` / `confirm`|`ui.toast` / `ui.notification`|
|通知红点|`notifications.*`|`notifications.read` / `write`|
|定时任务|`tasks.schedule`|`tasks.schedule`|
|钱包|`wallet.get` / `pay`|`wallet.read` / `wallet.pay`|

### 5.3 APP 信息

```js
const launchContext = await AiPhone.app.getLaunchContext();
```
从卡片进入带 `messageId`、`sessionId`、`directiveId` 等；从 + 号面板进入带 `source="chat_plus_action"`、`characterId`、`entry` 等；桌面打开通常为 `null`。

### 5.4 私有数据库

```js
const item = await AiPhone.db.create("orders", { title: "冬阴功汤", status: "draft" });
const list = await AiPhone.db.list("orders", { limit: 100 });
const one  = await AiPhone.db.get("orders", item.id);
await AiPhone.db.update("orders", item.id, { status: "paid" });
await AiPhone.db.delete("orders", item.id);
```
**不提供 upsert**：先 `list` 找 id 再 `update`，找不到再 `create`。大文件不要以 dataURL 存进记录。

### 5.5 读取角色

```js
const launchContext = await AiPhone.app.getLaunchContext();
const characters = await AiPhone.characters.list();
const selectedCharacterId = launchContext?.characterId || characters[0]?.id;
```

### 5.7 外部 API

只能访问 manifest `network.allowedDomains` 中的域名（支持 `*.example.com`，不允许 localhost/内网）。默认浏览器直连；目标没开 CORS 时传 `proxy: true`，代理时 `timeoutMs` 不超过 120 秒。

```js
const response = await AiPhone.network.fetch({ url, method: "GET", timeoutMs: 30000 });
if (response.ok) console.log(response.json || response.text);
```

### 5.8 让角色生成文本

```js
const result = await AiPhone.ai.generate({
  characterId: selectedCharacterId,
  appTags: ["bookshelf", "note"],
  messages: turns.slice(-24),
  instruction: "……"
});
// result.text 最终回复；result.appendMessages 本轮新增上下文，下一轮作为 messages 传回
```
多角色：传 `characterIds` 数组或群聊 `sessionId`，输出格式由自己的预设约定，暂不支持工具调用。

### 5.10 媒体库（大文件正确存法）

```js
const stored = await AiPhone.media.put({ dataUrl });   // → { ref: "media-store://..." }，单件上限 25MB
await AiPhone.db.create("clips", { audioRef: stored.ref });
const img = await AiPhone.media.get({ ref });          // → { dataUrl }，不要用 blob objectURL
await AiPhone.media.delete({ ref });                   // 删记录时同步删媒体
```
播放音频一律用 `AiPhone.voice.play`，不要自建 `<audio>` 或 Web Audio。

### 5.11 注册和调用 APP 工具

工具只用于读写状态、调用接口这类真实动作；只是生成文字用 `ai.generate`。

* `permissions` 需包含 `chat.tools`。
* `visibility: "shared"` 进入“设置 → 聊天工具箱”，普通聊天可调用；多个 shared 工具会折叠成一个工具套件。
* APP 自己的生成链路要看到工具，预设条目里需要 `{{tools}}`。

```json
"extensions": { "tools": [{
  "id": "submit-takeout-review",
  "name": "提交外卖评价",
  "description": "……只有需要真正提交评价时才调用。",
  "visibility": "shared",
  "handler": "submitTakeoutReview",
  "parameterSchema": { "type": "object", "properties": { "orderId": { "type": "string" } }, "required": ["orderId"] }
}]}
```

```js
AiPhone.tools.handle("submitTakeoutReview", async (args, context) => {
  const review = await AiPhone.db.create("reviews", { ...args, createdAt: new Date().toISOString() });
  return { success: true, data: { reviewId: review.id }, userNotice: "外卖评价已提交" };
});
```
handler 里的异步操作要 `await` 或返回 Promise，否则隐藏运行环境可能提前结束。

### 5.12-5.14 聊天联动

* `chat.sendCard({ characterId, role, summary, historyText, card })`：发卡片，`historyText` 进入 AI 可见历史，写真实名字不写占位词。
* `chat.updateCard({ messageId, sessionId, status, ... })`：更新原卡片。
* `chat.writeHistory({ characterId, role, content })`：只让角色知道发生过某事。
* `chat.requestReply({ characterId })`：触发普通聊天回复，不加载 APP 预设。

### 5.17-5.18 读聊天与监听

`getCurrentSession` 只在从卡片/+ 号进入时有值。`AiPhone.on("chat.message.created", handler)` 需在 `extensions.events` 声明，`background: true` 时 APP 未打开也会运行。

### 5.19-5.20 记忆

读：`memory.readCore` / `readLongTerm` / `readShortTerm` / `search`。写：`memory.add`（只写真正重要的）、`memory.suggest`（待确认）、`memory.addTimeline`（写短期事件流；已用 sendCard/writeHistory 写过就不重复）。

### 5.21-5.24 其他

`ui.toast` / `ui.confirm`；`notifications.create` / `setBadge`；`tasks.schedule`（浏览器关闭时不跑，重开补跑，action 不继承 appTags）；`wallet.get` / `wallet.pay`。
