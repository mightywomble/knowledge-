# Install Promtail and Loki for Kubernetes Monitoring

### Install Helm Repos

Add the Grafana Helm Charts to your Helm cli:

```
helm repo add grafana <https://grafana.github.io/helm-charts>
helm repo update
```

### Install Loki

Run

```yaml
helm upgrade --install loki grafana/loki-distributed -n monitoring --set service.type=LoadBalancer
```

### Check Pods

```yaml
kubectl get pods -n monitoring 
```

expected output

```yaml
NAME                                                        READY   STATUS    RESTARTS   AGE
alertmanager-kube-prometheus-stack-alertmanager-0           2/2     Running   0          23h
kube-prometheus-stack-grafana-76858ff8dd-nnh94              3/3     Running   0          23h
kube-prometheus-stack-kube-state-metrics-7f6967956d-tzrkm   1/1     Running   0          23h
kube-prometheus-stack-operator-79b45fdb47-ccqc6             1/1     Running   0          23h
kube-prometheus-stack-prometheus-node-exporter-bxbtc        1/1     Running   0          23h
kube-prometheus-stack-prometheus-node-exporter-j9gjg        1/1     Running   0          23h
kube-prometheus-stack-prometheus-node-exporter-r2fqw        1/1     Running   0          23h
**loki-loki-distributed-distributor-b8448bd4b-2twdh           1/1     Running   0          3m15s
loki-loki-distributed-gateway-9d8b76d6d-mvxps               1/1     Running   0          3m15s
loki-loki-distributed-ingester-0                            1/1     Running   0          3m15s
loki-loki-distributed-querier-0                             1/1     Running   0          3m15s
loki-loki-distributed-query-frontend-6db884fbdd-zfs2s       1/1     Running   0          3m15s**
prometheus-kube-prometheus-stack-prometheus-0               2/2     Running   0          23h
```

Use watch to view changes while starting

```yaml
watch kubectl get pods -n monitoring 
```

### Check Services

run

```yaml
kubectl get services -n monitoring
```

Expected results

```yaml
NAME                                             TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
alertmanager-operated                            ClusterIP      None            <none>        9093/TCP,9094/TCP,9094/UDP      23h
kube-prometheus-stack-alertmanager               LoadBalancer   10.108.18.33    10.10.0.242   9093:31456/TCP,8080:31493/TCP   23h
kube-prometheus-stack-grafana                    LoadBalancer   10.105.8.65     10.10.0.241   80:31941/TCP    23h
kube-prometheus-stack-kube-state-metrics         ClusterIP      10.97.4.3       <none>        8080/TCP        23h
kube-prometheus-stack-operator                   ClusterIP      10.107.254.83   <none>        443/TCP         23h
kube-prometheus-stack-prometheus                 LoadBalancer   10.108.245.2    10.10.0.243   9090:30518/TCP,8080:31169/TCP   23h
kube-prometheus-stack-prometheus-node-exporter   ClusterIP      10.105.24.16    <none>        9100/TCP        23h
**loki-loki-distributed-distributor                ClusterIP      10.102.78.123   <none>        3100/TCP,9095/TCP               4m9s
loki-loki-distributed-gateway                    ClusterIP      10.97.122.135   <none>        80/TCP          4m9s
loki-loki-distributed-ingester                   ClusterIP      10.110.15.213   <none>        3100/TCP,9095/TCP               4m9s
loki-loki-distributed-ingester-headless          ClusterIP      None            <none>        3100/TCP,9095/TCP               4m9s
loki-loki-distributed-memberlist                 ClusterIP      None            <none>        7946/TCP        4m9s
loki-loki-distributed-querier                    ClusterIP      10.108.56.75    <none>        3100/TCP,9095/TCP               4m9s
loki-loki-distributed-querier-headless           ClusterIP      None            <none>        3100/TCP,9095/TCP               4m9s
loki-loki-distributed-query-frontend             ClusterIP      10.100.183.25   <none>        3100/TCP,9095/TCP,9096/TCP      4m9s
loki-loki-distributed-query-frontend-headless    ClusterIP      None            <none>        3100/TCP,9095/TCP,9096/TCP      4m9s
pro**metheus-operated                              ClusterIP      None            <none>        9090/TCP        23h
```

### Add Loki to Grafana

To add the Loki Data Source head to home→ Conections → data Source in Grafana

Select Loki as a Data Source

Use the URL

```yaml
<http://loki-loki-distributed-query-frontend.monitoring:3100>
```

### Doing this automatically

I think this can be added to values.yaml using which should setup Loki automagically.

