# ReasonPod - LLM-Driven Reasoning for Intelligent Resource Scheduling in Edge AIoT Systems

**Student:** Sopheanith Ny

**Supervisor:** Dr. Eyhab Al-Masri

**Institution:** University of Washington Tacoma

## Project Description

ReasonPod is a research project that improves how Kubernetes decides which node to run a workload on. The default Kubernetes scheduler picks nodes based on basic resource availability. ReasonPod replaces that with a smarter approach that combines TOPSIS, a multi-criteria decision algorithm, with a locally deployed large language model through Ollama. The LLM generates dynamic weights for five scheduling criteria: execution time, energy consumption, available CPU, available memory, and resource balance. Instead of treating all five criteria equally, ReasonPod adjusts the weights based on the type of workload being scheduled.

So far the baseline is in place. In the spring, the default Kubernetes scheduler was compared against the static TOPSIS scheduler across two workloads on a three-node Minikube cluster. The key finding is that TOPSIS chose a different node than the default scheduler in all twelve runs, which shows that TOPSIS makes deliberate multi-criteria decisions instead of simple resource checks. In October, three new workloads (small, medium, and large) were built as Kubernetes Jobs and run through the TOPSIS scheduler. The LLM weight generation has not been built yet and is planned for later this fall quarter.

## Project Status

| Done | Not started yet |
|---|---|
| Three-node Minikube cluster running and healthy | GKE cluster and migration from Minikube |
| TOPSIS scheduler deployed and scheduling test jobs | Cluster monitoring module |
| Spring baseline: default scheduler vs TOPSIS on two workloads | Ollama LLM reasoning service |
| Small, medium, and large workloads as Kubernetes Jobs | Policy translation module |
| | Dynamic weights inside the TOPSIS scheduler |
| | Full experiments across workload types and competition levels |

## Repository Layout

```
kubernetes/            scheduler-rbac.yaml and scheduler-deployment.yaml (from GreenCube)
workload/
  small_workload/      spring Task 1 pods (linear regression)
  medium_workload/     spring Task 2 pods (CNN simulation)
  jobs_workloads/      October Jobs
    small/             job-small.yaml
    medium/            job-medium.yaml
    large/             job-large.yaml
    topsis-test-job.yaml
data/                  experiment logs and collected metrics
figures/               plots
results/               summary result tables
README.md
```

The scheduler source code (TopsisScheduler.java) is not copied into this repo. It lives in the GreenCube repo, and this project uses the prebuilt scheduler image.

## Code Credit

