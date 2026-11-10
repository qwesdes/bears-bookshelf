# 小手机自定义 APP 制作说明

## 1. 这是什么

你正在为"仿真 AI 互动小手机"制作一个能安装进【应用市场】的自定义 APP，可以是购物、打卡、短信、外卖、论坛、日记、工具箱等独立应用。

**心智模型**：一个 APP 就是一个跑在小手机 iframe 沙盒里的网页。它**不能**直接读取宿主内部数据，只能通过小手机提供的安全 SDK（`window.AiPhone`）请求能力——读写自己的私有数据、读取角色、让角色生成内容、向聊天室发卡片等。

一个 APP 包由三类东西组成：

* **前端**：`index.html`（必需）+ `assets/` 静态资源。
* **manifest.json**：声明 APP 身份、权限、扩展入口。
* **声明文件（可选）**：`presets.json`，声明 APP 场景的提示词条目，和角色生成链路联动。

> 最快验证：只写一个 `index.html` 就能做原型测试。正式发布再补 `manifest.json` 和 `icon.png`。

---

## 2. 应用包结构

最终由用户压缩成 `.zip` 上传。最小结构：

```txt
my-app/
├── manifest.json
├── index.html
├── icon.png
└── assets/
```

复杂 APP 可在根目录额外放声明文件。**没有相关需求就不要生成**；一旦生成，必须在 `manifest.resources` 里声明它们。

|文件|职责（一句话）|
|-|-|
|`manifest.json`|APP 身份、权限、扩展入口。正式发布必需|
|`index.html`|前端入口。原型测试时可单独存在|
|`icon.png`|APP 图标。正式发布建议必需|
|`assets/`|图片、音频、字体等静态资源|
|`presets.json`（可选，需要自定义 APP 内角色有全局全量记忆、日程信息、绑定世界书等时使用）|提示词条目，用于约束角色输出内容、格式等。会进入宿主提示词预设管理系统，用户可以自定义编辑。调用 `AiPhone.ai.generate` 后，APP 定义的预设条目将会追加在后面，共同组成提示词。|

---

## 3. manifest.json 字段参考

`manifest.json` 必须是 JSON 对象。完整示例：

```json
{
  "sdkVersion": "1.0",
  "id": "waimai.demo",
  "name": "浮岛外卖",
  "version": "1.0.0",
  "author": "你的名字",
  "description": "一个可以浏览店铺、选择餐品，并和聊天室外卖卡片联动的 APP。",
  "icon": "icon.png",
  "entry": "index.html",
  "permissions": ["app.data.read", "app.data.write", "characters.read", "chat.sendCard", "chat.requestReply", "ai.generate", "ui.toast"],
  "resources": { "presets": ["presets.json"] },
  "extensions": {
    "chat": {
      "directives": [{
        "id": "waimai-order",
        "label": "外卖下单",
        "syntax": "[外卖下单:收餐人:店铺:餐品:备注]",
        "description": "当角色想主动给用户点一份外卖，并让用户确认下单时使用。",
        "card": {
          "appLabel": "浮岛外卖", "title": "确认外卖订单", "subtitle": "{{店铺}}",
          "body": "{{餐品}} · {{备注}}", "status": "待确认", "accentColor": "#f97316",
          "sections": [{ "title": "订单信息", "rows": [
            { "label": "收餐人", "value": "{{收餐人}}" },
            { "label": "店铺", "value": "{{店铺}}" },
            { "label": "餐品", "value": "{{餐品}}" },
            { "label": "备注", "value": "{{备注}}" }
          ]}],
          "actions": [{ "label": "确认下单" }]
        }
      }],
      "plusActions": [{
        "id": "order-food-for-character", "label": "给TA点外卖",
        "entry": "order-food-for-character", "presentation": "panel", "panelHeight": "72vh",
        "description": "从聊天室打开外卖选择面板，给当前聊天对象点一份外卖。"
      }]
    }
  }
}
```

### 3.1 顶层字段

* **id**：可选的开发者标识，只能用英文、数字、点、下划线、短横线。安装和市场发布会生成自己的内部 APP ID，**创作者不需要手动保证全局唯一**。
* **permissions**：只声明实际会用到的能力（清单见第 6 节）。需要外部 API、语音、工具、钱包、定时任务时，再按需增加对应权限和字段。

### 3.2 resources：只声明包里真实存在的文件

