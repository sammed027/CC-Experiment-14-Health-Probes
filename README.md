# Cloud Computing Experiment 14: Kubernetes Health Checks

A hands-on demonstration of Kubernetes **liveness** and **readiness** probes using a small Flask application packaged with Docker and deployed as two replicas.

## Aim

To deploy a containerized Flask application to Kubernetes, configure HTTP health probes, and observe how Kubernetes handles unhealthy containers and pods that should temporarily stop receiving traffic.

## Learning objectives

- Build a Docker image for a Python Flask application.
- Deploy two replicas using a Kubernetes Deployment.
- Configure liveness and readiness probes.
- Observe pod readiness, restart counts, and Service endpoints.
- Simulate failure and recovery.
- Explain the difference between container recovery and traffic eligibility.

## Technologies

- Python 3.12 and Flask
- Docker
- Kubernetes and kubectl
- Kind (Kubernetes in Docker)

## Directory structure

```text
CC-Experiment-14-Health-Probes/
├── app.py
├── requirements.txt
├── Dockerfile
├── deployment.yaml
├── service.yaml
└── README.md
```

## Liveness vs readiness

| Probe | Question answered | Effect when it fails |
|---|---|---|
| Liveness | Is the application process healthy enough to continue? | After consecutive failures reach the threshold, Kubernetes restarts the container |
| Readiness | Should this pod receive traffic now? | Pod becomes not-ready and is excluded from eligible Service backends; no restart is caused by readiness failure alone |

A process can be alive but not ready. For example, it may still be initializing or temporarily unable to serve requests. Keeping the checks separate avoids unnecessary restarts.

## Application endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/` | Application information |
| GET | `/live` | Liveness check |
| GET | `/ready` | Readiness check |
| POST | `/fail-liveness` | Simulate liveness failure |
| POST | `/fail-readiness` | Simulate readiness failure |
| POST | `/recover` | Restore both health states |

The failure flags are in-memory variables for this lab. Restarting the container starts a new process and resets the flags to healthy.

## Setup and execution

### 1. Create a Kind cluster

Skip cluster creation if you already have a suitable cluster running.

```bash
kind create cluster --name health-lab
kubectl cluster-info
kubectl get nodes
```

### 2. Build and load the Docker image

Run from the directory containing the Dockerfile:

```bash
docker build -t health-probe-demo:v1 .
kind load docker-image health-probe-demo:v1 --name health-lab
```

### 3. Create namespace and deploy

