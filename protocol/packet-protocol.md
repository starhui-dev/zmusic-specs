# ZMusic 通信包协议

## 文档状态

| 字段 | 值 |
|------|----|
| 状态 | 草案 |
| 业务协议版本 | 1 |
| 通道 | `zmusic:packet` |
| 适用范围 | ZMusic 服务端实现与客户端 Mod |
| 兼容策略 | 不兼容旧版 ZMusic / AudioBuffer 协议 |

本文档定义 ZMusic 服务端实现与客户端 Mod 之间的通信包协议。协议仍处于草案阶段，字段命名、错误码和状态机可能在首个稳定版本前调整。

## 目标

定义 ZMusic 服务端实现与客户端 Mod 之间的全新通信协议，用于音乐播放、停止、状态同步和能力协商。

本协议不兼容旧版 ZMusic / AudioBuffer 协议，也不为旧版通道、字段或消息格式保留适配逻辑。

## 设计原则

- 业务协议版本显式声明，后续升级可协商。
- 服务端负责搜索、鉴权、播放决策和状态管理。
- 客户端负责实际音频播放、停止和本地播放状态回报。
- Payload 使用 JSON，外层使用简单二进制帧，便于跨 Bukkit、BungeeCord、Velocity 和未来服务端 Mod 实现传输。
- 单个通信包只承载一个完整消息，不做分片。
- 第一阶段只设计单曲播放、歌词 URL 下发和当前歌词行上报，不设计队列、歌单同步或通过通信包传输整份歌词文件。

## 通道

第一阶段统一使用一个 Minecraft 自定义负载/插件消息通道承载 ZMusic 通信包：

```text
zmusic:packet
```

要求：

- 通信包协议本身不绑定服务端实现形态；Bukkit、BungeeCord、Velocity 插件和未来服务端 Mod 都应复用同一帧格式与 JSON envelope。
- 该通道是 ZMusic 客户端与服务端之间的统一业务通信通道。
- 通道名表示该通道承载 ZMusic 通信包，具体业务语义由 JSON 消息体中的 `type` 字段决定。
- Bukkit、BungeeCord、Velocity 启动时注册该通道。
- 客户端 Mod 登录服务器后也注册并监听该通道。
- 服务端所有业务消息都走该通道。
- 如后续需要调试或大数据传输，再新增通道，不在第一阶段引入。

## 传输路径

### Bukkit 单端

```text
ZMusic Bukkit Plugin -> Minecraft Plugin Message -> Client Mod
Client Mod -> Minecraft Plugin Message -> ZMusic Bukkit Plugin
```

Bukkit 插件可直接通过在线玩家发送消息到客户端。

### BungeeCord / Velocity 代理端

```text
ZMusic Proxy Plugin -> Player Current Backend Server -> Client Mod
Client Mod -> Backend Server -> ZMusic Proxy Plugin
```

代理端无法直接绕过玩家所在后端服务器发送客户端插件消息，因此发送目标是玩家当前连接的后端服务器，由 Minecraft 插件消息链路送达客户端。

### 服务端 Mod

```text
ZMusic Server Mod -> Minecraft Custom Payload -> Client Mod
Client Mod -> Minecraft Custom Payload -> ZMusic Server Mod
```

服务端 Mod 使用对应加载器提供的自定义负载或网络通道 API 传输 ZMusic 通信包。协议帧格式和 JSON envelope 与插件实现保持一致。

协议层不区分 Bukkit / BungeeCord / Velocity 或服务端 Mod，平台差异由传输层封装。

## 帧格式

通信包 payload 使用二进制帧：

```text
+------------+-------------+----------------+
| magic      | version     | json_payload   |
| 4 bytes    | 1 byte      | remaining      |
+------------+-------------+----------------+
```

字段说明：

| 字段 | 类型 | 说明 |
|------|------|------|
| `magic` | ASCII bytes | 固定为 `ZMPK`，表示 ZMusic Packet |
| `version` | unsigned byte | 通信包帧版本，第一版为 `1` |
| `json_payload` | UTF-8 JSON bytes | 消息体 |

说明：

