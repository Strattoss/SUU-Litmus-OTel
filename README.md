# Litmus Chaos & OpenTelemetry Demo

**Authors: Adam Biśta, Gracjan Filipek, Jakub Gawron, Krzysztof Usnarski**

2025, group 20

## 1. Introduction

This document shows how to use Litmus Chaos and OpenTelemetry to test how well a Kubernetes-based application can handle failures. We use Sock Shop — a free, open-source demo app built to mimic a real online store — because it's perfect for testing microservices and DevOps tools like monitoring, tracing, and chaos engineering.

## 2. Theoretical foundations

| Layer | Technology | Purpose |
| --- | --- | --- |
| Chaos | Litmus Chaos | Inject pod‑level & network faults via CRDs (Custom Resource Definition) |
| Metrics backend | Prometheus | Collect traces, scrape metrics |
| Metrics visualization | Grafana | Display dashboards |
| Platform | Kubernetes | Cluster |
| Application | Sock Shop | Workload under test |
| Infrastructure as Code | YAML files | Declarative deployment |

LitmusChaos is a Cloud-Native Chaos Engineering Framework with cross-cloud support. It is a CNCF Incubating project with adoption across several organizations. Its mission is to help Kubernetes SREs and Developers to find weaknesses in both Non-Kubernetes as well as platforms and applications running on Kubernetes by providing a complete Chaos Engineering framework and associated Chaos Experiments.

Litmus can be used to run chaos experiments initially in the staging environment and eventually in production to find bugs and vulnerabilities, fixing which leads to an increased resilience of the system. Litmus adopts a "Kubernetes-native" approach to define chaos intent in a declarative manner via Kubernetes custom resources (CRs).

### Importance of resilience
As Litmus documentation says: 'Keep verifying is the key. Robust testing before upgrades is not good enough, mainly because you cannot possibly consider all sorts of faults during upgrade testing. This introduces the concept of Chaos Engineering. The process of "continuously verifying if your service is resilient against faults" is called Chaos Engineering'.

**Key chaos terms**:

- **Steady‑state**: the normal, measurable behaviour of the system (e.g. request rate, latency, error rate) under healthy conditions.

- **Hypothesis**: the assumption that the system will continue to behave normally despite injected faults.

- **Chaos Experiments**: they are fundamental units within the LitmusChaos architecture. Users can choose between readily available chaos experiments or create new ones to construct a required Chaos Scenario.

- **Chaos Scenario**: more than a simple chaos experiment. It supports the user in defining the expected result, observing the result, analysing the overall system behaviour, and in the decision-making process if the system needs to be tuned for improving the resilience.

- **Chaos Engineering**: the process of "continuously verifying if your service is resilient against faults".

- **Fault**. A type of disruption in the application. Examples of faults:

![Chaos faults image](./images/chaos-faults-image.png)

- **Blast radius**: the scope or impact zone of the experiment (e.g. one pod, one service, or full namespace).

- **Abort criteria**: predefined conditions that will stop the experiment if the system becomes unstable (e.g. 5xx errors > 5%).

## 3. Case study concept description

Hypothesis: Sock Shop sustains p95 latency < 500 ms and HTTP 5xx ≤ 1 % while facing:

1. Random pod deletion (***pod-delete***).

2. 200 ms inter‑service network latency (***network-latency***).

The experiment workflow:

1. Establish steady‑state via load generator (50 RPS - Requests Per Second).

2. Inject the fault with Litmus CRDs.

3. Observe real‑time traces & metrics in Grafana.

4. Compare KPIs (Key Performance Indicator - A measurable value that shows how well a system is performing, e.g p95 **latency < 500 ms** or  **HTTP 5xx < 1%** )  against the hypothesis and produce a verdict (Pass / Fail).

**Experiment:**
Introducing 200 ms inter-service network latency to existing microservices cluster. 

![Fault point 1; network latency](./images/fault-point-1.png)
![Fault point 2; pod deletion](./images/fault-point-2.png)


## 4. Solution architecture

