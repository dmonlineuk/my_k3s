# Observability

## Create the monitoring namespace

Run this:

```bash
kubectl create namespace monitoring
```

Then verify:

```bash
kubectl get namespaces
```

Expected output:

```stdout
NAME              STATUS   AGE
...
monitoring        Active   10s
```

---

## Add the Prometheus Helm repository

Run:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update
```

Then:

```bash
helm search repo prometheus-community/kube-prometheus-stack
```

It is recommended to pin the chart version rather than blindly installing whatever happens to be latest. The current upstream chart is 88.2.0 at the time of writing, with Prometheus Operator 0.93.0 and Grafana 12.10.0 (?). Pinning prevents unexpected outcomes when updating later.

---

## Set DNS

Configure DNS so:

```
grafana → 192.168.100.100
```

Then to test, use ping, nslookup or something similar:

```bash
ping grafana
```

Response should be something like:

```stdout
PING grafana.loc (192.168.100.100)
```

---

## Failover test

This is worth doing **before** installing anything. First confirm the VIP is currently on `lb-01`:

```bash
ip addr show dev enp1s0
```

Expected output:

```stdout
192.168.100.100
```

Then from somewhere else on the network:

```bash
curl -v http://192.168.100.100/
```

Expected response from Traefik:

```stdout
404 page not found
```

Now temporarily stop Keepalived on **lb-01**:

```bash
sudo systemctl stop keepalived
```

Don't stop HAProxy — we're specifically testing VIP failover. Within a couple of seconds, `192.168.100.100` should move to `lb-02`. On `lb-02`:

```bash
ip addr show dev enp1s0
```

You should now see:

```stdout
192.168.100.99
192.168.100.100
```

Then from the network:

```bash
curl -v http://192.168.100.100/
```

You should **still get the Traefik 404**. If that works, we've proven the entire HA ingress path works. Then restore `lb-01`:

```bash
sudo systemctl start keepalived
```

Because it has priority `200` versus `100`, it should reclaim the VIP. Check:

```bash
ip addr show dev enp1s0
```

on both machines.

---

## Install Prometheus

Create the values.yaml (one is inside this repo folder). Let's install this deployment using helm:

```bash
helm upgrade --install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values.yaml
```

Test things with:

```bash
helm list -n monitoring
kubectl get pods -n monitoring
```

To fetch the password for the admin user:

```bash
kubectl --namespace monitoring get secrets kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```

Using a browser, visit `http://grafana.loc` to log into grafana as the admin user:
