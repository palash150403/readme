# AWX-Installation

1. Create a VM with 4 CPU and min 8 GB RAM (F4s v2)

2. Install Kubectl, docker, Make docker sudo user, Minikube

3. Start minikube

```
minikube start --cpus=4 --memory=6g --addons=ingress
```

4. make dir minikube
```
mkdir minikube
cd minikube
```

5. make a aliase

```
alias kubectl="minikube kubectl --"
```

6. Make Kustomization Files

```
vim kustomization.yaml
```
```
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - github.com/ansible/awx-operator/config/default?ref=2.19.1
images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.19.1
namespace: awx
```
Use latest tags from https://github.com/ansible/awx-operator/tags

```
kubectl apply -k .
```

```
vim awx-server.yaml
```

```
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx-server
spec:
  service_type: nodeport
```
```
vim kustomization.yaml
```
add - awx-server.yaml in file
```
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - github.com/ansible/awx-operator/config/default?ref=2.19.1
  - awx-server.yaml
images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.19.1
namespace: awx
```

7. Run Kustomization

```
kubectl apply -k .
```

8. check for all svc and pods

```
kubectl get svc -n awx
NAME                                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
awx-operator-controller-manager-metrics-service   ClusterIP   10.110.54.158   <none>        8443/TCP       89m
awx-server-postgres-15                            ClusterIP   None            <none>        5432/TCP       88m
awx-server-service                                NodePort    10.99.30.190    <none>        80:31656/TCP   88m


kubectl get pods -n awx
NAME                                               READY   STATUS      RESTARTS       AGE
awx-operator-controller-manager-58b7c97f4b-tbxqh   2/2     Running     2 (31m ago)    89m
awx-server-migration-24.6.1-h46m9                  0/1     Completed   0              86m
awx-server-postgres-15-0                           1/1     Running     1 (30m ago)    88m
awx-server-task-6f8dfc7cb5-5m7n8                   4/4     Running     4 (30m ago)    88m
awx-server-web-6d4fdb8f87-rhzc7                    3/3     Running     3 (105s ago)   88m
```

9. Get minikube ip

```
minikube ip
```

10. Now do port forwarding

```
kubectl port-forward -n awx service/awx-server-service --address 0.0.0.0 31656:80
```

```
http://<vm-ip>:31656/
```

11. Get Token Password
```
kubectl get secret awx-server-admin-password -o jsonpath="{.data.password}" -n awx | base64 -d; echo
```
12. Login Details
```
username: admin
password: token
```
## Refs

https://www.youtube.com/watch?v=n3SzwzbxfRE
https://www.linuxtechi.com/install-ansible-awx-on-ubuntu/
https://bobcares.com/blog/setup-ansible-awx-on-ubuntu/
https://ansible.readthedocs.io/projects/awx-operator/en/latest/installation/basic-install.html