- 不使用 Java `DataOutputStream.writeUTF`，避免 65535 字节限制和跨语言细节问题。
- 帧 `version` 只表示二进制帧格式版本；JSON 中的 `protocolVersion` 表示业务协议版本。
- JSON 字符串必须使用 UTF-8。
- 单个完整通信包最大不超过 32760 字节，包含 `magic`、`version` 和 `json_payload`。
- JSON payload 最大不超过 32755 字节，即完整通信包上限扣除 5 字节帧头。
- 超过大小限制时直接丢弃并记录 debug 日志。

## JSON 消息通用结构

所有 JSON payload 使用统一 envelope：

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "type": "server.play",
  "timestamp": 1730000000000,
  "data": {}
}
```

字段说明：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 消息 ID，用于日志、错误关联和去重 |
| `type` | string | 是 | 消息类型 |
| `timestamp` | number | 是 | 发送方 Unix 毫秒时间戳 |
| `data` | object | 是 | 业务数据 |

`timestamp` 只用于日志、排序参考和调试。接收方不得仅凭 `timestamp` 做鉴权或拒绝消息；如果需要业务过期判断，应使用明确业务字段，例如 `audio.expiresAt`。

消息 ID：

- 协议内生成的 ID 必须使用 UUID 字符串，包括 `id`、`requestId`、`targetRequestId`、`playerClientId` 和 `replyTo` 指向的消息 ID。
- UUID 必须使用标准 36 字符文本格式，例如 `550e8400-e29b-41d4-a716-446655440000`。
- 第三方外部 ID 不受该规则约束，例如 `song.id`。
- 同一发送方短时间内不得重复。
- 错误响应应通过 `replyTo` 带上原始消息 ID。

## 消息方向

消息类型按方向分组：

| 前缀 | 方向 | 说明 |
|------|------|------|
| `client.*` | 客户端 -> 服务端 | 客户端握手、状态、错误 |
| `server.*` | 服务端 -> 客户端 | 播放、停止、配置 |

## 版本协商

### `client.hello`

客户端连接到服务器且 ZMusic 通信通道可用后发送。客户端重连、切换服务器或通信通道重新注册后，应重新发送 `client.hello`。

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440001",
  "type": "client.hello",
  "timestamp": 1730000000000,
  "data": {
    "protocolVersion": 1,
    "modVersion": "4.0.0-dev",
    "minecraftVersion": "1.21.1",
    "loader": "fabric",
    "playerClientId": "550e8400-e29b-41d4-a716-446655443001",
    "capabilities": [
      "play.url",
      "stop",
      "status",
      "progress",
      "lyrics.url"
    ]
  }
}
```

服务端行为：

- 记录玩家支持的业务协议版本和能力。
- 记录 `playerClientId`，用于日志、诊断、重连识别和重复握手判断。
- 业务协议版本不支持时返回 `server.unsupported_protocol`。
- 支持时返回 `server.hello`。

`playerClientId` 由客户端生成并持久化，必须使用标准 UUID 字符串。同一个客户端安装实例应尽量保持稳定；重装 Mod、清理配置或首次启动时可以重新生成。服务端不得将 `playerClientId` 当作安全身份凭据。