```bash
kubectl create namespace probe-lab
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

If the namespace already exists, do not create it again.

### 4. Verify Kubernetes resources

```bash
kubectl get deployments -n probe-lab
kubectl get pods -n probe-lab -o wide
kubectl get services -n probe-lab
kubectl rollout status deployment/health-probe-app -n probe-lab
```

Expected initial state: two replicas become Running and Ready.

### 5. Access the service

In one terminal:

```bash
kubectl port-forward service/health-probe-service 8081:80 -n probe-lab
```

In another terminal:

```bash
curl http://127.0.0.1:8081/
curl http://127.0.0.1:8081/live
curl http://127.0.0.1:8081/ready
```

## Demonstration A: readiness failure

Use a pod-specific port-forward so the request reaches the exact replica being tested.

1. Find the pod name:

```bash
kubectl get pods -n probe-lab
```

2. Forward a local port to that pod (replace `<pod-name>`):

```bash
kubectl port-forward pod/<pod-name> 8082:8080 -n probe-lab
```

3. From another terminal, make that pod unready:

```bash
curl -X POST http://127.0.0.1:8082/fail-readiness
kubectl get pods -n probe-lab
kubectl get endpointslices -n probe-lab
```

**Expected:** the selected pod becomes not-ready (often shown as `0/1`) but remains Running. Its restart count should not increase. It is removed from the Service's ready backends while the other healthy replica can continue serving.

4. Restore the health state:

```bash
curl -X POST http://127.0.0.1:8082/recover
kubectl get pods -n probe-lab
kubectl get endpointslices -n probe-lab
```

After the next successful readiness check, the pod should become Ready and return to the eligible backends.

## Demonstration B: liveness failure

Keep the pod-specific port-forward active and ensure the selected pod is healthy before testing.

1. Trigger liveness failure:

```bash
curl -X POST http://127.0.0.1:8082/fail-liveness
```

2. Watch pod state:

```bash
kubectl get pods -n probe-lab -w
```

3. Once Kubernetes restarts the failing container, stop watching with Ctrl+C and inspect:

```bash
kubectl get pods -n probe-lab
kubectl describe pod <pod-name> -n probe-lab
```

**Expected:** after the configured consecutive liveness failures, Kubernetes restarts the container. The restart count increases. The new process starts with healthy in-memory state and should become Ready again.

## Probe configuration

| Setting | Liveness | Readiness |
|---|---:|---:|
| HTTP path | `/live` | `/ready` |
| Port | 8080 | 8080 |
| Initial delay | 5 seconds | 3 seconds |
| Period | 5 seconds | 3 seconds |
| Timeout | 2 seconds | 2 seconds |
| Failure threshold | 3 | 2 |

These are demonstration values. Production values should be tuned to the application's startup time and recovery characteristics.

## Evidence to capture

Capture screenshots from your own run and add them to a `screenshots/` folder. Suggested evidence:

1. Docker image build completed.
2. Kind cluster nodes are Ready.
3. Deployment and both pods are running.
4. Kubernetes Service is created.
5. Healthy application, liveness, and readiness endpoint responses.
6. Readiness failure: pod is not-ready while its restart count remains unchanged.
7. EndpointSlices showing the eligible Service backends.
8. Readiness recovery.
9. Liveness failure and increased container restart count.
10. Pod description/events showing liveness probe failures.

Suggested filenames: `01-docker-build.png`, `02-cluster-nodes.png`, `03-pods-ready.png`, `04-service.png`, `05-health-endpoints.png`, `06-readiness-failure.png`, `07-endpointslices.png`, `08-readiness-recovery.png`, `09-liveness-restart.png`, and `10-pod-events.png`.

Do not use another person's screenshots as evidence of your execution. Fill the observation table below only after running the experiment.

## Results and observations

| Test | Expected behavior | Actual observation |
|---|---|---|
| Initial health | Both probes pass | Record after execution |
| Readiness failure | Pod not-ready; no container restart | Record after execution |
| Readiness recovery | Pod returns to Ready | Record after execution |
| Liveness failure | Container restarts after threshold | Record after execution |

## Troubleshooting

- **ImagePullBackOff:** verify the image was built and loaded into the same Kind cluster, and that image name/tag match the Deployment.
- **Pods remain Pending:** run `kubectl describe pod <pod-name> -n probe-lab` and inspect events/resources.
- **Probe failures:** verify that Flask listens on `0.0.0.0:8080` and that the configured paths are correct.
- **Port-forward fails:** confirm the pod/Service exists and use the correct namespace.
- **No liveness restart yet:** allow for the initial delay, probe period, and consecutive failure threshold.

## Conclusion

The experiment demonstrates Kubernetes health monitoring through two independent HTTP probes. Readiness controls whether a pod is eligible to receive Service traffic, while liveness detects a failing container and triggers a restart after repeated failures. Together, the probes improve application availability and recovery behavior in a Kubernetes cluster.

## Viva questions

**1. What is a Kubernetes probe?**  
A periodic check used by Kubernetes to determine the health or availability state of a container.

**2. What does a liveness probe do?**  
It detects an unhealthy or stuck container and can trigger a restart.

**3. What does a readiness probe do?**  
It decides whether a pod should receive traffic through a Service.

**4. Does readiness failure restart a container?**  
No. It marks the pod as not ready and removes it from eligible Service backends.

**5. What happens after repeated liveness failures?**  
Kubernetes restarts the container according to the pod restart policy.

**6. Can an application be live but not ready?**  
Yes. It may be running while still initializing or waiting for a dependency.

**7. Why use two replicas?**  
To show that one healthy replica can continue serving while another becomes temporarily unready.

**8. Why use a pod-specific port-forward for failure testing?**  
It guarantees the test request reaches the intended pod rather than an arbitrary Service backend.

**9. What is the role of a Kubernetes Service?**  
It provides a stable network endpoint and routes traffic to eligible pods selected by labels.

**10. What is the key difference between liveness and readiness?**  
Liveness influences container restarts; readiness influences whether a pod receives traffic.

---

**Originality note:** This experiment is documented as an independently prepared implementation of the lab topic. Validate commands in your own cluster and add your own screenshots before submission.
