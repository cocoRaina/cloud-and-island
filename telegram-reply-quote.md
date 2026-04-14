# Telegram 引用消息功能

> 让 Claude 看到你在回复哪条消息，对话不再断线。

## 这是什么

在 Telegram 里，你可以长按一条消息然后"引用回复"。默认情况下，Claude Code 的 Telegram 插件**不会**把引用信息传递给 Claude——它只收到你发的文字，不知道你在回复哪条消息。

这意味着你引用了它之前说的某句话回复"这个不对"，它可能一脸懵：什么不对？

这篇教程教你怎么改一行代码，让 Claude 能看到完整的引用上下文。

---

## 效果

改完之后，当你在 Telegram 里引用回复一条消息，Claude 收到的消息会带上这些额外信息：

- `reply_to_message_id`：被引用消息的 ID
- `reply_to_text`：被引用消息的文字内容

在 Claude 的对话里，消息会显示成这样：

```
<channel source="telegram" chat_id="你的ID" message_id="102" 
  reply_to_message_id="99" reply_to_text="今天天气真好">
不是很好吧，在下雨
</channel>
```

Claude 一眼就能看到：你引用的是"今天天气真好"这条消息，然后说"不是很好吧"。上下文完整了。

---

## 怎么改

### 找到插件文件

Telegram 插件的代码在这个路径（注意版本号可能不同）：

```
~/.claude/plugins/cache/claude-plugins-official/telegram/<版本号>/server.ts
```

比如当前版本是 `0.0.5`，完整路径就是：

```
~/.claude/plugins/cache/claude-plugins-official/telegram/0.0.5/server.ts
```

### 找到消息通知代码

在 `server.ts` 里搜索 `notifications/claude/channel`，找到发送消息通知的那段代码。你会看到一个 `meta` 对象，里面有 `chat_id`、`message_id`、`user`、`ts` 这些字段。

大概长这样：

```typescript
mcp.notification({
  method: 'notifications/claude/channel',
  params: {
    content: text,
    meta: {
      chat_id,
      ...(msgId != null ? { message_id: String(msgId) } : {}),
      user: from.username ?? String(from.id),
      user_id: String(from.id),
      ts: new Date((ctx.message?.date ?? 0) * 1000).toISOString(),
      // ↑ 在 ts 这一行后面，加入引用消息的代码
      ...(imagePath ? { image_path: imagePath } : {}),
    },
  },
})
```

### 加入引用消息代码

在 `ts` 那一行后面，加入这三行：

```typescript
...(ctx.message?.reply_to_message ? {
  reply_to_message_id: String(ctx.message.reply_to_message.message_id),
  reply_to_text: ctx.message.reply_to_message.text ?? '(media)',
} : {}),
```

加完之后的完整 `meta` 部分：

```typescript
meta: {
  chat_id,
  ...(msgId != null ? { message_id: String(msgId) } : {}),
  user: from.username ?? String(from.id),
  user_id: String(from.id),
  ts: new Date((ctx.message?.date ?? 0) * 1000).toISOString(),
  ...(ctx.message?.reply_to_message ? {
    reply_to_message_id: String(ctx.message.reply_to_message.message_id),
    reply_to_text: ctx.message.reply_to_message.text ?? '(media)',
  } : {}),
  ...(imagePath ? { image_path: imagePath } : {}),
},
```

### 重启 Claude Code

改完代码后，需要重启 Claude Code 才能生效。

然后在 Telegram 里随便引用一条消息回复，看看 Claude 是不是能正确识别到引用的内容。

---

## 注意事项

- **插件更新会覆盖修改**：这个文件在插件缓存目录里，如果插件更新到新版本，你的修改会丢失，需要重新加一次。
- **只传文字引用**：如果被引用的消息是图片、语音等媒体，`reply_to_text` 会显示 `(media)`。
- **不需要改 CLAUDE.md**：Claude 天然能理解 `reply_to_message_id` 和 `reply_to_text` 这些字段的含义，不用额外解释。

---

## 原理

Telegram Bot API 的消息对象里有一个 `reply_to_message` 字段，包含了被引用消息的完整信息。Claude Code 的 Telegram 插件在收到消息后，会通过 MCP 的 notification 机制把消息传递给 Claude。我们做的就是把 `reply_to_message` 里的关键信息（消息 ID 和文字内容）一起传过去。

这样 Claude 在收到消息时，就能看到完整的对话上下文，不会出现"你在说啥"的情况。