The application will be run on Amazon Web Service leveraging the Elastic Kubernetes Service. The business application, as mentioned in the previous section 1., will be the [SockShop](https://github.com/ocp-power-demos/sock-shop-demo), used widely as a demo application because of its microservice-centered structure. 

SockShop architecture schema:

![Socket shop architecture](./images/socket-shop-architecture.png)

Sock shop exporter will be scrapped for the ongoing communication statistics by the metrics collecting backend - [Prometheus](https://prometheus.io/).

In order to visualize the collected metrics and to observe changes that we expect when introducing the faults with [Litmus](https://litmuschaos.io/), we use [Grafana](https://grafana.com/) and set it up to work correctly with [Prometheus](https://prometheus.io/).

In the cluster there are three defined namespaces: `monitoring`,  `sock-shop` and `litmus`.

<!-- TODO: insert the whole cluster architecture diagram (with namespaces, litmus pods, experiments, engines, prometheus and grafana) -->

![Cluster architecture](./images/Litmus-architecture.svg)

1. **Application --> Prometheus**
   The Sock Shop application exposes metrics, which are scraped by Prometheus in the `monitoring` namespace.

2. **Prometheus --> Grafana**
   Grafana retrieves time series data from Prometheus to visualize system behavior and experiment outcomes.

3. **Litmus Operator --> Chaos Runner**
   The Litmus Operator manages and triggers the Chaos Runner based on defined `ChaosEngine` resources.

4. **Chaos Runner --> Application**
   The Chaos Runner injects faults into the application, such as deleting pods, to test its resilience.

5. **Chaos Runner --> ChaosResult**
   After executing an experiment, the Chaos Runner writes the outcome (e.g., verdict, duration) into a `ChaosResult` custom resource.

6. **Chaos Exporter --> ChaosResult**
   The Chaos Exporter reads experiment results from `ChaosResult` using the Kubernetes API.

7. **Chaos Exporter --> Prometheus**
   The Chaos Exporter exposes those results as Prometheus metrics, which are scraped for analysis and visualization.


## 5. Environment configuration description

Amazon Elastic Kubernetes Service - configuration of the cluster:

![Cluster config 1](./images/cluster-conf-1.png)
![Cluster config 2](./images/cluster-conf-2.png)
![Cluster config 3](./images/cluster-conf-3.png)
![Cluster config 4](./images/cluster-conf-4.png)
![Cluster config 5](./images/cluster-conf-5.png)

Node group configuration:

![Node group config 1](./images/cluster-group-conf-1.png)
![Node group config 2](./images/cluster-group-conf-2.png)

## 6. Installation method

In order to install the application on the Kubernetes cluster, one must first establish a connection with the AWS EKS cluster.

Verify the connection setup:
```bash
aws eks --region us-east-1 update-kubeconfig --name <cluster_name>
```

## 7. How to reproduce - step by step. Infrastructure as Code approach

All the necessary source code is in the `deploy/` directory in this repository.

Create sock shop namespace:
```bash
kubectl apply -f deploy/sock-shop/01-namespace.yaml
```

Install Litmus infrastructure components:
```bash
kubectl apply -f https://litmuschaos.github.io/litmus/litmus-operator-v3.0.0.yaml
```
```bash
kubectl apply -f https://litmuschaos.github.io/litmus/litmus-admin-rbac.yaml
```

Add monitoring (Prometheus + Grafana + Chaos Exporter):
```bash
kubectl apply -f deploy/litmus-metrics/chaos-exporter.yaml
kubectl apply -f deploy/monitoring/01-monitoring-ns.yaml
kubectl apply -f deploy/monitoring/02-prometheus-rbac.yaml
kubectl apply -f deploy/monitoring/03-prometheus-configmap.yaml
kubectl apply -f deploy/monitoring/04-prometheus-alert-rules.yaml
kubectl apply -f deploy/monitoring/05-prometheus-deployment.yaml
kubectl apply -f deploy/monitoring/06-prometheus-svc.yaml
kubectl apply -f deploy/monitoring/07-grafana-deployment.yaml
kubectl apply -f deploy/monitoring/08-grafana-svc.yaml
```

Deploy sock shop:
```bash
kubectl apply -f deploy/sock-shop/
```
```bash
watch -n 5 kubectl get pods -n sock-shop
```

Get the Prometheus cluster IP address:
```bash
kubectl get svc -n monitoring
```

Set up Grafana:
```bash
kubectl port-forward service/grafana 3000:3000 --namespace=monitoring
```
Log into Grafana (default credentials: username: admin, password: admin) and create a new datasource, inputing Prometheus address.

Create new dashboard. `Dashboard` --> `New` --> `Import` --> `Copy` [this for sockshop metrics](https://raw.githubusercontent.com/litmuschaos/litmus/master/demo/sample-applications/sock-shop/deploy/monitoring/10-grafana-dashboard.json) and [this for litmus chaos experiment metrics](https://raw.githubusercontent.com/litmuschaos/litmus/master/monitoring/grafana-dashboards/sock-shop/Sock-Shop-Performance-Under-Chaos.json).

## 8. Demo deployment steps:

### 1. Configuration set-up

The neccessary environment setup steps has been descibed in sections [Environment configuration description](#5-environment-configuration-description) and [Installation method](#6-installation-method).

Cluster components setup has been described in section [How to reproduce - step-by-step](#7-how-to-reproduce---step-by-step-infrastructure-as-code-approach).

### 2. Data preparation

Download and install Litmus experiments ([chaos experiments](https://github.com/litmuschaos/chaos-charts/tree/master)):
```bash
tar -zxvf <(curl -sL https://github.com/litmuschaos/chaos-charts/archive/3.0.0.tar.gz)
```
```bash
find chaos-charts-3.0.0 -name experiments.yaml | grep kubernetes | xargs kubectl apply -n litmus -f
```

### 3. Execution procedure

Choose an experiment from the [experiments](./experiments/) directory and run it:
```bash
kubectl apply -f experiments/<chosen-experiment>.yaml
```
**note**: `catalogue-experiments.yaml` contains three experiments: `pod-cpu-hog` ,`pod-delete` and `network-latency`.


Below are descriptions and visualizations of the three experiments included in `catalogue-experiments.yaml`:

### pod-delete
Deletes pods of an application to test self-healing and availability mechanisms.  
[Documentation](https://litmuschaos.github.io/litmus/experiments/categories/pods/pod-delete/)  
![pod-delete](./images/pod-delete-experiment.png)

### pod-network-latency
Injects network latency into a container to simulate poor network conditions and test application behavior.  
[Documentation](https://litmuschaos.github.io/litmus/experiments/categories/pods/pod-network-latency/)  
![pod-network-latency](./images/pod-network-latency-experiment.png)

### pod-cpu-hog 
Simulates high CPU usage on a specific container to test how the application handles resource stress.  
[Documentation](https://litmuschaos.github.io/litmus/experiments/categories/pods/pod-cpu-hog/)  
![pod-cpu-hog](./images/stress-the-cpu-experiment.png)



Verify the experiment 
```bash
 kubectl describe chaosengine catalogue-experiments -n litmus
```

In case of an error, check the logs:
```bash
kubectl logs <pod-name> -n litmus
```


### 4. Results presentation

![grafana-experiments](./images/grafana-experiments.png)

The Litmus Chaos experiments were successfully executed and monitored using Grafana dashboards. Three different types of faults were injected: `pod-cpu-hog`, `pod-delete`, and `pod-network-latency`. All chaos experiments completed successfully without failures.

**Chaos Experiments Injection**:
  Each of the three experiments was triggered and shown in the "Chaos Experiments" panel with one injection per fault type.

![grafana-experiments-pods-affected](./images/grafana-experiments-pods-affected.png)
![catalogue-latency-p95-less-500ms](./images/catalogue-latency-p95-less-500ms.png)
**System Behavior Metrics**:

* **Catalogue QPS** temporarily dropped during the `pod-delete` fault injection.
* **Catalogue Latency** briefly spiked during the `pod-network-latency` injection, with the 99th percentile reaching approximately 2 seconds.

* **Catalogue p95 Latency** temporarily increased during the `pod-network-latency` injection, peaking at **495 ms around 15:34:30**, but remained **within the hypothesis threshold (p95 < 500 ms)**.



## 9. Using AI in the project

## 10. Summary – conclusions

## 11. References