### `server.hello`

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440002",
  "type": "server.hello",
  "timestamp": 1730000000000,
  "data": {
    "protocolVersion": 1,
    "serverVersion": "4.0.0-dev",
    "platform": "bukkit",
    "capabilities": [
      "play.url",
      "stop",
      "status",
      "progress",
      "lyrics.url"
    ],
    "limits": {
      "maxPacketBytes": 32760,
      "maxJsonBytes": 32755,
      "maxLyricsBytes": 262144
    }
  }
}
```

`platform` 可选值：

| 值 | 说明 |
|----|------|
| `bukkit` | Bukkit / Spigot / Paper / Folia 单端插件 |
| `bungee` | BungeeCord 代理插件 |
| `velocity` | Velocity 代理插件 |
| `fabric_server` | Fabric 服务端 Mod |
| `forge_server` | Forge 服务端 Mod |
| `neoforge_server` | NeoForge 服务端 Mod |

能力说明：

| 能力 | 说明 |
|------|------|
| `play.url` | 支持通过 URL 播放音频 |
| `stop` | 支持停止当前 ZMusic 播放 |
| `status` | 支持上报播放状态变化 |
| `progress` | 支持上报播放进度，并在 `client.progress` 中携带当前歌词状态 |
| `lyrics.url` | 支持通过服务端下发的 URL 拉取歌词 |

`limits.maxLyricsBytes` 表示客户端从 `lyrics.url` 或 `lyrics.translationUrl` 拉取歌词文件时建议接受的最大字节数，不表示通信包内字段大小。

## 播放

### `server.play`

服务端请求客户端播放一首歌曲：

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440003",
  "type": "server.play",
  "timestamp": 1730000000000,
  "data": {
    "requestId": "550e8400-e29b-41d4-a716-446655441001",
    "mode": "replace",
    "song": {
      "id": "1859245776",
      "source": "netease",
      "title": "稻香",
      "artists": ["周杰伦"],
      "album": "魔杰座",
      "coverUrl": "https://example.com/cover.jpg",
      "durationMillis": 223000
    },
    "audio": {
      "type": "url",
      "url": "https://example.com/audio.mp3",
      "expiresAt": 1730003600000,
      "headers": {}
    },
    "lyrics": {
      "type": "url",
      "format": "lrc",
      "url": "https://example.com/lyrics.lrc",
      "translationUrl": "https://example.com/lyrics-translated.lrc",
      "sizeBytes": 12345,
      "translationSizeBytes": 12345
    }
  }
}
```

字段说明：

| 字段 | 类型 | 说明 |
|------|------|------|
| `requestId` | string | 本次播放请求 ID，必须使用标准 UUID 字符串 |
| `mode` | string | 第一阶段固定 `replace`，表示替换当前播放 |
| `song.id` | string | 音源内歌曲 ID，可选 |
| `song.source` | string | 音源标识，第一阶段至少支持 `netease` 和 `custom` |
| `song.coverUrl` | string | 专辑封面 URL，可选 |
| `audio.type` | string | 第一阶段固定 `url` |
| `audio.url` | string | 客户端直接拉取的音频 URL |
| `audio.expiresAt` | number | 音频 URL 过期时间，Unix 毫秒时间戳，可选 |
| `audio.headers` | object | 可选请求头，第一阶段默认空对象 |
| `lyrics` | object/null | 歌词资源，可选；无歌词时可省略或传 `null` |
| `lyrics.type` | string | 第一阶段固定 `url` |
| `lyrics.format` | string | 歌词格式，第一阶段为 `lrc` |
| `lyrics.url` | string | 原文歌词 URL |
| `lyrics.translationUrl` | string | 翻译歌词 URL，可选 |
| `lyrics.sizeBytes` | number | 原文歌词字节数，可选 |
| `lyrics.translationSizeBytes` | number | 翻译歌词字节数，可选 |

音源标识：

| 值 | 说明 |
|----|------|
| `netease` | 网易云音乐 |
| `custom` | 自定义或直链音源 |

说明：

- `netease` 音源建议提供 `song.id`。
- `custom` 音源可以省略 `song.id`。
- `song.id` 是第三方音源 ID，保持字符串，不做 UUID 限制。
- 客户端不得假设 `song.id` 的格式、长度或数值类型。
- 服务端应限制 `song.id` 长度，建议不超过 128 字符。
- 播放请求和状态关联以 `requestId` 为准，不依赖 `song.id`。

音频请求头规则：

- `audio.headers` 只用于客户端拉取 `audio.url` 时附加 HTTP 请求头。
- 客户端可以选择忽略 `audio.headers`。
- 客户端必须拒绝危险请求头，包括 `Host`、`Content-Length`、`Transfer-Encoding`、`Connection`、`Cookie` 和 `Authorization`。
- 客户端应限制请求头名称和值的长度。
- 服务端不应通过 `audio.headers` 下发用户隐私凭据或长期有效 token。

音频 URL 过期规则：

- 如果存在 `audio.expiresAt`，客户端收到 `server.play` 时可用本地时间判断 URL 是否已过期。
- 如果客户端判断 URL 已过期，应拒绝播放并发送 `client.error`，其中 `phase` 为 `audio`，`code` 为 `audio_url_expired`，`retryable` 为 `true`。
- 播放过程中 URL 过期不影响已经建立的播放流。
- 如果播放过程中因重新请求音频 URL 失败，应发送 `audio_url_expired` 或 `audio_load_failed`。
- 客户端本地时间可能不准确，`audio.expiresAt` 只作为提前拒绝和调试依据；最终以 HTTP 请求结果为准。

