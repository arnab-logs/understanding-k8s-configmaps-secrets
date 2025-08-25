
---

````markdown
# Learning Kubernetes: ConfigMaps & Secrets  

![Kubernetes](https://img.shields.io/badge/Kubernetes-326ce5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Minikube](https://img.shields.io/badge/Minikube-FFCB2B?style=for-the-badge&logo=kubernetes&logoColor=black)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)

This repository documents my learning journey with **ConfigMaps** and **Secrets** in Kubernetes.  
Here, I explore how to inject configuration into Pods via environment variables, mount them as files, and manage sensitive information with Secrets.  
````
---

## 📌 ConfigMaps  

### 1. Creating a ConfigMap  

We start by creating a `cm.yaml`:  

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-cm
data:
  db-port: "3306"
````

Apply it:

```bash
kubectl apply -f cm.yaml
```

⚠️ Always use `apply` instead of `create`.

* `create` will fail if the resource already exists.
* `apply` will update the resource if it exists, or create a new one if it doesn’t.

Check ConfigMaps:

```bash
kubectl get cm
kubectl describe cm test-cm
```

---

### 2. Using ConfigMap as Environment Variables

We modify `deployment.yaml` to pull values from the ConfigMap:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-python-app
  labels:
    app: sample-python-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-python-app
  template:
    metadata:
      labels:
        app: sample-python-app
    spec:
      containers:
        - name: python-app
          image: arnablogs/python-sample-app-demo:v1
          env:
            - name: DB_PORT
              valueFrom:
                configMapKeyRef:
                  name: test-cm
                  key: db-port
          ports:
            - containerPort: 8000
```

Apply the Deployment:

```bash
kubectl apply -f deployment.yaml
```

Exec into a pod and check env vars:

```bash
kubectl exec -it <pod-name> -- /bin/bash
env | grep DB_PORT
```

✅ You should see `DB_PORT=3306`.

---

### 3. Problem with Env Vars & Solution

If we later update `cm.yaml` (e.g., change `db-port: "3306"` → `db-port: "3307"`) and apply it, the Pods will **not pick up the new value** automatically.

* Environment variables inside containers are immutable.
* To see the new value, Pods must be restarted.

But in production, restarting Pods may cause traffic loss.

👉 Solution: Use **VolumeMounts** instead of env variables.

---

### 4. Using ConfigMap as VolumeMounts

Updated `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-python-app
  labels:
    app: sample-python-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-python-app
  template:
    metadata:
      labels:
        app: sample-python-app
    spec:
      containers:
        - name: python-app
          image: arnablogs/python-sample-app-demo:v1
          volumeMounts:
            - name: db-connection
              mountPath: /opt
          ports:
            - containerPort: 8000
      volumes:
        - name: db-connection
          configMap:
            name: test-cm
```

Apply the Deployment:

```bash
kubectl apply -f deployment.yaml
```

Now check inside the Pod:

```bash
kubectl exec -it <pod-name> -- /bin/bash
ls /opt
cat /opt/db-port
```

✅ Output will be `3306`.

If we update the ConfigMap (`3306` → `3307`), the mounted file in `/opt/db-port` updates **live**, without restarting the Pod. 🎉

---

## 🔐 Secrets

### 1. Creating a Secret

```bash
kubectl create secret generic test-secret --from-literal=db-port="3306"
```

Check the Secret:

```bash
kubectl describe secret test-secret
```

Output will show:

```
db-port:  <redacted>
```

### 2. Editing a Secret

```bash
kubectl edit secret test-secret
```

Kubernetes stores Secrets in **base64 encoding**. Example:

```
db-port: MzMwNg==
```

Decode it:

```bash
echo "MzMwNg==" | base64 --decode
3306
```

⚠️ Note: Base64 is **not encryption**, just encoding.
For stronger security, tools like **HashiCorp Vault** or KMS should be used to encrypt secrets at rest.

---

## 📝 Summary

* **ConfigMaps** → Store non-sensitive config (env vars, config files).
* **Secrets** → Store sensitive info (passwords, tokens, certs).
* Use **env vars** for static configs.
* Use **VolumeMounts** for dynamic configs that can change without Pod restarts.
* Secrets are base64-encoded by default → use external tools for stronger encryption.

---

