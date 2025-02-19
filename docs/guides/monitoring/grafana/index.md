---
title: KubeStash Grafana Dashboard | KubeStash
description: Using KubeStash Grafana Dashboard
menu:
  docs_{{ .version }}:
    identifier: monitoring-grafana-dashboard
    name: Grafana Dashboard
    parent: monitoring
    weight: 30
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# KubeStash Grafana Dashboard

Grafana provides an elegant graphical user interface to visualize data. You can create a beautiful dashboard easily with a meaningful representation of your Prometheus metrics.

We provide a pre-built Grafana dashboard to our **KubeStash** users. In this guide, we are going to show you how you can import this dashboard from your Grafana UI.


## Before You Begin

- At first, you need to setup a Prometheus monitoring stack in your cluster. Please, follow the [Prometheus Operator](/docs/guides/monitoring/prom-operator/index.md) guide to setup your monitoring stack if you haven't done already.
- Then, install KubeStash with monitoring enabled. Please, follow [this guide](/docs/guides/monitoring/prom-operator/index.md#enable-monitoring-in-kubestash) if you haven't done already.
- You must have `kubestash_dashboard.json` file. Please, contact us to get the dashboard JSON file.

## Install Panopticon

KubeStash Grafana dashboard depends on our another product called [Panopticon](https://blog.byte.builders/post/introducing-panopticon/). It is a Kubernetes state metric exporter similar to [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) but generic for all Kubernetes resources including CRDs. In this section, we are going to show the installation procedure for Panopticon.

**Issue License:**

Like other AppsCode products, [Panopticon](https://blog.byte.builders/post/introducing-panopticon/) also need a license to run. You can grab a 30 days trial license for Panopticon from [here](https://license-issuer.appscode.com/?p=panopticon-enterprise).

>**If you already have a license for KubeDB or KubeStash, you do not need to issue a new license for Panopticon. Your existing KubeDB or KubeStash license will work with Panopticon.**

**Install Panopticon:**

Now, install Panopticon using the following commands:

```bash
$ helm repo add appscode https://charts.appscode.com/stable/
$ helm repo update

$ helm install panopticon appscode/panopticon -n kubeops \
    --create-namespace \
    --set monitoring.enabled=true \
    --set monitoring.agent=prometheus.io/operator \
    --set monitoring.serviceMonitor.labels.release=prometheus-stack \
    --set-file license=/path/to/license-file.txt
```

Make sure to use the appropriate label in `monitoring.serviceMonitor.labels` field according to your setup. This label is used by the Prometheus server to select the desired ServiceMonitor.

## Import KubeStash Garafana Dashboard

At first, let's port-forward the respective service for the Grafana dashboard so that we can access it through our browser locally.

```bash
$ kubectl port-forward -n monitoring service/prometheus-stack-grafana 3000:80
Forwarding from 127.0.0.1:3000 -> 3000
Forwarding from [::1]:3000 -> 3000
```

Now, go to [http://localhost:3000](http://localhost:3000/) in your browser and login to your Grafana UI. If you followed the [Prometheus Operator](/docs/guides/monitoring/prom-operator/index.md) guide to deploy your Prometheus stack, then the default username and password should be `admin`, and `prom-operator` respectively.

Then, on the Grafana UI, click the `+` icon from the left sidebar and then click on `Import` button as below,

<figure align="center">
  <img alt="Import KubeStash Grafana Dashboard: Step 1" src="/docs/guides/monitoring/grafana/images/import_dashboard_1.png">
<figcaption align="center">Fig: Import KubeStash Grafana Dashboard (Step 1)</figcaption>
</figure>

Then, on the import UI, you can either upload the `kubestash_dashboard.json` file by clicking the `Upload JSON file` button or you can paste the content of the JSON file in the text area labeled as `Import via panel json`.

<figure align="center">
  <img alt="Import KubeStash Grafana Dashboard: Step 2" src="/docs/guides/monitoring/grafana/images/import_dashboard_2.png">
<figcaption align="center">Fig: Import KubeStash Grafana Dashboard (Step 2)</figcaption>
</figure>

If you followed the instruction properly, you should see the KubeStash Grafana dashboard in your Grafana UI.

<figure align="center">
  <img alt="KubeStash Grafana Dashboard" src="/docs/guides/monitoring/grafana/images/kubestash_grafana_dashboard.png">
<figcaption align="center">Fig: KubeStash Grafana Dashboard</figcaption>
</figure>

>If your cluster does not have any backup configured, you may see the dashboard panels are empty. Nothing to worry about here. Just, run some backups and your dashboard should be populated automatically.