歌词字段规则：

- `lyrics` 可省略或为 `null`，表示没有歌词资源。
- 当 `lyrics.type` 为 `url` 时，`lyrics.url` 和 `lyrics.format` 必填。
- 第一阶段 `lyrics.format` 固定为 `lrc`。
- `lyrics.translationUrl` 是原文歌词的附加翻译资源，可选。
- 不支持只传 `lyrics.translationUrl` 而不传 `lyrics.url`。
- `lyrics.sizeBytes` 和 `lyrics.translationSizeBytes` 是可选大小提示。
- 如果 `lyrics.sizeBytes` 或 `lyrics.translationSizeBytes` 大于 `server.hello.limits.maxLyricsBytes`，客户端可以拒绝拉取对应歌词文件。
- 如果实际下载大小超过 `server.hello.limits.maxLyricsBytes`，客户端应停止读取并将对应歌词标记为失败。
- 歌词加载失败不影响音频播放。
- 原文歌词加载失败时，客户端应发送 `client.error`，其中 `phase` 为 `lyrics`，`code` 为 `lyrics_load_failed`。
- 翻译歌词加载失败但原文歌词可用时，`client.progress.lyrics.state` 仍为 `ready`，并省略 `translation`。
- 翻译歌词加载失败时，客户端可以发送 `client.error`，其中 `phase` 为 `lyrics`，`code` 为 `lyrics_translation_load_failed`。

客户端行为：

- 收到后停止当前 ZMusic 播放。
- 如果当前已有 ZMusic 播放，应先对旧 `requestId` 发送 `client.status`，其中 `state` 为 `stopped`，`reason` 为 `server_replace`。
- 通过基础校验后，客户端可以发送 `client.status`，其中 `state` 为 `loading`，`reason` 为 `loading_started`。
- 拉取 `audio.url`。
- 如果存在 `lyrics.url`，客户端自行拉取歌词；如果存在 `lyrics.translationUrl`，客户端自行拉取翻译歌词。
- 开始播放后发送 `client.status`，状态为 `playing`。
- 播放中每秒发送 `client.progress`。
- 如果无法播放，必须发送 `client.status`，状态为 `failed`；如果有详细错误，再发送 `client.error`。

服务端行为：

- 发送前确认玩家已完成 `client.hello`。
- 如果玩家未握手，提示需要安装或启用客户端 Mod。
- 保存 `requestId` 与玩家当前播放状态。
- 播放音量由客户端本地控制，服务端第一阶段不下发音量。

## 停止

### `server.stop`

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440004",
  "type": "server.stop",
  "timestamp": 1730000000000,
  "data": {
    "requestId": "550e8400-e29b-41d4-a716-446655442001",
    "targetRequestId": "550e8400-e29b-41d4-a716-446655441001",
    "reason": "command"
  }
}
```

字段说明：

| 字段 | 类型 | 说明 |
|------|------|------|
| `requestId` | string | 本次停止请求 ID，必须使用标准 UUID 字符串 |
| `targetRequestId` | string | 要停止的播放请求 ID，必须使用标准 UUID 字符串，可选；省略时表示停止当前 ZMusic 播放 |
| `reason` | string | 停止原因 |

`reason` 可选值：

`server.stop.reason` 描述服务端发出停止命令的原因，不与 `client.status.reason` 共用枚举。

| 值 | 说明 |
|----|------|
| `command` | 玩家或管理员执行停止命令 |
| `replace` | 新播放替换旧播放 |
| `disconnect` | 玩家断开或服务端清理 |
| `server_shutdown` | 服务端实现或服务器关闭 |
| `permission_revoked` | 玩家播放权限被撤销 |

客户端行为：

- 如果 `targetRequestId` 省略，停止当前 ZMusic 播放。
- 如果 `targetRequestId` 等于当前播放请求 ID，停止当前 ZMusic 播放。
- 如果 `targetRequestId` 不等于当前播放请求 ID，忽略该停止请求，不改变当前播放。
- 忽略过期停止请求时，客户端可以发送 `client.error`，其中 `phase` 为 `playback`，`code` 为 `stale_request`，`retryable` 为 `false`。
- 实际执行停止时，清理本地播放状态。
- 实际执行停止后，发送 `client.status`，状态为 `stopped`。

## 状态上报

### `client.status`

客户端上报播放状态变化。该消息不承担每秒进度同步；播放进度和当前歌词由 `client.progress` 上报。

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440005",
  "type": "client.status",
  "timestamp": 1730000000000,
  "data": {
    "requestId": "550e8400-e29b-41d4-a716-446655441001",
    "state": "playing",
    "reason": "play_started",
    "durationMillis": 223000
  }
}
```