```json
"resources": {
  "presets": ["presets.json"],
  "assets": ["assets/menu-cover.png"]
}
```

* `assets` 声明图片、音频、字体等静态资源路径，APP 运行时可用 `AiPhone.app.getAssetUrl("assets/menu-cover.png")` 拿到可访问地址。
* 如果需要注册 APP 工具，不要额外创建 `tools.json`，请直接写在 `manifest.extensions.tools`。
* 如果 APP 有自己的语音配置、音色列表或提示音文件，可放在 `assets` 或 APP 私有数据里，再由 APP 自己读取。

### 3.3 extensions（可选）

`extensions.chat` 用来把 APP 接到小手机最核心的 APP——聊天 APP。富媒体指令是角色可以使用的操作，这里注册的新富媒体指令会追加在后面；+ 号面板是用户在输入栏里使用的面板。

> APP 的聊天室扩展统一写在 `extensions.chat.*`，后台事件监听统一写在 `extensions.events`。

#### extensions.chat.directives —— 富媒体指令

给角色注册一段它可以直接写进聊天回复的文本标记，格式就是 `syntax` 里定义的样子。

* 配好后，角色生成回复前会收到这条可用指令说明。
* 角色在回复里写出它时，宿主自动把标记渲染成 APP 卡片，并作为一条角色行为写进 AI 可见的聊天历史。
* `card` 可选，字段和 `AiPhone.chat.sendCard({ card })` 同一套：`title`、`subtitle`、`body`、`sections`、`actions`、`accentColor`、`html`、`height` 等。
* `card` 字符串里可以使用占位符：`{{arg1}}`、`{{参数1}}`，以及按 `syntax` 命名的 `{{收餐人}}` 等；不写 `card` 时使用宿主默认卡片模板。

如果想在角色输出该指令后立刻同步数据或执行动作：

1. 在 `extensions.chat.directives` 里声明富媒体指令。
2. 在 `extensions.events` 里声明监听 `chat.message.created`。
3. 在 APP 代码里用 `AiPhone.on("chat.message.created", handler)` 监听新聊天消息。
4. handler 里判断消息是否来自角色、是否包含本 APP 的指令；匹配就由 APP 自己写数据库、发通知、更新卡片等。

希望 APP 没打开时也能监听，`extensions.events` 里要写 `background: true`。只需要用户点击卡片后再处理，就不需要事件监听。

#### extensions.chat.plusActions —— + 号面板入口

|presentation|效果|
|-|-|
|`panel`|聊天内底部面板（`panelHeight` 只对它生效）|
|`modal`|聊天内居中弹窗|
|`fullscreen` / `app`|打开完整 APP 页面|
|`none`|不显示 UI，只让 APP 代码执行|

用户点击后 `AiPhone.app.getLaunchContext()` 返回 `source="chat_plus_action"`、`sessionId`、`characterId`、`actionId`、`entry`、`data`、`directiveId`、`presentation` 等。完成后可调用 `AiPhone.chat.sendCard` / `writeHistory` / `requestReply` 写回当前聊天，或 `AiPhone.app.close()` 关闭面板。

常用字段：`id`、`label`、`entry`、`presentation`、`panelHeight`、`data`、`directiveId`。

#### extensions.tools —— 角色可调用的 APP 工具

声明角色可以调用的真实动作后，角色可以在聊天 APP 中调用该动作。每个工具要有 `name`、`description`、参数结构和 `handler` 名；APP 代码里用 `AiPhone.tools.handle(handlerName, fn)` 实现。APP 没打开时，宿主会临时启动隐藏运行环境执行 handler，默认最多保留 5 分钟，可写 `timeoutMs`，最高 30 分钟。

* `visibility: "private"` → 仅本 APP 内部生成可用。
* `visibility: "shared"` → 共享给普通聊天和其他 APP，进入“设置 → 聊天工具箱”。
* 工具是否真的对角色可见，由同场景 tags 的预设条目里的 `{{tools}}` 决定。

#### extensions.events —— 监听宿主事件

```json
"events": [{
  "event": "chat.message.created",
  "entry": "events",
  "background": true,
  "description": "监听聊天新消息；background 为 true 时，APP 未打开也会临时运行后台 handler。"
}]
```

导入器兼容 `["chat.message.created"]` 字符串简写，新包推荐对象写法。
