# webhook-bridge

Alertmanager → 飞书 Webhook 转发桥。接收 Alertmanager 告警，渲染为飞书卡片消息（lark_md 富文本格式）并发送到飞书群。

## 特性

- 飞书卡片消息，支持粗体、颜色、超链接等富文本样式
- 告警（firing）红色卡片 / 恢复（resolved）绿色卡片
- 可配置模板，修改 `template.json` 即时生效，无需重启
- 支持飞书加签验证
- 快捷链接（Grafana / Alertmanager / Prometheus 等）
- 健康检查端点 `/health`
- 基于 gunicorn 生产级部署

## 快速开始

### 1. 创建配置

```bash
cp .env.example .env
```

编辑 `.env`，填入飞书 Webhook 地址：

```dotenv
FEISHU_WEBHOOK_URL=https://open.feishu.cn/open-apis/bot/v2/hook/xxxxxxxx
```

### 2. 启动服务

```bash
docker compose up -d
```

### 3. 配置 Alertmanager

在 Alertmanager 配置中添加 webhook receiver：

```yaml
receivers:
  - name: 'feishu'
    webhook_configs:
      - url: 'http://webhook-bridge:5000/webhook'
        send_resolved: true
```

## 模板配置

编辑 `template.json` 自定义告警卡片样式，修改后下次告警自动生效。

```json
{
  "firing": {
    "header_color": "red",
    "header_title": "⚠ {{project_name}} 环境异常告警",
    "fields": [
      "**告警名称：** <font color='red'>{{summary}}</font>",
      "**告警类型：** {{status}}",
      "**告警级别：** {{severity}}",
      "**开始时间：** {{starts_at}}",
      "**结束时间：** {{ends_at}}",
      "**故障位置：** {{source}}",
      "**故障描述：** <font color='red'>{{description}}</font>"
    ]
  },
  "resolved": {
    "header_color": "green",
    "header_title": "✅ {{project_name}} 环境恢复信息",
    "fields": [
      "**告警名称：** <font color='green'>{{summary}}</font>",
      "..."
    ]
  },
  "project_name": "监控系统",
  "links": {
    "grafana": { "text": "grafana", "url": "http://your-grafana:3000" },
    "alertmanager": { "text": "alertmanager", "url": "http://your-alertmanager:9093" },
    "prometheus": { "text": "prometheus", "url": "http://your-prometheus:9090" }
  },
  "note": "webhook-bridge"
}
```

### 模板变量

| 变量 | 说明 |
|------|------|
| `{{project_name}}` | 环境/项目名称 |
| `{{alertname}}` | 告警规则名称（英文） |
| `{{summary}}` | 告警摘要（中文，回退到 alertname） |
| `{{status}}` | 告警状态（firing / resolved） |
| `{{severity}}` | 告警级别 |
| `{{source}}` | 告警来源（instance 或容器名） |
| `{{description}}` | 告警描述 |
| `{{starts_at}}` | 开始时间（已转为本地时区） |
| `{{ends_at}}` | 结束时间 |

### 卡片颜色

`header_color` 可选值：`blue`、`red`、`orange`、`green`、`purple`、`indigo`、`grey`

### 快捷链接

`links` 中配置了 `url` 的会渲染为蓝色可点击超链接，未配置的显示为粗体文本。

## 环境变量

| 变量 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `FEISHU_WEBHOOK_URL` | 是 | - | 飞书机器人 Webhook 地址 |
| `FEISHU_SECRET` | 否 | - | 飞书加签密钥 |
| `TZ_OFFSET` | 否 | `8` | 时区偏移（小时） |
| `WEBHOOK_BRIDGE_PORT` | 否 | `5000` | 服务端口 |

## 本地构建

如需本地构建镜像，编辑 `docker-compose.yml`，注释 `image` 行并取消 `build` 注释：

```yaml
services:
  webhook-bridge:
    # image: iruiiii/webhook-bridge:latest
    build: .
    image: webhook-bridge:latest
```

然后：

```bash
docker compose up -d --build
```

## API

| 端点 | 方法 | 说明 |
|------|------|------|
| `/webhook` | POST | 接收 Alertmanager 告警 payload |
| `/health` | GET | 健康检查 |

## License

MIT