字段说明：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `requestId` | string | 是 | 关联播放请求 ID，必须使用标准 UUID 字符串 |
| `state` | string | 是 | 播放状态 |
| `reason` | string | 否 | 状态变化原因 |
| `durationMillis` | number | 是 | 音频总时长；未知时为 `-1` |

`state` 可选值：

| 值 | 说明 |
|----|------|
| `loading` | 正在加载音频 |
| `playing` | 正在播放 |
| `stopped` | 已停止 |
| `ended` | 自然播放结束 |
| `failed` | 播放失败 |

`reason` 可选值：

`client.status.reason` 描述客户端播放状态变化的原因，不与 `server.stop.reason` 共用枚举。

| 值 | 说明 |
|----|------|
| `loading_started` | 开始加载音频 |
| `play_started` | 播放已开始 |
| `server_stop` | 服务端要求停止 |
| `server_replace` | 被服务端新的播放请求替换 |
| `natural_end` | 自然播放结束 |
| `audio_error` | 音频加载或请求错误 |
| `decode_error` | 音频解码错误 |
| `player_error` | 播放器内部错误 |

上报时机：

- 开始加载时可上报 `loading`。
- 成功开始播放后必须上报 `playing`。
- 播放结束后必须上报 `ended`。
- 停止后必须上报 `stopped`。
- 播放失败时必须发送 `client.status`，状态为 `failed`；如果有详细错误，再发送 `client.error`。
- `stopped`、`ended`、`failed` 等终态也必须携带 `durationMillis`；已知时填真实总时长，未知时填 `-1`。

### `client.progress`

客户端上报播放进度和当前歌词。播放中目标频率为 1 Hz，允许轻微计时抖动；停止、结束或失败后不要求持续上报。

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440006",
  "type": "client.progress",
  "timestamp": 1730000000000,
  "data": {
    "requestId": "550e8400-e29b-41d4-a716-446655441001",
    "positionMillis": 12000,
    "durationMillis": 223000,
    "lyrics": {
      "state": "ready",
      "lineIndex": 3,
      "text": "还记得你说家是唯一的城堡",
      "translation": "..."
    }
  }
}
```

字段说明：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `requestId` | string | 是 | 关联播放请求 ID，必须使用标准 UUID 字符串 |
| `positionMillis` | number | 是 | 当前播放位置 |
| `durationMillis` | number | 是 | 音频总时长；未知时为 `-1` |
| `lyrics` | object | 是 | 当前歌词状态 |

进度字段规则：

- `positionMillis >= 0`。
- 正常情况下 `positionMillis <= durationMillis`。
- 当 `durationMillis = -1` 时，只要求 `positionMillis >= 0`。
- 服务端收到超出范围的进度时，应在展示层 clamp 到合理范围，不应仅因此踢出玩家。

`lyrics.state` 可选值：

| 值 | 说明 |
|----|------|
| `none` | 没有歌词 |
| `loading` | 歌词加载中 |
| `ready` | 歌词可用 |
| `failed` | 歌词加载失败 |

说明：

- 没有歌词时，`lyrics.state` 为 `none`，只需要携带 `state`。
- 歌词加载中时，`lyrics.state` 为 `loading`，只需要携带 `state`。
- 歌词加载失败时，`lyrics.state` 为 `failed`，只需要携带 `state`，详细错误仍通过 `client.error` 上报。
- 歌词可用时，`lyrics.state` 为 `ready`，`lineIndex` 和 `text` 必填，`translation` 可选。
- `lineIndex` 从 `0` 开始。
- 歌词已加载但当前播放位置未命中任何歌词行时，`lyrics.state` 仍为 `ready`，`lineIndex` 为 `-1`，`text` 为空字符串。
- 当 `lineIndex` 为 `-1` 时，`translation` 应省略或为空字符串。
- 服务端只有在 `durationMillis > 0` 时计算播放百分比。
- 服务端可用该消息更新占位符、BossBar、计分板或其他展示层。

## 错误

### `client.error`

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440007",
  "type": "client.error",
  "timestamp": 1730000000000,
  "data": {
    "requestId": "550e8400-e29b-41d4-a716-446655441001",
    "phase": "audio",
    "code": "audio_load_failed",
    "message": "Failed to load audio URL",
    "retryable": true,
    "details": {
      "httpStatus": 404
    }
  }
}
```

