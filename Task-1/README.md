\# Task Session 1: Multi-Container Pods (Init Container \& Sidecar)



\## 1. Overview



This task demonstrates Kubernetes multi-container Pods using an Init Container, a Main Container, and a Sidecar Container.



The objective is to deploy an Nginx web application on a 3-node Minikube cluster. The application displays the current timestamp and automatically updates it every 5 seconds without restarting Nginx.



\## 2. Kubernetes Environment



| Component | Configuration |

|---|---|

| Kubernetes | v1.37.0 |

| Cluster | Minikube |

| Nodes | 1 Control Plane + 2 Workers |

| CNI | Calico |

| Container Runtime | containerd |

| Namespace | webapp |

| Deployment | web-deployment |

| Replicas | 3 |



\## 3. Pod Architecture



Each Pod contains three containers:



\### Init Container: init-done

\- Image: busybox:1.36

\- Executes `echo done`.

\- Exits successfully with exit code 0.

\- Ensures initialization completes before application containers start.



\### Main Container: nginx

\- Image: nginx:1.27

\- Listens on port 80.

\- Creates the initial HTML page with the current timestamp.

\- Runs Nginx in foreground mode.



\### Sidecar Container: time-updater

\- Image: busybox:1.36

\- Updates the Nginx HTML page every 5 seconds.

\- Uses `pgrep` to identify the Nginx process.

\- Accesses the Nginx filesystem through `/proc/<pid>/root`.

\- Uses the `SYS\_PTRACE` Linux capability.



\### Shared Process Namespace



The Pod uses:



`shareProcessNamespace: true`



This allows containers within the same Pod to see each other's processes.



\*\*Important:\*\* No application volumes are configured. The HTML file is updated through the shared process namespace and `/proc`.



\## 4. Deployment



The Kubernetes manifest is available in `deployment.yaml`.



Apply the configuration:



```bash

kubectl apply -f deployment.yaml

```



Verify the Deployment:



```bash

kubectl get deployments -n webapp

```



Verify the Pods:



```bash

kubectl get pods -n webapp -o wide

```



Expected result: Three Pods, each with two running application containers.



\## 5. Application Access



Expose the application locally using port forwarding:



```bash

kubectl port-forward -n webapp deploy/web-deployment 8080:80

```



Open:



http://localhost:8080



The webpage displays:



`hello in MLOPS MENA COMMUNITY : <TIMESTAMP>`



The timestamp automatically changes every 5 seconds.



\## 6. Verification



The following commands were used to verify the implementation.



\*\*Check cluster nodes:\*\*



```bash

kubectl get nodes -o wide

```



\*\*Check Init Container logs:\*\*



```bash

kubectl logs deployment/web-deployment -n webapp -c init-done

```



\*\*Check the Nginx process from the Sidecar:\*\*



```bash

kubectl exec deployment/web-deployment -n webapp -c time-updater -- ps

```



\*\*Verify timestamp updates:\*\*



```bash

kubectl exec deployment/web-deployment -n webapp -c nginx -- cat /usr/share/nginx/html/index.html

```



\*\*Verify that no application volumes are configured:\*\*



```bash

kubectl get deployment web-deployment -n webapp -o jsonpath='{.spec.template.spec.volumes}'

```



\## 7. Init Container Failure Experiment



To investigate Kubernetes behavior when an Init Container fails, the initialization command was intentionally modified:



```bash

echo 'init failed'; exit 1

```



After applying the modified configuration:



```bash

kubectl apply -f deployment.yaml

```



The new Pod entered:



`Init:Error` and `Init:CrashLoopBackOff`



The failure was verified using:



```bash

kubectl logs POD\_NAME -n webapp -c init-done

```



And:



```bash

kubectl describe pod POD\_NAME -n webapp

```



The Init Container terminated with exit code 1.



\### Observations



1\. The new Pod could not complete initialization.

2\. Nginx and time-updater did not start in the failed Pod.

3\. The previous healthy Pods remained running during the failed rollout.

4\. The existing application remained accessible through the old Pods.



This experiment demonstrates Init Container execution requirements and Deployment rolling update behavior.



\## 8. Recovery



The original initialization command was restored:



```bash

echo done

```



The corrected configuration was applied:



```bash

kubectl apply -f deployment.yaml

```



The rollout was verified:



```bash

kubectl rollout status deployment/web-deployment -n webapp

```



Final result:



```text

deployment "web-deployment" successfully rolled out

```



All three Pods returned to `2/2 Running`.



\## 9. Screenshots



The `screenshots/` directory contains evidence of:



1\. Three-node Minikube cluster.

2\. Running application Pods.

3\. Successful Init Container execution.

4\. Nginx webpage.

5\. Automatic timestamp updates.

6\. Shared Nginx processes visible from the Sidecar.

7\. Init Container failure.

8\. Failure diagnosis and exit code.

9\. Successful application recovery.



\## 10. Conclusion



Successfully implemented and tested a Kubernetes multi-container application using an Init Container, an Nginx Main Container, and a Sidecar Container.



The task demonstrates shared process namespaces, container coordination, Linux process filesystem access, automatic application updates, Init Container failure handling, and Deployment recovery.



