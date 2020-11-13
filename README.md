# starboard-trivy-tutorial

### 1. Create a K8S cluster
For this tutorial, I used a simple 3 node K8S cluster on DigitalOcean, but it works as well on Docker for Mac and I guess on any other environment
Just to make sure access to the cluster is available
```
kubectl get nodes
```

### 2. Installing the Starboard operator
Clone the github repo
```
git clone https://github.com/aquasecurity/starboard.git
```
First install the custom resources for the starboard operator
```
kubectl apply -f deploy/crd/vulnerabilityreports.crd.yaml
```
and the necessary resources to run the operator
```
kubectl apply -f deploy/static/01-starboard-operator.ns.yaml \
    -f deploy/static/02-starboard-operator.sa.yaml \
    -f deploy/static/03-starboard-operator.clusterrole.yaml \
    -f deploy/static/04-starboard-operator.clusterrolebinding.yaml
```
### Create an operator
To quickly scan the default namespace
```
kubectl apply -f deploy/static/05-starboard-operator.deployment.yaml
```
but if you want to customize more
```
export NS="mydemo"

kubectl apply -f - <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: starboard-operator
  namespace: starboard-operator
  labels:
    app: starboard-operator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: starboard-operator
  template:
    metadata:
      labels:
        app: starboard-operator
    spec:
      serviceAccountName: starboard-operator
      automountServiceAccountToken: true
      securityContext:
        runAsNonRoot: true
        runAsUser: 10000
        fsGroup: 10000
      containers:
        - name: operator
          image: docker.io/aquasec/starboard-operator:0.6.0
          imagePullPolicy: IfNotPresent
          securityContext:
            privileged: false
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
          env:
            - name: OPERATOR_NAMESPACE
              value: "starboard-operator"
            - name: OPERATOR_TARGET_NAMESPACES
              value: $NS
            - name: OPERATOR_CONCURRENT_SCAN_JOBS_LIMIT
              value: "3"
            - name: OPERATOR_SCAN_JOB_RETRY_AFTER
              value: "30s"
            - name: OPERATOR_SCANNER_TRIVY_ENABLED
              value: "true"
            - name: OPERATOR_SCANNER_TRIVY_VERSION
              value: "0.11.0"
            - name: OPERATOR_SCANNER_AQUA_CSP_ENABLED
              value: "false"
            - name: OPERATOR_SCANNER_AQUA_CSP_VERSION
              valueFrom:
                secretKeyRef:
                  name: starboard-operator
                  key: OPERATOR_SCANNER_AQUA_CSP_VERSION
                  optional: true
            - name: OPERATOR_SCANNER_AQUA_CSP_HOST
              valueFrom:
                secretKeyRef:
                  name: starboard-operator
                  key: OPERATOR_SCANNER_AQUA_CSP_HOST
                  optional: true
            - name: OPERATOR_SCANNER_AQUA_CSP_USER
              valueFrom:
                secretKeyRef:
                  name: starboard-operator
                  key: OPERATOR_SCANNER_AQUA_CSP_USERNAME
                  optional: true
            - name: OPERATOR_SCANNER_AQUA_CSP_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: starboard-operator
                  key: OPERATOR_SCANNER_AQUA_CSP_PASSWORD
                  optional: true
            - name: OPERATOR_METRICS_BIND_ADDRESS
              value: ":8080"
            - name: OPERATOR_HEALTH_PROBE_BIND_ADDRESS
              value: ":9090"
            - name: OPERATOR_LOG_DEV_MODE
              value: "false"
          ports:
            - name: metrics
              containerPort: 8080
            - name: probes
              containerPort: 9090
          readinessProbe:
            httpGet:
              path: /readyz/
              port: probes
            initialDelaySeconds: 5
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /healthz/
              port: probes
            initialDelaySeconds: 5
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 10
EOF
```




```
kubectl create ns $NS


kubectl apply -n $NS -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```
```
kubectl get po -n $NS
```
```
kubectl get vulnerablitiesreport -n $NS
```
```
kubectl get vulnerabilityreport  -n $NS
```
```
kubectl get vulnerabilityreport  -n $NS replicaset-nginx-deployment-7848d4b86f-nginx -o yaml

```