`phase` 可选值：

| 值 | 说明 |
|----|------|
| `handshake` | 握手阶段 |
| `audio` | 音频加载或解码阶段 |
| `lyrics` | 歌词加载阶段 |
| `playback` | 播放执行阶段 |
| `protocol` | 协议解析或校验阶段 |

客户端错误码：

| code | 说明 |
|------|------|
| `unsupported_protocol` | 业务协议版本不支持 |
| `unsupported_message` | 消息类型不支持 |
| `invalid_payload` | payload 无法解析或缺字段 |
| `audio_url_expired` | 音频 URL 已过期 |
| `audio_load_failed` | 音频加载失败 |
| `audio_decode_failed` | 音频解码失败 |
| `lyrics_load_failed` | 歌词加载失败 |
| `lyrics_translation_load_failed` | 翻译歌词加载失败 |
| `playback_failed` | 播放器启动失败 |
| `stale_request` | 请求已过期或不匹配当前播放 |

### `server.error`

服务端也可以给客户端返回错误：

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440008",
  "type": "server.error",
  "timestamp": 1730000000000,
  "data": {
    "replyTo": "550e8400-e29b-41d4-a716-446655440001",
    "phase": "handshake",
    "code": "unsupported_protocol",
    "message": "Business protocol version is not supported",
    "retryable": false,
    "details": {
      "supportedProtocols": [1]
    }
  }
}
```

服务端错误码：

| code | 说明 |
|------|------|
| `unsupported_protocol` | 业务协议版本不支持 |
| `unsupported_message` | 消息类型不支持 |
| `invalid_payload` | payload 无法解析 |
| `not_ready` | 玩家尚未完成握手 |
| `rate_limited` | 消息频率过高 |
| `permission_denied` | 权限不足 |
| `internal_error` | 服务端内部错误 |

`server.error.data.replyTo` 规则：

- 当错误由某条客户端消息触发时，`replyTo` 必填，值为原始消息 ID。
- 当服务端主动通知错误且没有明确原始消息时，可以省略 `replyTo`。
- `client.error` 主要通过 `requestId` 关联播放请求，不使用 `replyTo`。

## 消息确认策略

第一阶段不定义独立消息确认机制。

- `client.hello` 必须有 `server.hello` 或 `server.error`。
- `server.play` 的实际结果通过 `client.status` 或 `client.error` 表达。
- `server.stop` 的实际结果通过 `client.status` 或 `client.error` 表达。
- `client.progress` 是高频状态流，不做消息级确认。
- 基础校验失败或业务执行失败统一使用 `client.error` 或 `server.error` 表达。

## 服务端状态机

服务端按玩家维护状态：

```text
UNKNOWN
  -> READY       收到 client.hello 且协议支持
  -> UNSUPPORTED 收到 client.hello 但协议不支持

READY
  -> LOADING     发送 server.play
  -> READY       发送 server.stop 或无播放

LOADING
  -> PLAYING     收到 client.status playing
  -> READY       收到 client.status failed/stopped 或 client.error

PLAYING
  -> READY       收到 client.status ended/stopped/failed
  -> PLAYING     收到 client.progress
  -> LOADING     发送新的 server.play
