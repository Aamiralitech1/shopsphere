# Vprofile Kubernetes Manifests - Deploy Guide

## Pehle image names update karo

Har file mein `<DOCKERHUB_USERNAME>` ko apne actual DockerHub username se
replace karo:
- `02-db-statefulset.yaml`
- `05-app-deployment.yaml`
- `06-web-deployment.yaml`

```bash
sed -i 's/<DOCKERHUB_USERNAME>/your-actual-username/g' *.yaml
```

## Deploy karne ka order (zaroori hai, isi sequence mein)

```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-secrets.yaml
kubectl apply -f 02-db-statefulset.yaml
kubectl apply -f 03-cache-deployment.yaml
kubectl apply -f 04-mq-statefulset.yaml

# DB/cache/MQ ready hone ka wait karo (initContainers app mein bhi wait karenge,
# lekin pehle khud verify kar lo)
kubectl get pods -n vprofile -w

kubectl apply -f 05-app-deployment.yaml
kubectl apply -f 06-web-deployment.yaml
kubectl apply -f 07-hpa.yaml
kubectl apply -f 08-node-monitor-daemonset.yaml
```

## Verify karo - har concept ko live dekho

```bash
# Sab pods dekho
kubectl get pods -n vprofile -o wide

# StatefulSet pods - fixed naam dikhega (vprodb-0, vpromq01-0)
kubectl get statefulset -n vprofile

# Deployment pods - random naam (vproapp-xxxxx, vproweb-xxxxx)
kubectl get deployment -n vprofile

# PVC - sirf StatefulSets ke liye banenge
kubectl get pvc -n vprofile

# DaemonSet - kube-system mein, ek pod per node
kubectl get daemonset -n kube-system

# HPA status - current vs desired replicas
kubectl get hpa -n vprofile

# App pod ke andar 2 containers honge (app + elasticsearch sidecar)
kubectl describe pod -n vprofile -l app=vproapp | grep -A2 "Containers:"
```

## Access karo

```bash
kubectl get svc vproweb -n vprofile
```

`EXTERNAL-IP` mil jaye to browser mein kholo aur `admin_vp` / `admin_vp` se login karo.

## Test karo - data persistence (StatefulSet ka asli proof)

```bash
kubectl delete pod vprodb-0 -n vprofile
# Pod recreate hoga same naam (vprodb-0) ke sath, same PVC attach hoga
# Login data aur tables GAYAB NAHI HONGE
```

## Test karo - HPA scaling (load generate karke)

```bash
kubectl run -i --tty load-generator --rm --image=busybox:1.36 -n vprofile \
  --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://vproapp:8080/login; done"

# Dusri terminal mein dekho replicas badhte hue
kubectl get hpa vproapp-hpa -n vprofile -w
```

## Common issues

- **App pod CrashLoopBackOff**: initContainer logs check karo -
  `kubectl logs <pod> -c wait-for-db -n vprofile` - ho sakta hai DB abhi
  initialize ho raha ho, kuch second wait karo.
- **HPA shows `<unknown>` for targets**: metrics-server install nahi hua,
  upar diya gaya command chalao.
- **Elasticsearch sidecar OOMKilled**: kops node ka memory kam hai to
  ES_JAVA_OPTS kam karo (`-Xms128m -Xmx128m`) ya ES container resources kam karo.
