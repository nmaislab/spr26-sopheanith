# ReasonPod — LLM-Driven Adaptive Kubernetes Scheduling

**Student:** Sopheanith Ny

**Supervisor:** Dr. Eyhab Al-Masri

**Institution:** University of Washington Tacoma

## Project Description

ReasonPod is a research project that improves how Kubernetes decides which node to run a workload on. The default Kubernetes scheduler picks nodes based on basic resource availability. ReasonPod replaces that with a smarter approach that combines TOPSIS, a multi-criteria decision algorithm, with a locally deployed large language model through Ollama and etc. The LLM generates dynamic weights for five scheduling criteria: execution time, energy consumption, available CPU, available memory, and resource balance. Instead of treating all five criteria equally, ReasonPod adjusts the weights based on the type of workload being scheduled.

This quarter established the behavioral baseline by comparing the default Kubernetes scheduler against the static TOPSIS scheduler across two workload types on a three-node Minikube cluster. The key finding is that TOPSIS consistently chose a different node than the default scheduler across all twelve runs, confirming that TOPSIS makes deliberate multi-criteria decisions rather than simple resource checks. The dynamic LLM weight generation will be integrated during the summer phase.

## Dataset / Workloads

No external dataset download is needed. Both workloads are self-contained Python scripts embedded directly inside the pod YAML files.

**Task 1 - Linear Regression (small workload):**
A Python script that generates 10,000 random data points and computes slope and intercept using ordinary least squares. Resource requests are set to 200m CPU and 128Mi memory.

**Task 2 - CNN Inference Simulation (medium workload):**
A Python script that simulates three convolutional layers with ReLU activations on a 32x32 input followed by a fully connected output layer, written using only the Python standard library. Resource requests are set to 500m CPU and 256Mi memory.

Both YAML files are in the `workloads/` directory.

## Required Libraries

```bash
pip install kubernetes requests numpy
```

| Library | Version |
|------------|---------|
| kubernetes | latest |
| requests | latest |
| numpy | latest |

Note: PyTorch and TensorFlow are not required. The CNN workload is implemented using only Python's built-in math and random modules.

## Environment Setup

- Python 3.12.6
- Minikube v1.37.0
- kubectl v1.34.1
- Docker Desktop (required for Minikube driver)

```bash
# Create and activate a virtual environment
python -m venv venv
venv\Scripts\activate        # Windows
source venv/bin/activate     # Mac/Linux

# Install dependencies
pip install kubernetes requests numpy
```

## Running the TOPSIS Scheduler

First make sure your Minikube cluster is running with at least three nodes:

```bash
minikube start --nodes=3 --driver=docker
```

Deploy the TOPSIS scheduler to the cluster:

```bash
kubectl apply -f kubernetes/scheduler-rbac.yaml
kubectl apply -f kubernetes/scheduler-deployment.yaml
```

Verify the scheduler is running:

```bash
kubectl get pods -n kube-system | grep topsis
```

You should see the topsis-scheduler pod with a Running status.

## Running Experiments

**Task 1 - Linear Regression:**

```bash
# Default scheduler
kubectl apply -f workloads/small_workload/linear-regression-default.yaml
kubectl logs linear-regression-default
kubectl delete pod linear-regression-default

# TOPSIS scheduler
kubectl apply -f workloads/small_workload/linear-regression-topsis.yaml
kubectl logs linear-regression-topsis
kubectl delete pod linear-regression-topsis
```

**Task 2 - CNN Inference:**

```bash
# Default scheduler
kubectl apply -f workloads/medium_workload/cnn-default.yaml
kubectl logs cnn-default
kubectl delete pod cnn-default

# TOPSIS scheduler
kubectl apply -f workloads/medium_workload/cnn-topsis.yaml
kubectl logs cnn-topsis
kubectl delete pod cnn-topsis
```

## Output / Results

After each run, capture three things:

```bash
# Completion time — printed in pod logs
kubectl logs <pod-name>

# Node placement and scheduling events
kubectl describe pod <pod-name>

# Resource utilization on the assigned node
kubectl top nodes
```

Raw experiment logs are stored in `data/` as CSV files. The summary results table with means and standard deviations across all runs is in `results/summary_results.csv`.

## Reproducing Results

Run these commands in order from a clean cluster to reproduce the full results table:

```bash
# Start cluster
minikube start --nodes=3 --driver=docker

# Deploy TOPSIS scheduler
kubectl apply -f kubernetes/scheduler-rbac.yaml
kubectl apply -f kubernetes/scheduler-deployment.yaml

# Wait for scheduler to be ready
kubectl get pods -n kube-system | grep topsis

# Task 1 - run 3 times per scheduler, delete pod between each run
kubectl apply -f workloads/small_workload/linear-regression-default.yaml
kubectl logs linear-regression-default
kubectl describe pod linear-regression-default
kubectl top nodes
kubectl delete pod linear-regression-default

kubectl apply -f workloads/small_workload/linear-regression-topsis.yaml
kubectl logs linear-regression-topsis
kubectl describe pod linear-regression-topsis
kubectl top nodes
kubectl delete pod linear-regression-topsis

# Task 2 - run 3 times per scheduler, delete pod between each run
kubectl apply -f workloads/medium_workload/cnn-default.yaml
kubectl logs cnn-default
kubectl describe pod cnn-default
kubectl top nodes
kubectl delete pod cnn-default

kubectl apply -f workloads/medium_workload/cnn-topsis.yaml
kubectl logs cnn-topsis
kubectl describe pod cnn-topsis
kubectl top nodes
kubectl delete pod cnn-topsis
```

Repeat each block three times and record the completion time from logs, node placement from describe, and CPU and memory from top nodes after each run.
