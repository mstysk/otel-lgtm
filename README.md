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

## 見方（Grafana > Explore > Prometheus）

メトリクスは `claude_code_cost_usage_USD_total` / `claude_code_token_usage_tokens_total` /
`claude_code_active_time_seconds_total` / `claude_code_lines_of_code_count_total` などの名前で入ります。

期間内の費用をモデルごとに:

```promql
sum by (model) (increase(claude_code_cost_usage_USD_total[$__range]))
```

### メトリクスが no data になるとき

Claude Code はメトリクスを delta temporality で送ります。
Prometheus の OTLP receiver は既定では delta を `invalid temporality and type combination` で拒否するため、
`compose.yml` で `PROMETHEUS_EXTRA_ARGS=--enable-feature=otlp-deltatocumulative` を渡して cumulative に変換しています。
この設定を外すと、ログ（Loki）は届くのにメトリクスだけが空になります。

grafana/otel-lgtm は各コンポーネントのログを既定で捨てるので、送信の失敗はコンテナのログに出ません。
調べるときは `compose.yml` の `environment` に一時的に `ENABLE_LOGS_ALL: "true"` を足して作り直すか、
Prometheus の `otelcol_exporter_send_failed_metric_points_total` が増えていないかを見ます。
