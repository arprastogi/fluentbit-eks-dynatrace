# Reference Youtube Link - https://youtu.be/3sMGLmSzCjc?si=OagixCV9ikZqykxT

# fluentbit-eks-dynatrace

To configure Fluent Bit on your existing AWS EKS cluster to forward logs to the Dynatrace endpoint, follow these steps:

### Prerequisites:
- AWS CLI and kubectl installed and configured.
- EKS cluster up and running.
- Dynatrace tenant URL and API token with the necessary permissions.

---

### Step 1: Create a Namespace for Fluent Bit
```bash
kubectl create namespace fluent-bit
```

---

### Step 2: Create a ConfigMap for Fluent Bit
Create a `fluent-bit-config.yaml` file with the following content:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: fluent-bit
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Log_Level     info
        Parsers_File  parsers.conf

    [INPUT]
        Name          tail
        Path          /var/log/containers/*.log
        Parser        docker
        Tag           kube.*
        Refresh_Interval 5
        Mem_Buf_Limit 5MB
        Skip_Long_Lines On

    [FILTER]
        Name          kubernetes
        Match         kube.*
        Kube_URL      https://kubernetes.default.svc.cluster.local
        Merge_Log     On
        K8S-Logging.Exclude Off
        K8S-Logging.Parser On

    [OUTPUT]
        Name          http
        Match         *
        Host          <DYNATRACE_ENDPOINT>
        Port          443
        URI           /api/v2/logs/ingest
        Format        json
        tls           On
        HTTP_User     none
        HTTP_Passwd   <DYNATRACE_API_TOKEN>
        Header        Content-Type application/json
        Header        Authorization "Api-Token <DYNATRACE_API_TOKEN>"
```

Replace `<DYNATRACE_ENDPOINT>` and `<DYNATRACE_API_TOKEN>` with your Dynatrace details.

Apply the ConfigMap:
```bash
kubectl apply -f fluent-bit-config.yaml
```

---

### Step 3: Deploy Fluent Bit DaemonSet
Create a `fluent-bit-daemonset.yaml` file:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: fluent-bit
  labels:
    app: fluent-bit
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:latest
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: config
          mountPath: /fluent-bit/etc/
          readOnly: true
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "<DYNATRACE_ENDPOINT>"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "443"
        - name: FLUENT_ELASTICSEARCH_USER
          value: "none"
        - name: FLUENT_ELASTICSEARCH_PASSWORD
          value: "<DYNATRACE_API_TOKEN>"
        resources:
          limits:
            memory: 200Mi
            cpu: 200m
          requests:
            memory: 100Mi
            cpu: 100m
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: config
        configMap:
          name: fluent-bit-config
```

Apply the DaemonSet:
```bash
kubectl apply -f fluent-bit-daemonset.yaml
```

---

### Step 4: Create Service Account and Role Binding
```bash
kubectl create serviceaccount fluent-bit -n fluent-bit
kubectl create clusterrolebinding fluent-bit-role \
  --clusterrole=cluster-admin \
  --serviceaccount=fluent-bit:fluent-bit
```

---

### Step 5: Verify Fluent Bit Logs
```bash
kubectl logs -n fluent-bit -l app=fluent-bit
```

---

### Step 6: Validate in Dynatrace
- Navigate to **Dynatrace > Logs & Events**.
- Check if logs are being ingested from EKS.

---

### Additional Recommendations:
- Implement Role-Based Access Control (RBAC) for security.
- Fine-tune the Fluent Bit configuration for specific namespaces or services.
- Use Fluent Bit plugins for additional data enrichment.

Let me know if you'd like a complete Helm-based deployment approach or enhancements.


### **Step 1: Implement Role-Based Access Control (RBAC) for Fluent Bit**

Since Fluent Bit needs access to Kubernetes metadata, we must assign appropriate permissions to access the API server securely.

---

### **Create a Fluent Bit Service Account, Role, and RoleBinding**

Create a `fluent-bit-rbac.yaml` file:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: fluent-bit
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit-role
rules:
  - apiGroups: [""]
    resources:
      - pods
      - namespaces
      - nodes
    verbs:
      - get
      - list
      - watch
  - apiGroups: ["apps"]
    resources:
      - replicasets
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluent-bit-role-binding
subjects:
  - kind: ServiceAccount
    name: fluent-bit
    namespace: fluent-bit
roleRef:
  kind: ClusterRole
  name: fluent-bit-role
  apiGroup: rbac.authorization.k8s.io
```

### **Apply the RBAC Configuration:**
```bash
kubectl apply -f fluent-bit-rbac.yaml
```

---

## ✅ **Now Fluent Bit has secure access to Kubernetes API to fetch metadata.**

---

## **Step 2: Fine-Tune Fluent Bit to Collect Logs Only from Specific Namespaces or Services**

We can control log collection based on Kubernetes namespace or service labels using Fluent Bit's `FILTER` and `INPUT` configuration.

---

### **Update Fluent Bit ConfigMap (`fluent-bit-config.yaml`)**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: fluent-bit
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush        5
        Log_Level    info
        Parsers_File parsers.conf

    [INPUT]
        Name         tail
        Path         /var/log/containers/*.log
        Tag          kube.*
        Parser       docker
        Refresh_Interval 5
        Mem_Buf_Limit 5MB
        Skip_Long_Lines On

    [FILTER]
        Name         kubernetes
        Match        kube.*
        Kube_URL     https://kubernetes.default.svc.cluster.local
        Merge_Log    On
        K8S-Logging.Exclude Off
        K8S-Logging.Parser On

    # Filter logs only from a specific namespace (e.g., 'payments' namespace)
    [FILTER]
        Name         grep
        Match        kube.*
        Regex        kubernetes['namespace_name'] payments

    # Filter logs only from a specific service (e.g., 'orders-service')
    [FILTER]
        Name         grep
        Match        kube.*
        Regex        kubernetes['labels']['app'] orders-service

    [OUTPUT]
        Name         http
        Match        *
        Host         <DYNATRACE_ENDPOINT>
        Port         443
        URI          /api/v2/logs/ingest
        Format       json
        tls          On
        Header       Content-Type application/json
        Header       Authorization "Api-Token <DYNATRACE_API_TOKEN>"
```

---

### ✅ This configuration will:
- Collect logs **only from the `payments` namespace**.
- Forward logs **only from `orders-service`** running in that namespace.

---

### **Apply the Updated ConfigMap**
```bash
kubectl apply -f fluent-bit-config.yaml
```

---

### **Step 3: Restart Fluent Bit DaemonSet**
```bash
kubectl rollout restart daemonset fluent-bit -n fluent-bit
```

---

## 🎯 **Additional Fine-Tuning Options**
| Filter Type           | Purpose                                   |
|----------------|------------------------------------|
| `Grep Filter`     | Filter based on specific log patterns |
| `Record Modifier Filter` | Add custom tags or fields |
| `Throttle Filter` | Limit the log ingestion rate |
| `Lua Script Filter` | Advanced filtering logic with custom scripts |

---

## ✅ **Final Architecture Flow**
```
AWS EKS Cluster
├── Namespace: payments
│    ├── orders-service
│    ├── Fluent Bit DaemonSet
│    ├── Fluent Bit RBAC
│    ├── Filtered Logs -> Dynatrace Endpoint
```

---

## 🔥 **Next Steps**
- Should I help you with setting up Fluent Bit Helm Chart for Production-Ready Deployment?
- Do you want me to show how to segregate logs based on different microservices and forward them to different Dynatrace environments?

Let me know how you'd like to proceed! 😊


### 🎯 **Step 4: Deploy Fluent Bit using Helm for Production-Ready Deployment**

Using Helm makes Fluent Bit deployment more scalable and manageable across multiple namespaces and services in AWS EKS.

---

## ✅ **Step 4.1: Install Fluent Bit Helm Chart**
Add Fluent Bit Helm repo:

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
```

---

## ✅ **Step 4.2: Create a Helm `values.yaml` for Fluent Bit Configuration**

Create a file `fluentbit-values.yaml`:

```yaml
# Fluent Bit Helm Chart Config for Dynatrace Integration

serviceAccount:
  create: true
  name: fluent-bit

rbac:
  create: true

input:
  tail:
    enabled: true
    path: /var/log/containers/*.log
    parser: docker
    tag: kube.*

filter:
  kubernetes:
    enabled: true
    match: kube.*
    merge_log: on
    k8s_logging_parser: on
    k8s_logging_exclude: off

  grep:
    enabled: true
    match: kube.*
    regex:
      - "kubernetes['namespace_name'] payments"   # Filter logs only from 'payments' namespace
      - "kubernetes['labels']['app'] orders-service"  # Filter logs only from 'orders-service'

output:
  http:
    enabled: true
    host: "<DYNATRACE_ENDPOINT>"
    port: 443
    uri: /api/v2/logs/ingest
    format: json
    tls: on
    header:
      - "Authorization: Api-Token <DYNATRACE_API_TOKEN>"
      - "Content-Type: application/json"
```

---

## ✅ **Step 4.3: Install Fluent Bit Helm Chart with Custom Values**
```bash
helm install fluent-bit fluent/fluent-bit \
  -n fluent-bit \
  -f fluentbit-values.yaml
```

---

## ✅ **Step 4.4: Validate Fluent Bit Logs**
```bash
kubectl logs -n fluent-bit -l app.kubernetes.io/name=fluent-bit
```

---

## ✅ **Step 5: Segregate Logs from Different Microservices & Forward to Different Dynatrace Environments**

In `fluentbit-values.yaml`, you can **route logs from different namespaces to different Dynatrace tenants.**

### Example:

```yaml
# Forward logs from 'payments' namespace to Dynatrace-1
output:
  http:
    - name: payments-logs
      match: kube.*
      host: "<DYNATRACE_ENV1_ENDPOINT>"
      port: 443
      uri: /api/v2/logs/ingest
      tls: on
      header:
        - "Authorization: Api-Token <DYNATRACE_API1_TOKEN>"
        - "Content-Type: application/json"

# Forward logs from 'inventory' namespace to Dynatrace-2
    - name: inventory-logs
      match: kube.*
      host: "<DYNATRACE_ENV2_ENDPOINT>"
      port: 443
      uri: /api/v2/logs/ingest
      tls: on
      header:
        - "Authorization: Api-Token <DYNATRACE_API2_TOKEN>"
        - "Content-Type: application/json"
```

---

## ✅ **Step 6: Verify in Dynatrace**

- Go to **Dynatrace UI → Logs & Events**  
- Check the logs being ingested from AWS EKS.  
- Validate the segregation of logs based on different namespaces/services.

---

## 🎯 **Final Architecture for Fluent Bit in EKS**
```
AWS EKS Cluster
├── Namespace: payments → Forward to Dynatrace-1
├── Namespace: inventory → Forward to Dynatrace-2
├── Namespace: monitoring → Forward to Dynatrace-3
├── Fluent Bit DaemonSet (via Helm)
├── Fluent Bit RBAC
└── Fluent Bit ConfigMap
```

---

## ✅ **Next Level**
- Do you want me to show how to create Fluent Bit custom dashboards in Dynatrace?
- Or should I help you with setting up **AWS CloudWatch as a fallback in case Dynatrace endpoint is down**?

Let me know how you'd like to proceed! 😊
------------------------------------------------------------------------------------------------------------------------------

### ✅ **Step 4: Segregate Logs Based on Microservices and Forward to the Same Dynatrace Environment**

---

## 🎯 **Approach:**
We'll use **Fluent Bit's `Match` and `Tag` filtering mechanism** to segregate logs based on different microservices (e.g., `auth-service`, `payments-service`, `orders-service`) and forward them to Dynatrace in a structured format.

---

## **Step 1: Update Fluent Bit ConfigMap (`fluent-bit-config.yaml`)**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: fluent-bit
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush        5
        Log_Level    info
        Parsers_File parsers.conf

    # Input to collect all logs from EKS pods
    [INPUT]
        Name         tail
        Path         /var/log/containers/*.log
        Parser       docker
        Tag          kube.*
        Mem_Buf_Limit 5MB
        Skip_Long_Lines On

    # Kubernetes metadata enrichment
    [FILTER]
        Name         kubernetes
        Match        kube.*
        Kube_URL     https://kubernetes.default.svc.cluster.local
        Merge_Log    On
        K8S-Logging.Exclude Off
        K8S-Logging.Parser On

    # Filter for auth-service logs
    [FILTER]
        Name         grep
        Match        kube.*
        Regex        kubernetes['labels']['app'] auth-service

    [OUTPUT]
        Name         http
        Match        kube.*
        Host         <DYNATRACE_ENDPOINT>
        Port         443
        URI          /api/v2/logs/ingest
        Format       json
        tls          On
        Header       Content-Type application/json
        Header       Authorization "Api-Token <DYNATRACE_API_TOKEN>"
        Json_Key     service_name
        Json_Value   auth-service

    # Filter for payments-service logs
    [FILTER]
        Name         grep
        Match        kube.*
        Regex        kubernetes['labels']['app'] payments-service

    [OUTPUT]
        Name         http
        Match        kube.*
        Host         <DYNATRACE_ENDPOINT>
        Port         443
        URI          /api/v2/logs/ingest
        Format       json
        tls          On
        Header       Content-Type application/json
        Header       Authorization "Api-Token <DYNATRACE_API_TOKEN>"
        Json_Key     service_name
        Json_Value   payments-service

    # Filter for orders-service logs
    [FILTER]
        Name         grep
        Match        kube.*
        Regex        kubernetes['labels']['app'] orders-service

    [OUTPUT]
        Name         http
        Match        kube.*
        Host         <DYNATRACE_ENDPOINT>
        Port         443
        URI          /api/v2/logs/ingest
        Format       json
        tls          On
        Header       Content-Type application/json
        Header       Authorization "Api-Token <DYNATRACE_API_TOKEN>"
        Json_Key     service_name
        Json_Value   orders-service
```

---

### ✅ **Explanation:**
| Microservice       | Fluent Bit Filter              | Output to Dynatrace |
|----------------|---------------------------------|--------------------|
| `auth-service`   | Filter based on app label   | Sends logs with `service_name: auth-service` |
| `payments-service` | Filter based on app label   | Sends logs with `service_name: payments-service` |
| `orders-service`  | Filter based on app label   | Sends logs with `service_name: orders-service` |

---

### **Step 2: Apply Updated Fluent Bit ConfigMap**
```bash
kubectl apply -f fluent-bit-config.yaml
```

---

### **Step 3: Restart Fluent Bit DaemonSet**
```bash
kubectl rollout restart daemonset fluent-bit -n fluent-bit
```

---

## ✅ **Logs Segregated Successfully in Dynatrace! 🎯**
In Dynatrace:

| Service Name      | Logs Visible in Dynatrace |
|----------------|----------------------------|
| auth-service         | ✅ |
| payments-service | ✅ |
| orders-service     | ✅ |

---

## 🎯 **Optional Enhancements:**
| Feature              | Fluent Bit Plugin |
|-----------------|---------------------------|
| Add Custom Tags    | `record_modifier` |
| Filter Specific Pod Names  | `grep` |
| Throttle Specific Microservice Logs | `throttle` |
| Add Environment Info (e.g., `dev`, `staging`, `prod`) | `record_modifier` |

---

## 🚀 **Next Step**
Would you like me to now show **Fluent Bit Helm Chart Deployment for Production-Ready Setup with Best Practices?**
