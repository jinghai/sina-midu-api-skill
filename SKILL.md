---
name: "sina-midu-api-skill"
description: "Use when integrating Sina Midu public-opinion subscription APIs, generating curl examples, or troubleshooting authorizeCode, accessToken, ticket, offset, and polling behavior."
---

# 新浪蜜度 API 通用技能

## Overview
本技能用于处理新浪蜜度舆情数据同步接口的通用接入、说明编写、联调排障与调用示例生成。

统一口径：

- API 地址：`https://api-open-wx-www.yqt365.com/dataapp`
- 接口基础路径：`https://api-open-wx-www.yqt365.com/dataapp/api`
- 试用账户数据保留 1 天
- 正式账户数据保留 3 天

## When To Use
- 用户要求接入新浪蜜度舆情订阅接口
- 用户要求生成该 API 的 curl 示例、调用手册或对接说明
- 用户反馈 `authorizeCode`、`accessToken`、`ticket`、`offset`、轮询拉取异常
- 用户要求解释返回字段、错误码、来源类型或 offset 续拉规则

## Quick Reference

| 项目 | 口径 |
| :--- | :--- |
| 授权码接口 | `GET /oauth2/authorize` |
| Token 接口 | `GET /oauth2/token` |
| 数据拉取接口 | `POST /data/kafka/scribe/poll` |
| 数据提交格式 | `application/x-www-form-urlencoded` |
| 成功特征 | 响应中包含 `{"data":` |
| 核心消费位点 | `offset` |
| 频率限制 | 45 次/分 |

## Required Inputs

| 参数 | 说明 |
| :--- | :--- |
| `appId` | 账号标识 |
| `appSecret` | 账号密钥 |
| `ticket` | 订阅方案标识 |
| IP 白名单 | 如果接口开启 IP 限制则必须配置 |

## Call Flow
按以下顺序执行：

1. 调用授权码接口获取 `authorizeCode`
2. 调用 token 接口获取 `accessToken`
3. 使用 `accessToken + ticket + offset` 持续拉取数据
4. 保存最后一条记录的 `offset + 1` 作为下次请求位点

## Curl Examples

### 1. 获取授权码

```bash
curl --request GET \
  'https://api-open-wx-www.yqt365.com/dataapp/api/oauth2/authorize?appId=YOUR_APP_ID&responseType=code&state=state'
```

### 2. 获取 accessToken

```bash
curl --request GET \
  'https://api-open-wx-www.yqt365.com/dataapp/api/oauth2/token?appId=YOUR_APP_ID&appSecret=YOUR_APP_SECRET&grantType=authorization_code&authorizeCode=YOUR_AUTHORIZE_CODE'
```

### 3. 拉取订阅数据

```bash
curl --request POST \
  'https://api-open-wx-www.yqt365.com/dataapp/api/data/kafka/scribe/poll' \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'accessToken=YOUR_ACCESS_TOKEN' \
  --data-urlencode 'ticket=YOUR_TICKET' \
  --data-urlencode 'offset=1'
```

## Offset Rules
- 初始 `offset` 由提供方给出，文档示例常写 `1`
- 每条记录都会返回一个新的 `offset`
- 下次请求必须使用“最后一条记录的 `offset + 1`”
- 必须持久化保存最新 `offset`
- 相同 `offset` 会导致重复拉取
- 超过保留期未消费的数据会被清理，需要重新向提供方申请新的 `offset`

## Response Notes
- 正常数据流通常不是统一外层成功码结构
- 多条记录之间通过换行符 `\n` 分隔
- 每条记录都是独立 JSON
- 常见关键字段：
  - 外层：`data`、`offset`
  - 命中信息：`matchInfo.ticket`、`matchInfo.name`、`matchInfo.info`
  - 内容信息：`author`、`title`、`text`、`publishTime`、`url`
  - 扩展信息：`contentExt`、`userExt`、`videoExt`

## Common Error Codes

| 错误码 | 含义 |
| :--- | :--- |
| `20001` | `appId` 错误 |
| `20002` | `ticket` 错误 |
| `20007` | 结果为空 |
| `30001` | 参数格式错误 |
| `30004` | `accessToken` 失效 |
| `30005` | 接口未授权 |
| `30006` | 请求超限 |
| `30009` | IP 未授权 |
| `9999` | 系统错误 |

## Common Mistakes
- 把数据拉取接口当成 JSON Body 接口
- 没有缓存 `accessToken`，频繁重复申请
- 没有持久化 `offset`
- 用旧 `offset` 持续重复拉取
- 把“无新数据”误判为接口故障
- 忽略试用账户 1 天、正式账户 3 天的保留期差异

## Output Standard
当用户请求文档或说明时，默认输出内容应覆盖：

- API 地址与基础路径
- 三步调用流程
- curl 示例
- offset 规则
- 数据保留期
- 成功判断方式
- 常见错误码与排查建议
