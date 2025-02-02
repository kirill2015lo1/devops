rate 
increase

https://www.fosstechnix.com/prometheus-promql-tutorial-with-examples/

Колличество ошибок в минут
sum(increase(flask_app_requests_total {job="test", http_status=~"4.+|5.+"}[1m])) by (http_status)

Коллиество запрросов к приложению в минуту со всех подов, и разделение их по статусу, для понимаю какие статутос какое колличество 
sum(rate(flask_app_requests_total {job="test"}[1m])) by (http_status)


sum(increase(flask_app_requests_total {job="test", http_status=~"2.+|3.+"}[1m])) / sum(increase(flask_app_requests_total {job="test"}[1m]))
Процент успешных запросов общего числа в минуту, должно быть близко к 100 процентам


квантиль:  
histogram_quantile( 0.50,
    sum(rate(flask_app_request_latency_seconds_bucket{job="test"}[1m])) by (le,method)
)
legend:  
99-{{method}}




Правило для уведомления 
```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: flask-alerts
  namespace: monitoring
spec:
  groups:
    - name: flask-app-alerts
      rules:
        - alert: HighRequestCount
          expr: sum(rate(flask_app_requests_total[1m])) > 3
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "High request count for flask application"
            description: "The request rate for flask app has exceeded 3 requests per minute."

```

конфиг values для helm chart prometheus Для настройки alertmanager 
```
alertmanager:
  enabled: true
  apiVersion: v2
  enableFeatures: []
  forceDeployDashboards: false
  serviceAccount:
    create: true
    name: ""
    annotations: {}
    automountServiceAccountToken: true

  podDisruptionBudget:
    enabled: false
    minAvailable: 1
    maxUnavailable: ""

  config:
    global:
      resolve_timeout: 5m

    # Инфо для подавления алертов
    inhibit_rules:
      - source_matchers:
          - 'severity = critical'
        target_matchers:
          - 'severity =~ warning|info'
        equal:
          - 'namespace'
          - 'alertname'
      - source_matchers:
          - 'severity = warning'
        target_matchers:
          - 'severity = info'
        equal:
          - 'namespace'
          - 'alertname'
      - source_matchers:
          - 'alertname = InfoInhibitor'
        target_matchers:
          - 'severity = info'
        equal:
          - 'namespace'
      - target_matchers:
          - 'alertname = InfoInhibitor'

    # Маршрутизация алертов
    route:
      group_by: ['namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'telegram'  # Укажите, что уведомления будут отправляться в telegram
      routes:
        - receiver: 'telegram'
          matchers:
            - alertname = "Watchdog"

    # Конфигурация для Telegram
    receivers:
      - name: 'telegram'
        telegram_configs:
          - bot_token: 'YOUR_BOT_TOKEN'  # Замените на ваш токен бота
            chat_id: 'YOUR_CHAT_ID'      # Замените на ваш chat_id
            send_resolved: true

    # Шаблоны, если нужны
    templates:
      - '/etc/alertmanager/config/*.tmpl'

```
