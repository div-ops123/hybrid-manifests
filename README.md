# YouTube Clone Application and Spring PetClinic Sample Application Kubernetes Manifest

# Repository Structure

```text
~/eks-cluster-manifests/
├── youtube-clone-manifest/
│   ├── deployment.yaml       # YouTube Clone Deployment
│   ├── service.yaml          # YouTube Clone Service
│   ├── ingress.yaml          # YouTube Clone Ingress
├── java-mysql-manifest/
│   ├── java-deployment.yaml  # Java App Deployment
│   ├── java-service.yaml     # Java App Service
│   ├── java-ingress.yaml     # Java App Ingress
│   ├── mysql-statefulset.yaml # MySQL StatefulSet
│   ├── mysql-service.yaml    # MySQL Service
│   ├── mysql-configmap.yaml  # MySQL Configuration
│   ├── mysql-pvc.yaml        # PersistentVolumeClaim for MySQL
│   ├── mysql-cronjob.yaml    # MySQL Backup CronJob
```

**NOTES:**
- Keep Both Apps in prod Namespace
- Use distinct metadata.name and labels to differentiate apps
- Keeps all manifests in one repo for ArgoCD to manage.
- Simplifies Jenkins pipeline updates (single repo for image tag changes).

# YouTube Clone Applicaton

1. Create secret:
`kubectl create secret generic youtube-api-key --from-env-file=api-key.txt -n prod`

1a. Verigy secret:
`kubectl get secret youtube-api-key -n prod -o yaml`

Decode to confirm:
`kubectl get secret youtube-api-key -n prod -o jsonpath='{.data.REACT_APP_RAPID_API_KEY}' | base64 -d`

1b. Delete the file afterwards

---

# Visual Flow of Traffic:
```text
www.youtube-app.com
  ↓ (DNS resolves to ALB)
ALB (in public subnets, port 80/443)
  ↓ (targeting the ClusterIP Service’s virtual IP on port)
Worker Nodes’ ENIs (in private subnets)
  ↓ (kube-proxy routes traffic to ClusterIP Service)
ClusterIP Service (virtual IP, port 80)
  ↓ (load-balances to pods, targetPort 80)
Pods (on worker nodes, container port 80, app port 8080)
```

# ## API Key Management

# React App Environment Variable Consideration:

* ## Build-Time vs. Runtime:
    * **Build-Time:** If your React app uses REACT_APP_RAPID_API_KEY in the frontend code (e.g., fetch('https://api.rapidapi.com', { headers: { 'X-RapidAPI-Key': process.env.REACT_APP_RAPID_API_KEY } })), the key must be available during the Docker build process.

    **Solution:** Rebuild the Docker image with the API key injected
    Remove the env section from deployment.yaml since the key is embedded in the image.

    * **Runtime:** If your app has a backend (e.g., Node.js/Express) that uses REACT_APP_RAPID_API_KEY at runtime (e.g., to make API calls on behalf of the frontend), leave docker image, and deployment.yaml as is.

* ## Check Your App:
* Inspect your app’s code or Dockerfile (devopsdee/youtube-app:v1.0.0) to confirm whether it expects REACT_APP_RAPID_API_KEY at build time (frontend) or runtime (backend).
* If you’re unsure, test the pod’s environment:

`kubectl exec -it <pod-name> -n prod -- env | grep REACT_APP_RAPID_API_KEY`

If the variable is set, it’s runtime. If unset or the app fails to access the API, it’s likely build-time.

# Scaling: 
- With 2 replicas (per your `deployment.yaml`), your app uses at least 0.2 CPU and 256 MiB (2 × requests) and up to 1 CPU and 1024 MiB (2 × limits). Ensure your EKS nodes (provisioned by Terraform) have enough capacity.

# Resource Optimization
- Configured Kubernetes resource requests (100m CPU, 128Mi memory) and limits (500m CPU, 512Mi memory) for efficient pod scheduling.
- Monitored usage with Prometheus/Grafana to ensure cost-effective scaling in EKS.

# Verify the Service Account is assigned:
`kubectl get deployment aws-load-balancer-controller -n kube-system -o yaml`

- The `serviceAccountName` field assigns the Service Account (created by terraform) to the pods.
- Confirm the eks.amazonaws.com/role-arn annotation.

---

# ALB Controller

## How It Works:
1. The ALB Controller creates an ALB in public subnets (specified via subnet tags or annotations).
2. The ALB routes traffic to worker nodes in private subnets, targeting pods via the youtube-app.
3. Worker nodes remain in private subnets, ensuring no direct internet access.

### Check which Nodes ALB Controller are running on:
`kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller -o wide`

---

# Spring PetClinic Sample Application

# Features for Java App

1. Deployment for Java App:
Object: Deployment
Purpose: Manages stateless Java app pods (e.g., Spring Boot, Tomcat) with replicas for high availability.
Best Practices:
Set replicas: 2 for redundancy.
Use livenessProbe and readinessProbe (HTTP or TCP, depending on the app’s endpoint).
Define resource requests and limits for CPU/memory.
Inject sensitive data (e.g., database credentials) via AWS Parameter Store.

2. Service for Java App:
Object: Service
Purpose: Exposes the Java app internally (port 8080, typical for Java web apps) to MySQL and external traffic.
Best Practices:
Use ClusterIP for internal load balancing.
Map service port to container port (e.g., 8080).

