# otel-lgtm

Claude Code の OpenTelemetry（費用・token・所要時間）をローカルで受けて見るための、
[grafana/otel-lgtm](https://github.com/grafana/docker-otel-lgtm) の常駐構成です。
Grafana / Loki / Prometheus / Tempo / OTel Collector が 1 コンテナに入っています。

> grafana/otel-lgtm は開発・検証用のイメージです。本番の監視には使わないでください。

## 起動・停止

```bash
docker compose up -d     # 起動（restart: unless-stopped なので、Docker Desktop が動いていれば再起動後も自動で立ち上がる）
docker compose down      # 停止（データは volume lgtm-data に残る）
docker compose down -v   # 停止してデータも消す
```

- Grafana: <http://127.0.0.1:3333>（既定は admin / admin。変えるなら `.env` に `GF_SECURITY_ADMIN_PASSWORD=...`）
- OTLP: gRPC `127.0.0.1:4317` / HTTP `127.0.0.1:4318`
- どのポートも `127.0.0.1` にだけ bind しているので、LAN からは見えません。
- Docker Desktop を「ログイン時に起動」にしておくと、PC の再起動後も自動で受け始めます。

## Claude Code 側の設定

リポジトリの `.claude/settings.json` ではテレメトリを有効にできない仕様なので、
`~/.claude/settings.json` の `env` に入れます（新しく起動したセッションから効きます）。

```json
{
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "OTEL_METRICS_EXPORTER": "otlp",
    "OTEL_LOGS_EXPORTER": "otlp",
    "OTEL_EXPORTER_OTLP_PROTOCOL": "grpc",
    "OTEL_EXPORTER_OTLP_ENDPOINT": "http://127.0.0.1:4317",
    "OTEL_LOG_TOOL_DETAILS": "1",
    "OTEL_METRICS_INCLUDE_REPOSITORY": "1"
  }
}
```

- `OTEL_LOG_TOOL_DETAILS=1`: 独自のサブエージェント名を `agent_name` に出すのに要ります。ツールの引数（ファイルのパスやコマンド）も記録されますが、送り先はローカルだけです。
- `OTEL_METRICS_INCLUDE_REPOSITORY=1`: どのリポジトリでの作業かを `vcs_*` で見分けられるようにします。
- 外部の SaaS（Datadog など）に送る場合は、カスタムメトリクスの課金を抑えるため `OTEL_METRICS_INCLUDE_SESSION_ID=false` を検討してください（イベントからも `session_id` が消えます）。
- 一時的に止めたいときは、そのシェルで `CLAUDE_CODE_ENABLE_TELEMETRY=0 claude` で起動します。

項目の一覧は公式の [Monitoring usage](https://code.claude.com/docs/en/monitoring-usage) を参照してください。

## 見方（Grafana > Explore > Loki）

API リクエストごとのイベントは `event_name="api_request"` で、
`cost_usd` / `input_tokens` / `output_tokens` / `cache_read_tokens` / `cache_creation_tokens` /
`duration_ms` / `model` / `query_source`（main / subagent / sdk など）/ `agent_name` / `session_id` を持っています。

期間内の費用（定価換算の USD）をモデルと呼び出し元ごとに:

```logql
sum by (model, query_source, agent_name) (
  sum_over_time({service_name="claude-code"} | event_name="api_request" | unwrap cost_usd [$__range])
)
```

キャッシュの読み込み token をサブエージェント別に:

```logql
sum by (agent_name) (
  sum_over_time({service_name="claude-code"} | event_name="api_request" | unwrap cache_read_tokens [$__range])
)
```

中断（使用量の上限など）の件数:

```logql
sum by (status_code) (count_over_time({service_name="claude-code"} | event_name="api_error" [$__range]))
```

`$__range` は Grafana の時間範囲です。API で直接叩くときは `[1h]` のように書き換えます。
