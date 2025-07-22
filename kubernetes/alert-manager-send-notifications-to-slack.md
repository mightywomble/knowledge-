# Alert Manager send notifications to Slack

## Introduction

These will be instructions on integrating slack with the monitoring stack. This is done by updating the helm values with details about the slack alerts.. (I’l be honest at this point its a bit of a mystry to me, I’m not 100% how I got this working,b ut work it has on test and production.)

## **Create a Slack Webhook URL**

create a channel for alerts from Grafana. grafana-alerts as a channel name and grafana alert as a purpose

* Go to [https://api.slack.com/apps/new](https://api.slack.com/apps/new) to create a slack app
* Input App Name and select Workspace, then click “Create App”
* Click Incoming Webhooks
* Switch the radio button to On, then click Add New Webhook to Workspace
* Select the channel to send alerts, then click Allow
* You will get a Webhook URL. Copy it.

The webhook url will uses in helm chart of grafana stack , this will used by Grafa

## Create values.yaml

An update to the helm values.yaml is needed to update the AlertManager configuration and point the alerts to slack.

Create this as **values.cudos.yaml**

```yaml
  alertmanager:
  config:
    global: null
  service:
    type: LoadBalancer
  config:
    global:
      resolve_timeout: 5m
      slack_api_url: "<https://hooks.slack.com/services/T7RD4GgsgasdasgfO9hK4ly2qz7FybIQ9ozAN5Ch>"
    receivers:
    - name: "null"
    - name: slack
      slack_configs:
      - channel: '#intercloud-monitoring'
        send_resolved: false
        text: |-
          {{ range .Alerts }}
            *Alert:* {{ .Annotations.summary }} - `{{ .Labels.severity }}`
            *Description:* {{ .Annotations.description }}
            *Graph:* <{{ .GeneratorURL }}|:chart_with_upwards_trend:> *Runbook:* <{{ .Annotations.runbook }}|:spiral_note_pad:>
            *Details:*
            {{ range .Labels.SortedPairs }} • *{{ .Name }}:* `{{ .Value }}`
            {{ end }}
          {{ end }}
        title: '[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing
          | len }}{{ end }}] Monitoring Event Notification'
  route:
    group_by:
    - job
    group_interval: 5m
    group_wait: 30s
    repeat_interval: 12h
    routes:
    - match:
        alertname: DeadMansSwitch
      receiver: "null"
    - continue: true
      match: null
      receiver: slack
  enabled: true
  podDisruptionBudget:
    enabled: false
    minAvailable: 1
  serviceAccount:
    create: true
podMonitorSelectorNilUsesHelmValues: false
probeSelectorNilUsesHelmValues: false
grafana:
  ingress:
    enabled: true
    hosts:
    - alert.intercloud.cudos.org
prometheus:
    service:
    type: LoadBalancer
ruleSelectorNilUsesHelmValues: false
scrapeConfigSelectorNilUsesHelmValues: false
serviceMonitorSelectorNilUsesHelmValues: false
```

Save and Exit the file

## Update the Helm Install

```yaml
helm upgrade --install -f values-cudo.yaml kube-prometheus-stack prometheus-community/kube-prometheus-stack -n cudo-monitoring
```

## Notes

### External IP’s for Prometheus and AlertManager

In the values-cudo.yaml Ive added the following two sections

```yaml
prometheus:
    service:
    type: LoadBalancer
```

and

```yaml
  alertmanager:
  config:
    global: null
  service:
    type: LoadBalancer
```

These are added to bypass the caddy load balancer and get two internet accessible IP’s this can be seen using the command

```yaml
kubectl get services -n cudo-monitoring
```

which will return

```yaml
NAME                                             TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                         AGE
alertmanager-operated                            ClusterIP      None             <none>           9093/TCP,9094/TCP,9094/UDP      25h
**kube-prometheus-stack-alertmanager               LoadBalancer   10.128.42.52     139.144.247.89   9093:30999/TCP,8080:32354/TCP   25h**
kube-prometheus-stack-grafana                    ClusterIP      10.128.212.55    <none>           80/TCP                          25h
kube-prometheus-stack-kube-state-metrics         ClusterIP      10.128.27.158    <none>           8080/TCP                        25h
kube-prometheus-stack-operator                   ClusterIP      10.128.87.50     <none>           443/TCP                         25h
**kube-prometheus-stack-prometheus                 LoadBalancer   10.128.72.0      139.144.247.20   9090:30576/TCP,8080:30125/TCP   25h**
kube-prometheus-stack-prometheus-node-exporter   ClusterIP      10.128.21.253    <none>           9100/TCP                        25h
loki-loki-distributed-distributor                ClusterIP      10.128.122.67    <none>           3100/TCP,9095/TCP               25h
loki-loki-distributed-gateway                    ClusterIP      10.128.188.142   <none>           80/TCP                          25h
loki-loki-distributed-ingester                   ClusterIP      10.128.20.231    <none>           3100/TCP,9095/TCP               25h
loki-loki-distributed-ingester-headless          ClusterIP      None             <none>           3100/TCP,9095/TCP               25h
loki-loki-distributed-memberlist                 ClusterIP      None             <none>           7946/TCP                        25h
loki-loki-distributed-querier                    ClusterIP      10.128.202.31    <none>           3100/TCP,9095/TCP               25h
loki-loki-distributed-querier-headless           ClusterIP      None             <none>           3100/TCP,9095/TCP               25h
loki-loki-distributed-query-frontend             ClusterIP      10.128.243.70    <none>           3100/TCP,9095/TCP,9096/TCP      25h
loki-loki-distributed-query-frontend-headless    ClusterIP      None             <none>           3100/TCP,9095/TCP,9096/TCP      25h
prometheus-operated                              ClusterIP      None             <none>           9090/TCP                        25h
```

This will allow access to the interfaces using external IP’s

Disable this once setup by putting [\*\*#](https://www.google.com/url?sa=t\&rct=j\&q=\&esrc=s\&source=web\&cd=\&ved=2ahUKEwiTusqT5eyGAxWWA9sEHccoDAkQFnoECBgQAw\&url=https%3A%2F%2Fm.youtube.com%2Fwatch%3Fv%3DdI0GjHUKTkQ\&usg=AOvVaw2FRA3MCNcbsf6osJ3vRsr7\&opi=89978449)\*\* infront of those value statements.

### Config in AlertManager

This config shows up as follows in the AlertManager → status Window

```yaml
**Config**
global:
  resolve_timeout: 5m
  http_config:
    follow_redirects: true
    enable_http2: true
  smtp_hello: localhost
  smtp_require_tls: true
  **slack_api_url: <secret>**
  pagerduty_url: <https://events.pagerduty.com/v2/enqueue>
  opsgenie_api_url: <https://api.opsgenie.com/>
  wechat_api_url: <https://qyapi.weixin.qq.com/cgi-bin/>
  victorops_api_url: <https://alert.victorops.com/integrations/generic/20131114/alert/>
  telegram_api_url: <https://api.telegram.org>
  webex_api_url: <https://webexapis.com/v1/messages>
route:
  receiver: "null"
  group_by:
  - job
  continue: false
  routes:
  - receiver: "null"
    match:
      alertname: DeadMansSwitch
    continue: false
  - receiver: slack
    continue: true
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
inhibit_rules:
- source_matchers:
  - severity="critical"
  target_matchers:
  - severity=~"warning|info"
  equal:
  - namespace
  - alertname
- source_matchers:
  - severity="warning"
  target_matchers:
  - severity="info"
  equal:
  - namespace
  - alertname
- source_matchers:
  - alertname="InfoInhibitor"
  target_matchers:
  - severity="info"
  equal:
  - namespace
- target_matchers:
  - alertname="InfoInhibitor"
receivers:
- name: "null"
**- name: slack
  slack_configs:
  - send_resolved: false
    http_config:
      follow_redirects: true
      enable_http2: true
    api_url: <secret>
    channel: '#intercloud-monitoring'
    username: '{{ template "slack.default.username" . }}'
    color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
    title: '[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing
      | len }}{{ end }}] Monitoring Event Notification'
    title_link: '{{ template "slack.default.titlelink" . }}'
    pretext: '{{ template "slack.default.pretext" . }}'
    text: |-
      {{ range .Alerts }}
        *Alert:* {{ .Annotations.summary }} - `{{ .Labels.severity }}`
        *Description:* {{ .Annotations.description }}
        *Graph:* <{{ .GeneratorURL }}|:chart_with_upwards_trend:> *Runbook:* <{{ .Annotations.runbook }}|:spiral_note_pad:>
        *Details:*
        {{ range .Labels.SortedPairs }} • *{{ .Name }}:* `{{ .Value }}`
        {{ end }}
      {{ end }}
    short_fields: false
    footer: '{{ template "slack.default.footer" . }}'
    fallback: '{{ template "slack.default.fallback" . }}'
    callback_id: '{{ template "slack.default.callbackid" . }}'
    icon_emoji: '{{ template "slack.default.iconemoji" . }}'
    icon_url: '{{ template "slack.default.iconurl" . }}'
    link_names: false**
templates:
- /etc/alertmanager/config/*.tmpl
```

## Reference:

[https://medium.com/@abdulfayis/prometheus-grafana-alert-manager-slack-notifications-845ead17d429](https://medium.com/@abdulfayis/prometheus-grafana-alert-manager-slack-notifications-845ead17d429)

[https://gist.github.com/l13t/d432b63641b6972b1f58d7c037eec88f](https://gist.github.com/l13t/d432b63641b6972b1f58d7c037eec88f)