3. Ingress for Java App:
Object: Ingress
Purpose: Routes external traffic (e.g., java-app.your-domain.com) to the Java app via the AWS ALB (using AWS Load Balancer Controller).
Best Practices:
Add ALB annotations (e.g., kubernetes.io/ingress.class: alb).
Use health check paths (e.g., /health).
Optionally enable HTTPS with AWS Certificate Manager.
AWS Parameter Store Integration:
Object: Secret (or external secrets operator, but we’ll use Secret for simplicity)
Purpose: Stores sensitive data (e.g., database credentials, API keys) securely.
Best Practices:
Use AWS Systems Manager (SSM) Parameter Store to store secrets.
Integrate with Kubernetes via Secret populated by a sidecar or manual sync (e.g., AWS CLI in Jenkins).
Example: Store MySQL username/password in /java-app/mysql-credentials.

---

# Features for MySQL

1. **StatefulSet for MySQL:**
  Object: StatefulSet
  Purpose: Manages MySQL pods, ensuring stable network identities and persistent storage for the database.
  Best Practices:
  * Use a single replica for simplicity (or a leader-follower setup for HA).
  * Define livenessProbe and readinessProbe (TCP port 3306).
  * Set resource requests and limits.

2. **Service for MySQL:**
Object: Service
Purpose: Exposes MySQL internally (port 3306) for the Java app to connect.
Best Practices:
Store MySQL credentials in SSM:
Plan to sync these to a Kubernetes Secret (via a tool).
Use ClusterIP with a headless service for StatefulSet (if needed for DNS).
Ensure the Java app connects via the service name (e.g., mysql.java-app.svc.cluster.local:3306).

3. PersistentVolumeClaim (PVC) for MySQL:
Object: PersistentVolumeClaim
Purpose: Provides persistent storage for MySQL data, backed by AWS EBS (via EKS storage class).
Best Practices:
Use storageClassName: gp2 (AWS EBS default).
Request sufficient storage (e.g., 10Gi).
Mount at /var/lib/mysql.

4. ConfigMap for MySQL Configuration:
Object: ConfigMap
Purpose: Stores MySQL configuration (e.g., my.cnf) for tuning (e.g., buffer pool size, max connections).
Best Practices:
Mount as a volume in the MySQL container.
Keep configurations minimal for learning.

Secret for MySQL Credentials:
Object: Secret
Purpose: Stores MySQL root password and user credentials (synced from AWS Parameter Store).
Best Practices:
Use environment variables or mounted files for credentials.
Avoid hardcoding in manifests.

CronJob for MySQL Backups:
Object: CronJob
Purpose: Schedules periodic backups of MySQL data to AWS S3.
Best Practices:
Use a backup tool like mysqldump or xtrabackup in a sidecar container.
Store backups in an S3 bucket (e.g., s3://java-app-backups/).
S3 Backups: Ensure the S3 bucket for MySQL backups is created, via Terraform.
Secure S3 access with IAM roles for the pod.
Schedule daily backups (e.g., 0 2 * * * for 2 AM UTC).
Additional Best Practices (Kubernetes Objects)

NetworkPolicy:
Object: NetworkPolicy
Purpose: Restricts traffic to/from the Java app and MySQL pods.
Best Practices:
Allow Java app to MySQL (port 3306).
Allow Ingress traffic to Java app (port 8080).
Deny other traffic for security.

HorizontalPodAutoscaler (HPA):
Object: HorizontalPodAutoscaler
Purpose: Scales Java app pods based on CPU/memory usage.
Best Practices:
Set minReplicas: 2, maxReplicas: 5.
Target CPU utilization (e.g., 70%).

PodDisruptionBudget (PDB):
Object: PodDisruptionBudget
Purpose: Ensures a minimum number of Java app or MySQL pods are available during voluntary disruptions (e.g., node upgrades).
Best Practices:
Set minAvailable: 1 for MySQL.
Set minAvailable: 2 for Java app.

ServiceAccount and RBAC:
Objects: ServiceAccount, Role, RoleBinding
Purpose: Grants permissions for pods to access AWS Parameter Store or S3 (via IAM roles for service accounts, IRSA).
Best Practices:
Create a ServiceAccount for the Java app and MySQL.
Associate an IAM role with permissions for SSM and S3.

---

# Features Summary
Here’s a consolidated list of features and Kubernetes objects for the Java app and MySQL deployment:

Java App:
Deployment: Stateless app with replicas, health checks, resource limits.
Service: Internal load balancing (port 8080).
Ingress: External access via ALB.
Secret: AWS Parameter Store for sensitive data (e.g., DB credentials).
HorizontalPodAutoscaler: Auto-scaling based on load.
NetworkPolicy: Restrict traffic to MySQL and Ingress.

MySQL:
StatefulSet: Stable storage and network identity for the database.
Service: Internal access for Java app (port 3306).
PersistentVolumeClaim: EBS-backed storage for data.
ConfigMap: MySQL configuration.
Secret: Database credentials from Parameter Store.
CronJob: Scheduled backups to S3.
NetworkPolicy: Restrict traffic to Java app.
Shared:
ServiceAccount and RBAC: IAM roles for AWS SSM and S3 access.
PodDisruptionBudget: Ensure availability during disruptions.
CI/CD Integration:
Jenkins pipeline to build/push Java app Docker image and update manifests.
ArgoCD to sync java-app/ manifests to the java-app namespace.
Grafana/Prometheus to monitor Java app and MySQL metrics (e.g., CPU, memory, DB connections).