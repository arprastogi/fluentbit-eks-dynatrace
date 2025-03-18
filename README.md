Here’s a **step-by-step guide** to set up an AWS EKS cluster from scratch using the CLI on your EC2 jump server, including the creation of a new node group, RBAC configuration, fine-tuning, sample application deployment, and Fluent Bit configuration.

---

## 🎯 **Step 1: Set Up EC2 Jump Server with Required Pre-requisites**

### ✅ **1.1 Install Required Packages**
Connect to your EC2 jump server and run the following:
```bash
# Update packages
sudo yum update -y

# Install AWS CLI
sudo yum install aws-cli -y

# Install eksctl
curl -LO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz"
tar -xvzf eksctl_Linux_amd64.tar.gz
sudo mv eksctl /usr/local/bin
eksctl version

# Install kubectl
curl -o kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.29.0/2024-02-29/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv kubectl /usr/local/bin
kubectl version --client

# Install Helm (for Fluent Bit deployment later)
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

## 🔐 **Step 2: IAM Role for EC2 Server**

### ✅ **2.1 Create IAM Role for EC2**

1. Go to AWS CLI or AWS Console:
```bash
aws iam create-role --role-name EKSAdminRole --assume-role-policy-document file://trust-policy.json
```
2. Create a `trust-policy.json` file:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
3. Attach necessary permissions:
```bash
aws iam attach-role-policy --role-name EKSAdminRole --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

4. Attach the role to the EC2 instance.

---

## 🚀 **Step 3: Create EKS Cluster Using Existing VPC and Subnets**

### ✅ **3.1 Define Required Variables**
```bash
export CLUSTER_NAME=my-eks-cluster
export REGION=us-east-1
export VPC_ID=vpc-xxxxxxxxxxxx
export SUBNET_IDS=subnet-xxxxxxxxxxxx,subnet-xxxxxxxxxxxx
export NODE_GROUP_NAME=my-node-group
```

### ✅ **3.2 Create EKS Cluster**
```bash
eksctl create cluster \
  --name $CLUSTER_NAME \
  --region $REGION \
  --vpc-private-subnets $SUBNET_IDS \
  --without-nodegroup
```

---

## 🖥️ **Step 4: Create a Node Group with 2 Worker Nodes**

### ✅ **4.1 Create Node Group**
```bash
eksctl create nodegroup \
  --cluster $CLUSTER_NAME \
  --name $NODE_GROUP_NAME \
  --region $REGION \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 3 \
  --node-ami auto
```

---

## 🔗 **Step 5: Connect EC2 to EKS Cluster**

### ✅ **5.1 Update Kubeconfig**
```bash
aws eks update-kubeconfig --region $REGION --name $CLUSTER_NAME
```

### ✅ **5.2 Verify Connection**
```bash
kubectl get nodes
```

---

## 🔒 **Step 6: Configure Role-Based Access Control (RBAC)**

### ✅ **6.1 Create RBAC Policy for Namespace/Service**

1. Create a `rbac-role.yaml` file:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: my-app
  name: app-reader
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-reader-binding
  namespace: my-app
subjects:
- kind: User
  name: app-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: app-reader
  apiGroup: rbac.authorization.k8s.io
```

2. Apply the RBAC configurations:
```bash
kubectl apply -f rbac-role.yaml
```

---

## 🎯 **Step 7: Deploy Two Sample Services in Different Namespaces**

### ✅ **7.1 Create Two Namespaces**
```bash
kubectl create namespace app1
kubectl create namespace app2
```

### ✅ **7.2 Deploy Sample Applications**

1. **App 1 Deployment (Namespace: app1)**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1-deployment
  namespace: app1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        image: nginx
```
```bash
kubectl apply -f app1-deployment.yaml
```

2. **App 2 Deployment (Namespace: app2)**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2-deployment
  namespace: app2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: app2
        image: httpd
```
```bash
kubectl apply -f app2-deployment.yaml
```

---

## 📄 **Step 8: Configure Fluent Bit in a New Namespace**

### ✅ **8.1 Create Fluent Bit Namespace**
```bash
kubectl create namespace logging
```

### ✅ **8.2 Deploy Fluent Bit via Helm**
```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

# Install Fluent Bit
helm install fluent-bit fluent/fluent-bit \
  --namespace logging \
  --set config.outputs="[OUTPUT]\n    Name http\n    Match *\n    Host <logstash-endpoint>\n    Port 5044\n    URI /\n    Format json\n"
```

### ✅ **8.3 Verify Fluent Bit Pods**
```bash
kubectl get pods -n logging
```

---

## 📚 **Step 9: Configure Fluent Bit to Access Logs from All Namespaces**

### ✅ **9.1 Modify Fluent Bit Config**
Edit the Fluent Bit `values.yaml`:
```yaml
parsers: |
  [PARSER]
      Name   json
      Format json

input: |
  [INPUT]
      Name              tail
      Path              /var/log/containers/*.log
      Parser            json
      Tag               kube.*
      Refresh_Interval  5
      Mem_Buf_Limit     5MB
      Skip_Long_Lines   On
      DB                /fluent-bit/tail-db/tail-containers.db
      DB.Sync           Normal

output: |
  [OUTPUT]
      Name              http
      Match             *
      Host              <logstash-endpoint>
      Port              5044
      URI               /
      Format            json
```

### ✅ **9.2 Upgrade Fluent Bit with Updated Config**
```bash
helm upgrade fluent-bit fluent/fluent-bit --namespace logging -f values.yaml
```

---

## 📊 **Step 10: Verify and Monitor Logs**
### ✅ **10.1 Verify Fluent Bit is Forwarding Logs**
```bash
kubectl logs -n logging <fluent-bit-pod-name>
```

### ✅ **10.2 Check Logs in Logstash**
Confirm that logs are reaching the Logstash endpoint.

---

## 🎉 **Setup Complete!**

You've successfully:
✅ Created an EKS cluster  
✅ Configured Node Groups and Security  
✅ Applied RBAC for Namespace Security  
✅ Deployed Sample Applications  
✅ Configured Fluent Bit to Collect and Forward Logs to Logstash  

Let me know if you encounter any issues or need help troubleshooting! 🚀


---


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
