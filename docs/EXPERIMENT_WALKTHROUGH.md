# Chaos Experiment Walkthrough

## Step 1: Open Chaos Studio

1. Log in to your Harness account.
2. Navigate to the **Resilience Testing** module from the left sidebar.
3. Go to **Chaos Experiments** → click **+ New Experiment**.
4. Enter a name: `pod-delete-resilience-demo`.
5. Select the **Resilience Testing Infrastructure** you configured during setup (Kubernetes Harness Infrastructure / DDCR).
6. Click **Next** to enter the **Chaos Studio** (blank canvas).

## Step 2: Add the Pod Delete Fault

1. In the Chaos Studio canvas, click the **+** icon to add a new fault.
2. Search for **"Pod Delete"** under the Kubernetes category.
3. Configure the fault with the following parameters:

   | Parameter          | Value              |
   |--------------------|--------------------|
   | Namespace          | `chaos-demo`       |
   | Label Selector     | `app=resilience-demo` |
   | Duration           | `30s`              |
   | Pods Affected (%)  | `50`               |
   | Interval           | `10s`              |

4. Click **Apply Changes**.

## Step 3: Attach an HTTP Probe

1. Within the Pod Delete fault configuration, navigate to the **Probes** tab.
2. Click **+ Add Probe** → select **HTTP Probe**.
3. Configure the probe:

   | Parameter          | Value                                                      |
   |--------------------|------------------------------------------------------------|
   | Probe Name         | `frontend-health-check`                                    |
   | URL                | `http://resilience-demo-svc.chaos-demo.svc.cluster.local`  |
   | Method             | `GET`                                                      |
   | Expected Response  | `200`                                                      |
   | Mode               | `Continuous`                                                |
   | Interval           | `5s`                                                       |

4. Click **Apply Changes**.

## Step 4: Run and Analyze

1. Click **Run** in the top-right corner of the Chaos Studio.
2. Observe the experiment execution:
   - The fault will begin deleting pods in the `chaos-demo` namespace.
   - The HTTP probe will continuously check service availability.
3. After completion, review the **Resilience Score**:
   - **100%** → All probes passed. The system self-healed successfully.
   - **< 100%** → Some probes failed. Investigate pod scheduling, resource limits, or replica count.

## Step 5: Verify Recovery

```bash
# Confirm all 3 replicas are back and running
kubectl get pods -n chaos-demo -l app=resilience-demo

# Check deployment status
kubectl describe deployment resilience-demo -n chaos-demo
```
