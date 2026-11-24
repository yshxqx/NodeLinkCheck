# NodeLinkCheck 免 Sub-Store 补丁（Quantumult X）

> 基于 [I-am-R-E/Functional-Store-Hub](https://github.com/I-am-R-E/Functional-Store-Hub/tree/Master/NodeLinkCheck) 的 NodeLinkCheck 2.x 修改，原作者版权归其所有；本仓库仅改动 QX 取节点地址的方式。

原版逻辑：Sub-Store 的 `Operator.js` 把「节点名 → server/type」写进持久化 key `RebuildNodeLineJsonByRE`，面板脚本再读这份映射。

补丁改为：QX 下先用官方 API 直接问当前节点的服务器描述，问不到才回落到 Sub-Store 映射。

## 改动点

在 `Script/NodeLinkCheck.js` 的「入口 server」取值处插入：

```js
$configuration.sendMessage({ action: 'get_server_description', content: tag })
```

返回形如：

```
vmess=entry.example.com:443,udp-relay=false, tag=香港中转①
```

解析出 `type=vmess`、`server=entry.example.com`（自动去掉 `:端口`，兼容 `[IPv6]:443`），并把结果同时写入 `nodeInfo`，供后面 HTML 面板显示协议类型。

优先级：

1. `$configuration.get_server_description`（QX 原生，无需任何订阅处理）
2. Sub-Store 映射（原版行为，兼容旧用法）
3. 都拿不到 → 面板显示「未取到节点地址：请升级 QX 至 v1.0.30-build691+…」

## 兼容性

- `get_server_description` 需要 **QX v1.0.30-build691 / iOS 13+**，且**只对 event-interaction 脚本开放**——本脚本正是该类型，所以正好可用。
- Surge / Loon 分支未改动，仍走各自原有逻辑。
- 原有的 Sub-Store 用法完全不受影响（有映射时仍以映射为准）。

## 安装

**方式 A：本地脚本**

1. 把 `NodeLinkCheck.qx.js` 存到 `我的 iPhone → Quantumult X → Scripts`
2. `[task_local]` 加：

```
event-interaction NodeLinkCheck.qx.js, tag=Env代理链路检测, img-url=https://raw.githubusercontent.com/Koolson/Qure/master/IconSet/Color/Stack.png, enabled=true
```

**方式 B：自建 raw 链接**

推到自己的 GitHub 仓库，用 raw URL 替换配置文件里的脚本地址。

触发方式不变：在策略页对某个节点触发 UIAction，QX 会把节点 tag 作为 `$environment.params` 传入。

## 验证记录（node + vm 模拟 QX 环境）

| 场景 | 结果 |
|---|---|
| 无映射 + `get_server_description` 返回 `entry.example.com` | 正常：入口上海 CN / 出口 US → 「中转代理」 |
| 节点名含 `IEPL` | 正常：「IEPL专线」 |
| 无 `$configuration`（模拟旧版 QX） | 正常降级：面板提示「未取到节点地址…」 |
| 仅有 Sub-Store 映射、无 `$configuration` | 正常：仍走映射，结果「中转代理」 |

请求序列：`ip-api.com/json/?lang=zh-CN`（出口）→ `ip-api.com/json/entry.example.com?lang=zh-CN`（入口）。

## 仍然存在的限制

- 免费 ip-api：HTTP、45 次/分钟；节点 server 是 CDN 域名时“入口”地理信息无意义。
- 只能比对「节点配置里的 server」与「实际出口 IP」，测不出真正的多跳链路；专线类型仍是节点名关键词启发式。
- `get_server_description` 取的是配置里的地址，若节点名与实际 server 在客户端里被改写，可能与预期不符。