```yaml
grafana:
  sidecar:
    datasources:
      defaultDatasourceEnabled: true
  additionalDataSources:
    - name: Loki
      type: loki
      url: <http://loki-loki-distributed-query-frontend.monitoring:3100>
```

### Install Promtail

Promtail pushes data to Loki

A values file is needed specifically for the promtail config

```yaml
nano promtail-values.yaml
```

Add the following

```yaml
---
config:
clients:
- url: "<http://loki-loki-distributed-gateway/loki/api/v1/push>"
---
```

Run the command

```yaml
helm upgrade --install promtail grafana/promtail -f promtail-values.yaml -n monitoring

```

### Check Pods

```yaml
kubectl get pods -n monitoring 
```

expected output

```yaml
NAME                                                        READY   STATUS    RESTARTS   AGE
alertmanager-kube-prometheus-stack-alertmanager-0           2/2     Running   0          23h
kube-prometheus-stack-grafana-76858ff8dd-nnh94              3/3     Running   0          23h
kube-prometheus-stack-kube-state-metrics-7f6967956d-tzrkm   1/1     Running   0          23h
kube-prometheus-stack-operator-79b45fdb47-ccqc6             1/1     Running   0          23h
kube-prometheus-stack-prometheus-node-exporter-bxbtc        1/1     Running   0          23h
kube-prometheus-stack-prometheus-node-exporter-j9gjg        1/1     Running   0          23h
kube-prometheus-stack-prometheus-node-exporter-r2fqw        1/1     Running   0          23h
loki-loki-distributed-distributor-b8448bd4b-2twdh           1/1     Running   0          22m
loki-loki-distributed-gateway-9d8b76d6d-mvxps               1/1     Running   0          22m
loki-loki-distributed-ingester-0                            1/1     Running   0          22m
loki-loki-distributed-querier-0                             1/1     Running   0          22m
loki-loki-distributed-query-frontend-6db884fbdd-zfs2s       1/1     Running   0          22m
prometheus-kube-prometheus-stack-prometheus-0               2/2     Running   0          23h
**promtail-25bjr                                              1/1     Running   0          3m21s
promtail-5kmlt                                              1/1     Running   0          3m21s
promtail-h9mrf                                              1/1     Running   0          3m21s**
```

Use watch to view changes while starting

```yaml
watch kubectl get pods -n monitoring 
```

### Check Services

run

```yaml
kubectl get services -n monitoring
```

Expected results

```yaml
NAME                                             TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
alertmanager-operated                            ClusterIP      None            <none>        9093/TCP,9094/TCP,9094/UDP      23h
kube-prometheus-stack-alertmanager               LoadBalancer   10.108.18.33    10.10.0.242   9093:31456/TCP,8080:31493/TCP   23h
kube-prometheus-stack-grafana                    LoadBalancer   10.105.8.65     10.10.0.241   80:31941/TCP    23h
kube-prometheus-stack-kube-state-metrics         ClusterIP      10.97.4.3       <none>        8080/TCP        23h
kube-prometheus-stack-operator                   ClusterIP      10.107.254.83   <none>        443/TCP         23h
kube-prometheus-stack-prometheus                 LoadBalancer   10.108.245.2    10.10.0.243   9090:30518/TCP,8080:31169/TCP   23h
kube-prometheus-stack-prometheus-node-exporter   ClusterIP      10.105.24.16    <none>        9100/TCP        23h
loki-loki-distributed-distributor                ClusterIP      10.102.78.123   <none>        3100/TCP,9095/TCP               24m
loki-loki-distributed-gateway                    ClusterIP      10.97.122.135   <none>        80/TCP          24m
loki-loki-distributed-ingester                   ClusterIP      10.110.15.213   <none>        3100/TCP,9095/TCP               24m
loki-loki-distributed-ingester-headless          ClusterIP      None            <none>        3100/TCP,9095/TCP               24m
loki-loki-distributed-memberlist                 ClusterIP      None            <none>        7946/TCP        24m
loki-loki-distributed-querier                    ClusterIP      10.108.56.75    <none>        3100/TCP,9095/TCP               24m
loki-loki-distributed-querier-headless           ClusterIP      None            <none>        3100/TCP,9095/TCP               24m
loki-loki-distributed-query-frontend             ClusterIP      10.100.183.25   <none>        3100/TCP,9095/TCP,9096/TCP      24m
loki-loki-distributed-query-frontend-headless    ClusterIP      None            <none>        3100/TCP,9095/TCP,9096/TCP      24m
prometheus-operated                              ClusterIP      None            <none>        9090/TCP        23h
```

## Output

The resulting output of this is the ability to

Open Grafana

Head to Explorer

Choose Loki as a DataSource