The TOPSIS scheduler comes from the GreenCube project (https://github.com/nmaislab/GreenCube) by the AERIS Lab. I did not change the scheduling logic.

| Component | Source |
|---|---|
| TopsisScheduler.java and the TOPSIS scoring logic | Reused from GreenCube |
| Scheduler image `ppreejit/topsis-scheduler:v1` | Reused from GreenCube |
| `kubernetes/scheduler-rbac.yaml` and `kubernetes/scheduler-deployment.yaml` | Taken from GreenCube |
| Task annotation values (`task.type`, `task.size`, `task.loc`) | Follow the classes in the GreenCube README |
| Three-node Minikube setup and scheduler deployment | My own work |
| Spring workloads (linear regression and CNN simulation pods) | My own work |
| Small, medium, and large workload Jobs | My own work |
| TOPSIS test job | My own work |
| Experiment runs, measurements, and results | My own work |
| LLM reasoning service, policy translation, dynamic weights | Planned, not built yet |

GreenCube is based on this paper: Preethika Pradeep and Eyhab Al-Masri, "GreenPod: Energy-Optimized Scheduling for AIoT Workloads Using TOPSIS," 2025 IEEE World AI IoT Congress (AIIoT).

## Dataset / Workloads

No external dataset download is needed. Every workload is a self-contained Python script embedded directly inside its YAML file.

### Spring workloads (pods)

**Task 1 - Linear Regression (small workload):**
A Python script that generates 10,000 random data points and computes slope and intercept using ordinary least squares. Resource requests are set to 200m CPU and 128Mi memory.

**Task 2 - CNN Inference Simulation (medium workload):**
A Python script that simulates three convolutional layers with ReLU activations on a 32x32 input followed by a fully connected output layer, written using only the Python standard library. Resource requests are set to 500m CPU and 256Mi memory.

These YAML files are in `workload/small_workload/` and `workload/medium_workload/`.

### October workloads (Kubernetes Jobs)

Three Jobs with clearly different resource demands. Each one uses the `python:3.10-slim` image and the `topsis-scheduler`. The data and model weights are random, so the goal is to put different amounts of load on the cluster and not to be accurate.

| | Small | Medium | Large |
|---|---|---|---|
| What it does | Linear regression on 10,000 points | Linear regression on 500,000 points, learned over 20 passes (gradient descent) | CNN inference on 120 images (32x32) with two convolution layers |
| CPU request / limit | 100m / 500m | 250m / 1000m | 500m / 1000m |
| Memory request / limit | 64Mi / 128Mi | 256Mi / 512Mi | 512Mi / 1Gi |
| Script run time | about 0.01 s | about 4 s | about 17 s |
| Total Job time | 8 to 9 s | 15 to 18 s | 21 to 23 s |
| `task.size` / `task.loc` | small / 110 | scalable / 185 | distributed / 435 |
| File | `workload/jobs_workloads/small/job-small.yaml` | `workload/jobs_workloads/medium/job-medium.yaml` | `workload/jobs_workloads/large/job-large.yaml` |

All three Jobs use `completions: 1`, `backoffLimit: 0`, and `restartPolicy: Never`. The total Job time is longer than the script time because it includes the scheduler wait (the scheduler checks for new pods about every 10 seconds) and the container startup.

There is also a small test Job with three pods (100m CPU, 64Mi memory each) that is used to show the scheduler's node choices in its logs: `workload/jobs_workloads/topsis-test-job.yaml`.

## Required Libraries

The experiments so far do not require any external libraries. All workloads run entirely on Python's built-in modules inside the python:3.10-slim container image.

The following libraries will be required for the full ReasonPod implementation later this quarter:

```bash
pip install kubernetes requests numpy
```

| Library | Purpose |
|---|---|
| kubernetes | Interact with the Kubernetes API programmatically |
| requests | Send prompts to LLM HTTP endpoint |
| numpy | Numerical calculations in the policy translation module |

> Note: PyTorch and TensorFlow are not required. The CNN workloads are implemented using only Python's built-in math and random modules.

## Environment Setup

- Python 3.10 (matches python:3.10-slim container image used in experiments)
- Minikube v1.37.0
- kubectl v1.34.1
- Kubernetes v1.34.0 on all three nodes
- Docker Desktop (required for Minikube driver). Open Docker Desktop and wait until it is running before starting Minikube, or Minikube will give a Docker connection error.

```bash
# Create and activate a virtual environment
python -m venv venv
venv\Scripts\activate        # Windows
source venv/bin/activate     # Mac/Linux

# Install dependencies (for later phases)
pip install kubernetes requests numpy
```

## Running the TOPSIS Scheduler

First make sure your Minikube cluster is running with three nodes:

```bash
minikube start --nodes=3 --driver=docker
kubectl get nodes
```

All three nodes (`minikube`, `minikube-m02`, `minikube-m03`) should show `Ready`.

Deploy the TOPSIS scheduler to the cluster:

```bash
kubectl apply -f kubernetes/scheduler-rbac.yaml
kubectl apply -f kubernetes/scheduler-deployment.yaml
```

Verify the scheduler is running:

```bash
kubectl get pods -n kube-system | grep topsis      # Windows PowerShell: use findstr topsis
```

You should see the topsis-scheduler pod with a Running status.

Run the test Job and check which nodes the scheduler picked:

```bash
kubectl apply -f workload/jobs_workloads/topsis-test-job.yaml
kubectl get pods -o wide
kubectl logs -n kube-system deploy/topsis-scheduler | grep Binding     # Windows PowerShell: Select-String "Binding"
kubectl delete job topsis-test-job
```

Each pod should reach `Completed`, and the scheduler log should print one `Binding pod ... to node ...` line per pod.

## Running Experiments

### Spring experiments

**Task 1 - Linear Regression:**

```bash
# Default scheduler
kubectl apply -f workload/small_workload/linear-regression-default.yaml
kubectl logs linear-regression-default
kubectl delete pod linear-regression-default

# TOPSIS scheduler
kubectl apply -f workload/small_workload/linear-regression-topsis.yaml
kubectl logs linear-regression-topsis
kubectl delete pod linear-regression-topsis
```

**Task 2 - CNN Inference:**

```bash
# Default scheduler
kubectl apply -f workload/medium_workload/cnn-default.yaml
kubectl logs cnn-default
kubectl delete pod cnn-default

# TOPSIS scheduler
kubectl apply -f workload/medium_workload/cnn-topsis.yaml
kubectl logs cnn-topsis
kubectl delete pod cnn-topsis
```

### October workloads (small, medium, large)

Run one Job at a time if you want clean timing, since they share the same machine.

```bash
kubectl apply -f workload/jobs_workloads/small/job-small.yaml
kubectl apply -f workload/jobs_workloads/medium/job-medium.yaml
kubectl apply -f workload/jobs_workloads/large/job-large.yaml

# wait about 45 seconds, then check where each pod ran and how long it took
kubectl get pods -o wide
kubectl get jobs
kubectl logs job/workload-small
kubectl logs job/workload-medium
kubectl logs job/workload-large

# clean up
kubectl delete job workload-small workload-medium workload-large
```

## Output / Results

After each run, capture three things:

```bash
# Completion time printed in pod logs
kubectl logs <pod-name>            # for Jobs: kubectl logs job/<job-name>

# Node placement and scheduling events
kubectl describe pod <pod-name>

# Resource utilization on the assigned node
kubectl top nodes
```

For Jobs, `kubectl get jobs` shows the total Job duration, and the scheduler log shows the node it picked.

All results are recorded manually from the commands above.

### Spring baseline results (means over 3 runs per scheduler)

| Workload | Scheduler | Node chosen (all 3 runs) | Scheduling latency | Completion time | CPU on node | Memory on node |
|---|---|---|---|---|---|---|
| Linear regression | Default | minikube-m02 | about 1 s | 0.0239 s | 43m | 327Mi (4%) |
| Linear regression | TOPSIS | minikube (control plane) | about 2.3 s | 0.0253 s | 109m | 1530Mi (19%) |
| CNN inference | Default | minikube-m02 | about 1 s | 0.2589 s | 30m | 285Mi (3%) |
| CNN inference | TOPSIS | minikube (control plane) | about 1 s | 0.2545 s | 108m | 1127Mi (14%) |

The default scheduler always chose `minikube-m02` and TOPSIS always chose the control plane node, and neither one ever changed its choice. Completion times were almost the same. The higher CPU and memory on the TOPSIS node comes from the control plane running the system pods, not from the workload. The one slow TOPSIS run (about 5 s) came from an image pull.

### October workload results

| Workload | Node chosen | Script run time | Total Job time |
|---|---|---|---|
| Small | minikube | 0.0124 s | 8 s |
| Medium | minikube | 4.0038 s | 15 s |
| Large | minikube | 17.2358 s | 21 s |

These numbers come from one run of each Job on its own. A second run with all three Jobs at once gave nearly the same script times (0.0126 s, 4.3564 s, and 17.4566 s).

### Limitations

- The whole cluster runs on one physical machine through Docker Desktop, so the node a job lands on does not change how fast it runs.
- TOPSIS placed every pod on the control plane node, so changing the workload size has not changed where it places pods yet.
- The TOPSIS weights are still fixed at 0.2 for all five criteria, so this is the static baseline and not the LLM version.
- CPU showed 0% in the spring runs, and live CPU and memory were not captured for the new Jobs, because the jobs finish faster than the metrics server samples (about every 15 seconds). The CPU and memory numbers for the new workloads are the requests and limits from the YAML files.
- The run times come from only a few runs per job, not a large average.

## Reproducing Results

Run these commands in order from a clean cluster.

```bash
# Start cluster
minikube start --nodes=3 --driver=docker
kubectl get nodes

# Deploy TOPSIS scheduler
kubectl apply -f kubernetes/scheduler-rbac.yaml
kubectl apply -f kubernetes/scheduler-deployment.yaml

# Wait for scheduler to be ready
kubectl get pods -n kube-system | grep topsis

# Test Job: confirm TOPSIS schedules pods and logs the node
kubectl apply -f workload/jobs_workloads/topsis-test-job.yaml
kubectl get pods -o wide
kubectl logs -n kube-system deploy/topsis-scheduler | grep Binding
kubectl delete job topsis-test-job

# Small, medium, and large Jobs
kubectl apply -f workload/jobs_workloads/small/job-small.yaml
kubectl apply -f workload/jobs_workloads/medium/job-medium.yaml
kubectl apply -f workload/jobs_workloads/large/job-large.yaml
kubectl get pods -o wide
kubectl get jobs
kubectl logs job/workload-small
kubectl logs job/workload-medium
kubectl logs job/workload-large
kubectl delete job workload-small workload-medium workload-large

# Spring Task 1 - run 3 times per scheduler, delete pod between each run
kubectl apply -f workload/small_workload/linear-regression-default.yaml
kubectl logs linear-regression-default
kubectl describe pod linear-regression-default
kubectl top nodes
kubectl delete pod linear-regression-default

kubectl apply -f workload/small_workload/linear-regression-topsis.yaml
kubectl logs linear-regression-topsis
kubectl describe pod linear-regression-topsis
kubectl top nodes
kubectl delete pod linear-regression-topsis

# Spring Task 2 - run 3 times per scheduler, delete pod between each run
kubectl apply -f workload/medium_workload/cnn-default.yaml
kubectl logs cnn-default
kubectl describe pod cnn-default
kubectl top nodes
kubectl delete pod cnn-default

kubectl apply -f workload/medium_workload/cnn-topsis.yaml
kubectl logs cnn-topsis
kubectl describe pod cnn-topsis
kubectl top nodes
kubectl delete pod cnn-topsis
```

Repeat each spring block three times and record the completion time from logs, node placement from describe, and CPU and memory from top nodes after each run.