```

说明：

- 玩家断线时清理状态。
- 服务端实现关闭或禁用时，对在线且已握手玩家发送 `server.stop`，随后清理状态。
- 如果客户端长时间没有返回状态，服务端可保留状态，但命令反馈应提示“已发送播放请求”，不要声称“已开始播放”。

## 客户端状态机

```text
INIT
  -> READY       发送 client.hello 并收到 server.hello
  -> UNSUPPORTED 收到 server.error unsupported_protocol

READY
  -> LOADING     收到 server.play

LOADING
  -> PLAYING     音频加载并开始播放
  -> READY       播放失败

PLAYING
  -> READY       播放结束或收到 server.stop
  -> LOADING     收到新的 server.play
```

## 安全与校验

服务端发送前校验：

- 玩家在线。
- 玩家已完成握手。
- `audio.url` 不是空字符串。
- 完整通信包和 JSON payload 不超过协议限制。
- 服务端应避免发送异常巨大的字符串字段。

客户端接收后校验：

- `magic` 和 `version` 正确。
- JSON 可解析。
- `type` 支持。
- 必填字段存在。
- 音频 URL 协议只允许 `http` 或 `https`。
- 歌词 URL 协议只允许 `http` 或 `https`。
- 客户端应在解析后对字符串字段做合理保护，避免内存或 UI 问题。
- 不执行服务端传来的任意脚本、命令或本地文件路径。

## 限流

服务端应对客户端消息做简单限流：

- `client.hello`：每个玩家 10 秒内最多 3 次。
- `client.status`：每个玩家每秒最多 5 次。
- `client.progress`：每个玩家每秒最多 2 次。
- `client.error`：每个玩家每秒最多 5 次。

超过限制：

- debug 日志记录。
- 丢弃超额消息。
- `client.progress` 超频时只丢弃多余消息，不发送 `server.error`。
- 不踢出玩家，除非后续出现明显滥用。

## Kotlin 编码建议

协议相关代码建议放在：

```text
zmusic-core/src/main/kotlin/me/zhenxin/zmusic/protocol
├── ZMusicProtocol.kt
├── ProtocolCodec.kt
├── ProtocolMessage.kt
├── ProtocolMessageType.kt
└── ProtocolException.kt
```

建议常量：

```kotlin
object ZMusicProtocol {
    const val CHANNEL = "zmusic:packet"
    const val VERSION = 1
    val MAGIC = byteArrayOf('Z'.code.toByte(), 'M'.code.toByte(), 'P'.code.toByte(), 'K'.code.toByte())
    const val MAX_PACKET_BYTES = 32760
    const val MAX_JSON_BYTES = MAX_PACKET_BYTES - 5
}
```

编码流程：

1. 将 `ProtocolMessage` 序列化为 JSON UTF-8 字节。
2. 检查 JSON 字节长度不超过 `MAX_JSON_BYTES`。
3. 写入 `MAGIC`。
4. 写入 `VERSION`。
5. 写入 JSON bytes。

解码流程：

1. 检查总长度至少 5 字节。
2. 校验 `MAGIC`。
3. 校验 `VERSION`。
4. 剩余字节按 UTF-8 解析 JSON。
5. 校验 envelope 必填字段。

## 第一阶段实现范围

第一阶段只实现这些消息：

- `client.hello`
- `server.hello`
- `server.play`
- `server.stop`
- `client.status`
- `client.progress`
- `client.error`
- `server.error`

不实现：

- 通过通信包传输整份歌词文件。
- 队列同步。
- 歌单同步。
- 全服播放。
- 客户端音量 UI 配置同步。

## 验收标准

- Bukkit 单端能完成 `client.hello` / `server.hello`。
- `/zmusic play <keyword>` 能发送 `server.play` 给已握手客户端。
- 客户端开始播放后能回 `client.status playing`。
- 客户端播放中能每秒回 `client.progress`，包含当前进度和当前歌词行。
- `/zmusic stop` 能发送 `server.stop`，客户端回 `client.status stopped`。
- 未安装客户端 Mod 的玩家执行播放命令时，服务端给出明确提示。
- 协议编码/解码有单元测试覆盖：
  - 正常消息。
  - magic 错误。
  - version 错误。
  - JSON 缺字段。
  - 通信包或 JSON payload 超限。
