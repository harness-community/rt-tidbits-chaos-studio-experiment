# Harness Resilience Testing Tidbit - Pod Network Latency Chaos Experiment

> **Level:** 101 – Beginner  
> **Duration:** ~10 minutes  
> **Module:** Resilience Testing

Build your first chaos experiment from scratch using **Harness Chaos Studio**. This tutorial first deploys an **nginx-based Kubernetes service** (`resilience-demo` / `resilience-demo-svc` in `chaos-demo`), then configures Harness and Service Discovery so the application is visible before injecting a Pod Network Latency fault and validating steady-state behavior with an HTTP probe.

---

## Table of Contents

- [Overview](#overview)
- [Target Service](#target-service)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Deploy the Sample Application](#deploy-the-sample-application)
  - [1. Deploy the Sample Application](#1-deploy-the-sample-application)
  - [2. Verify the Deployment](#2-verify-the-deployment)
- [Configure Harness](#configure-harness)
  - [1. Install a Harness Delegate](#1-install-a-harness-delegate)
  - [2. Create a Kubernetes Connector](#2-create-a-kubernetes-connector)
  - [3. Create the Resilience Testing Infrastructure](#3-create-the-resilience-testing-infrastructure)
  - [4. Configure Service Discovery](#4-configure-service-discovery)
  - [5. Verify the nginx Application Is Discovered](#5-verify-the-nginx-application-is-discovered)
- [Run the Chaos Experiment](#run-the-chaos-experiment)
  - [Step 1: Create a New Experiment](#step-1-create-a-new-experiment)
  - [Step 2: Add the Pod Network Latency Fault](#step-2-add-the-pod-network-latency-fault)
  - [Step 3: Attach an HTTP Probe](#step-3-attach-an-http-probe)
  - [Step 4: Execute and Analyze](#step-4-execute-and-analyze)
- [Expected Outcome](#expected-outcome)
- [Cleanup](#cleanup)
- [Additional Resources](#additional-resources)
- [License](#license)

---



## Overview

**Resilience Testing** is the systematic verification of a system's ability to maintain service continuity and recover gracefully from failures. Rather than waiting for production incidents, we proactively validate recovery mechanisms under controlled conditions.

This tutorial demonstrates:


| Concept              | What You Will Learn                                                              |
| -------------------- | -------------------------------------------------------------------------------- |
| **Faults**           | How to inject controlled network delay (Pod Network Latency) into a running service |
| **Probes**           | How to set up HTTP health checks that validate steady-state behavior             |
| **Service Discovery** | How to discover the deployed nginx workload in Harness                          |
| **Resilience Score** | How to measure and quantify your system's ability to stay available under stress |


---



## Target Service

This tutorial deploys a multi-replica **nginx** web service into the `chaos-demo` namespace. That service is the fault-injection and HTTP probe target.

| Resource | Name | Details |
| -------- | ---- | ------- |
| **Namespace** | `chaos-demo` | Isolated namespace for the demo |
| **ConfigMap** | `resilience-demo-nginx` | Nginx listen config for container port **8080** |
| **Deployment** | `resilience-demo` | `nginx:stable-alpine`, **3 replicas**, label `app=resilience-demo` |
| **Service** | `resilience-demo-svc` | **ClusterIP** on port **80** → container port **8080** |

- Image: `nginx:stable-alpine` (default welcome page on `/`)
- Health: readiness and liveness HTTP probes on `/` port **8080**
- Chaos Studio probe URL: `http://resilience-demo-svc.chaos-demo.svc.cluster.local`

Three replicas let a Pod Network Latency fault that affects ~50% of pods leave unaffected replicas able to serve traffic while delayed pods experience injected network delay. All of this is defined in a single manifest: `k8s/deployment.yaml`.

---



## Repository Structure

```
rt-tidbits-chaos-studio-experiment/
├── README.md                           # This file — full tutorial guide
├── LICENSE                             # Apache License 2.0
└── k8s/
    └── deployment.yaml                 # Namespace, ConfigMap, nginx Deployment, and ClusterIP Service
```

---



## Prerequisites

Before starting, ensure the following are in place:

1. Access to a Harness account, organization, and project with permission to configure connectors, delegates, Resilience Testing infrastructure, and Service Discovery.
2. Access to a running Kubernetes cluster. Any of the following will work:


| Provider     | Command / Link                                                                            |
| ------------ | ----------------------------------------------------------------------------------------- |
| **Minikube** | `minikube start --cpus=2 --memory=4096`                                                   |
| **GKE**      | [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine)                    |
| **EKS**      | [Amazon Elastic Kubernetes Service](https://aws.amazon.com/eks/)                          |
| **AKS**      | [Azure Kubernetes Service](https://azure.microsoft.com/en-us/products/kubernetes-service) |


3. `kubectl` configured for the target cluster. Verify access:

```bash
kubectl cluster-info
kubectl get nodes
```

4. A cluster that permits the privileged helper pods required by the Pod Network Latency fault.

---



## Deploy the Sample Application



### 1. Deploy the Sample Application

Clone this repository and apply the Kubernetes manifests:

```bash
# Clone the repo
git clone https://github.com/animesh-sri-harness/rt-tidbits-chaos-studio-experiment-.git
cd rt-tidbits-chaos-studio-experiment-

# Deploy namespace, ConfigMap, nginx app (3 replicas), and ClusterIP service
kubectl apply -f k8s/deployment.yaml
```



### 2. Verify the Deployment

```bash
# Check that all 3 pods are running
kubectl get pods -n chaos-demo -l app=resilience-demo

# Expected output:
# NAME                               READY   STATUS    RESTARTS   AGE
# resilience-demo-xxxxx-aaaaa        1/1     Running   0          30s
# resilience-demo-xxxxx-bbbbb        1/1     Running   0          30s
# resilience-demo-xxxxx-ccccc        1/1     Running   0          30s

# Verify the service
kubectl get svc -n chaos-demo

# Test connectivity (from within the cluster)
kubectl run curl-test --rm -i --tty --image=curlimages/curl --namespace=chaos-demo \
  -- curl -s http://resilience-demo-svc.chaos-demo.svc.cluster.local
```

---

## Configure Harness

Configure Harness only after the sample application is running. This allows the first Service Discovery scan to find the nginx workload.

### 1. Install a Harness Delegate

Chaos experiments run through the **Delegate-Driven Chaos Runner (DDCR)**. Install a Harness Delegate that can reach the Kubernetes cluster by following [Install a Delegate on Kubernetes](https://developer.harness.io/3k-docs/platform/delegates-v2/install-a-delegate/install-kubernetes-delegate/), then confirm that it shows as **Connected** in Harness.

### 2. Create a Kubernetes Connector

Create a [Kubernetes connector](https://developer.harness.io/docs/platform/connectors/cloud-providers/ref-cloud-providers/kubernetes-cluster-connector-settings-reference) for the target cluster:

1. In the Harness project, go to **Project Settings → Connectors**.
2. Select **New Connector → Kubernetes Cluster**.
3. Configure the connector to use the credentials available to the Delegate.
4. Select the Delegate or matching Delegate tags.
5. Test the connection and save the connector.

### 3. Create the Resilience Testing Infrastructure

1. Go to **Resilience Testing → Project Settings → Resilience Testing Infrastructures**.
2. Select **Kubernetes (Harness Infrastructure)** and click **New Infrastructure**.
3. Pick or create an environment, then click **Continue**.
4. Configure:
   - **Deployment Type:** Kubernetes
   - **Infrastructure Type:** Direct Connection (Kubernetes)
   - **Connector:** the Kubernetes connector created above
   - **Namespace:** `chaos-demo`
5. Save the infrastructure and complete the setup wizard.
6. Wait until its status is **Active**.

For more information, see [Set up Kubernetes chaos infrastructure](https://developer.harness.io/docs/resilience-testing/chaos-testing/infrastructure/kubernetes).

### 4. Configure Service Discovery

Service Discovery is separate from the Resilience Testing infrastructure. It continuously scans the cluster and builds an inventory of Kubernetes workloads.

1. Go to **Resilience Testing → Project Settings → Discovery**.
2. Create a discovery agent.
3. Select the environment, infrastructure, and Kubernetes connector configured above.
4. Configure the agent to discover the `chaos-demo` namespace.
5. Choose the scan schedule or persistent-agent option available in your account.
6. Save the agent and follow the on-screen instructions to install any generated resources.
7. Wait for the first discovery scan to complete successfully.

See [Service Discovery](https://developer.harness.io/harness-platform/use-harness-platform/service-discovery) for details.

### 5. Verify the nginx Application Is Discovered

1. In **Project Settings → Discovery**, open the discovered workload inventory.
2. Filter by namespace `chaos-demo`.
3. Confirm that `resilience-demo` or `resilience-demo-svc` appears.
4. Optionally go to **Resilience Testing → Insights → Application Maps**, create a map, and select the discovered nginx service.

If it does not appear, confirm that the application pods are running, the discovery agent is healthy, the selected connector reaches the same cluster, and the agent includes the `chaos-demo` namespace.

> Service Discovery inventories workloads continuously. If your account uses Resilience Testing service onboarding, select the discovered nginx workload in the onboarding flow to create a testable Resilience Testing service.

---



## Run the Chaos Experiment



### Step 1: Create a New Experiment

1. Navigate to **Resilience Testing → Chaos Experiments**.
2. Click **+ New Experiment**.
3. Enter the experiment name: `pod-network-latency-resilience-demo`.
4. Select the **Resilience Testing Infrastructure** you created above.
5. Click **Next** to open the **Chaos Studio** (blank canvas).



### Step 2: Add the Pod Network Latency Fault

1. Click the **+** icon in the Studio to add a fault.
2. Search for **Kubernetes → Pod Network Latency**.
3. Configure:

  | Parameter         | Value                 |
  | ----------------- | --------------------- |
  | Namespace         | `chaos-demo`          |
  | Label Selector    | `app=resilience-demo` |
  | Network Latency   | `2s`                  |
  | Duration          | `30s`                 |
  | Pods Affected (%) | `50`                  |

4. Click **Apply Changes**.



### Step 3: Attach an HTTP Probe

1. In the fault configuration, go to the **Probes** tab.
2. Click **+ Add Probe** → **HTTP Probe**.
3. Configure:

  | Parameter         | Value                                                     |
  | ----------------- | --------------------------------------------------------- |
  | Probe Name        | `frontend-health-check`                                   |
  | URL               | `http://resilience-demo-svc.chaos-demo.svc.cluster.local` |
  | Method            | `GET`                                                     |
  | Expected Response | `200`                                                     |

4. Click **Apply Changes**.



### Step 4: Execute and Analyze

1. Click **Run** to start the experiment.
2. Monitor the execution:
  - Observe that target pods remain running while network delay is injected.
  - Watch the HTTP probe status — it should remain green throughout.
3. After completion, review the **Resilience Score**.

---



## Expected Outcome


| Metric               | Expected Value | Meaning                                                    |
| -------------------- | -------------- | ---------------------------------------------------------- |
| **Resilience Score** | 100%           | All probes passed; the service stayed reachable            |
| **Pod Status**       | Running        | Pods are delayed, not deleted or restarted                 |
| **HTTP Probe**       | All Green      | Service remained available during fault injection          |


If the Resilience Score is below 100%, investigate:

- Is the HTTP probe timeout high enough for the injected network latency?
- Are there sufficient replicas so some traffic can avoid delayed pods?
- Is the readiness probe configured correctly?

---



## Cleanup

Remove all resources when you are done:

```bash
kubectl delete namespace chaos-demo
```

---



## Additional Resources

- [Set up Kubernetes chaos infrastructure (DDCR)](https://developer.harness.io/docs/resilience-testing/chaos-testing/infrastructure/kubernetes)
- [Harness Service Discovery](https://developer.harness.io/harness-platform/use-harness-platform/service-discovery)
- [Dedicated delegate approach](https://developer.harness.io/docs/resilience-testing/chaos-testing/infrastructure/kubernetes/dedicated-delegate)
- [Install a Delegate on Kubernetes](https://developer.harness.io/3k-docs/platform/delegates-v2/install-a-delegate/install-kubernetes-delegate/)
- [Kubernetes Pod Network Latency Fault Reference](https://developer.harness.io/docs/chaos-engineering/faults/chaos-faults/kubernetes/pod/pod-network-latency)
- [HTTP Probe Configuration Guide](https://developer.harness.io/docs/chaos-engineering/features/probes/http-probe)

---



## License

This project is licensed under the Apache License 2.0. See the [LICENSE](./LICENSE) file for details.
