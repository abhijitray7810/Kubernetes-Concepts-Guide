# Pod
## Definition:
- smallest unit in kubernetes, wraps one or more containers with shared storage and network.
## Why:
- Small Deployable Unit in Kubernetes.
- Groups Containers that Need to Work Closely Together.
- Shares Network Namespace and storage Volumes.
- containers alone are not first-class citizens in the API.
## when:
- Running a single application Container.
- Running tightly coupled containers(sidecar pattern).
- Testing and Debugging.
- Almost never create pods directly except for quick debugging.
## Yml
```
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod
spec:
  containers:
  - name: app
    image: nginx:1.25
```
## Yml Example
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: web
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```
