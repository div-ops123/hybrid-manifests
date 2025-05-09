# YouTube Clone Application and Spring PetClinic Sample Application Kubernetes Manifest


### 1. Confirming Repository Structure

Single repository with manifests organized as:
- `youtube-clone-manifest/`: Manifests for the **2-tier YouTube Clone app.**
- `devops-learning-manifest/`: Manifests for the **3-tier app** (React frontend, Flask backend, RDS PostgreSQL).
- All resources in the `prod` namespace to align with your Terraform RBAC setup (`prod-editors` group for CI/CD).

#### Updated Repository Structure
Here’s how your local and GitHub repository should look:

```bash
~/my-first-cluster-manifests/
├── youtube-clone-manifest/
│   ├── deployment.yaml           # YouTube Clone Deployment
│   ├── service.yaml              # YouTube Clone Service
│   ├── ingress.yaml              # YouTube Clone Ingress
├── devops-learning-manifest/
│   ├── frontend-deployment.yaml  # React Frontend Deployment
│   ├── frontend-service.yaml     # React Frontend Service
│   ├── frontend-ingress.yaml     # React Frontend Ingress
│   ├── backend-deployment.yaml   # Flask Backend Deployment
│   ├── backend-service.yaml      # Flask Backend Service
│   ├── backend-ingress.yaml      # Flask Backend Ingress
│   ├── backend-job.yaml          # Database Migration Job
│   ├── external-service.yaml     # External Service for RDS PostgreSQL
│   ├── secret-provider-class.yaml # Sync Parameter Store to Kubernetes Secret
│   ├── configmap.yaml            # Configuration for Flask Backend
```

---

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

# 3-Tier DevOps Learning Platform App

github code: https://github.com/akhileshmishrabiz/kubernetes-zero-to-hero/blob/main/3-tier-app-eks/frontend/nginx.conf

# Visual Flow of "DevOps Learning Platform App" Traffic:

```text
frontend.devops-learning-platform.com
  ↓ (DNS resolves to ALB via Route 53)
ALB (in public subnets, port 80)
  ↓ (host: frontend.devops-learning-platform.com, path-based routing)
  ├── Path: /api
  │     ↓ (targeting backend ClusterIP Service, port 8000)
  │   Worker Nodes’ ENIs (in private subnets)
  │     ↓ (kube-proxy routes to ClusterIP Service)
  │   ClusterIP Service (devops-learning-backend-service, virtual IP, port 8000)
  │     ↓ (load-balances to backend pods, targetPort 8000)
  │   Backend Pods (on worker nodes, container port 8000, Flask)
  │     ↓ (queries RDS via External Service)
  │   External Service (devops-learning-postgres, ExternalName to RDS endpoint)
  │     ↓ (VPC private network)
  │   RDS PostgreSQL (in private subnets, port 5432)
  │     ↑ (returns query results)
  │   Backend Pods
  │     ↑ (returns API response)
  │   ClusterIP Service -> Worker Nodes -> ALB
  ↓
  └── Path: /
        │ (targeting frontend ClusterIP Service, port 80)
        ↓
      Worker Nodes’ ENIs (in private subnets)
        │
        ↓ (kube-proxy routes to ClusterIP Service)
      ClusterIP Service (devops-learning-frontend-service, virtual IP, port 80)
        │ (load-balances to frontend pods, targetPort 80)
        ↓ 
      Frontend Pods (on worker nodes, container port 80, Nginx)
        ↑ 
        │ (serves React app)
      ClusterIP Service -> Worker Nodes -> ALB
  ↑ (returns HTML/JS or API data to browser)
User’s Browser
```

### Features and Kubernetes Objects to Include

3-tier application consists of:
- **React Frontend**: Served via Nginx, using the image `livingdevopswithakhilesh/devopsdozo:frontend-latest`.
- **Flask Backend**: REST API, using the image `livingdevopswithakhilesh/devopsdozo:backend-latest`.
- **RDS PostgreSQL**: Managed database in the same VPC as the EKS cluster, deployed in private subnets.
- **Production-Like**: Uses IRSA, ASCP, and Parameter Store for secure credential management.

# Confirming Pods in `prod` Namespace Can Fetch Credentials
To ensure pods in the `prod` namespace (e.g., `devops-learning-backend`, `devops-learning-backend-migration`) can fetch credentials via ASCP, the following must be true:

- **IRSA:** Pods use `secrets-provider-aws-sa`, which assumes the `devops_learning_irsa` role with Parameter Store access.
- **ASCP:** The CSI driver and ASCP are installed, and the SecretProviderClass syncs Parameter Store to `devops-learning-secrets`.
- **Manifests:** Deployment and Job correctly reference the Secret and ServiceAccount.

---
Register the Domain:
Create a Hosted Zone:
Update Name Servers:

# Verification & Testing:
```bash
# Verify SecretProviderClass:
kubectl get secretproviderclass devops-learning-secrets -n prod

# Check devops-learning-secrets is created:
kubectl get secret devops-learning-secrets -n prod -o yaml
# If the Secret isn’t created: 
# 1. check the CSI Driver and ASCP pods:
kubectl get pods -n kube-system -l app=secrets-store-csi-driver
kubectl get pods -n kube-system -l app=secrets-store-csi-driver-provider-aws
# 2. View logs for errors
kubectl logs -n kube-system -l app=secrets-store-csi-driver-provider-aws

# Check secret contents (for debugging, decode Base64):
kubectl get secret devops-learning-secrets -n prod -o jsonpath='{.data.DB_USERNAME}' | base64 -d

# Monitor ASCP Logs: If secrets don’t sync, check ASCP logs:
kubectl logs -n kube-system -l app=secrets-provider-aws

# Check CSI driver pods:
kubectl get pods -n kube-system -l app.kubernetes.io/name=secrets-store-csi-driver

# Check ASCP pods:
kubectl get pods -n kube-system -l app=secrets-provider-aws

# Test EKS Add-Ons:
kubectl get sa -n kube-system -o yaml

kubectl get ingressclass
kubectl get ingress devops-learning-frontend-ingress -n prod
# Look for the Address field in kubectl describe output
kubectl describe ingress devops-learning-frontend-ingress -n prod


# Check secrets-provider-aws-sa:
# kubectl get sa secrets-provider-aws-sa -n prod -o yaml

# Verify ConfigMap:
kubectl get configmap devops-learning-config -n prod

# Verify the migration Job completed:
kubectl get job devops-learning-backend-migration -n prod
kubectl logs -n prod <job-pod>

# Backend Verify Deployment:
kubectl get deployment devops-learning-backend -n prod
kubectl describe deployment devops-learning-backend -n prod
kubectl get pods -n prod -l app=devops-learning-backend
kubectl get svc devops-learning-backend -n prod
kubectl get svc devops-learning-postgres -n prod
kubectl describe svc devops-learning-postgres -n prod
kubectl get pods -n prod -l app=devops-learning-frontend

# Confirm nginx.conf proxy to backend service
kubectl exec -n prod <frontend-pod> -- cat /etc/nginx/conf.d/default.conf

# Confirm the pod’s environment variables:
kubectl exec -n prod <backend-pod> -- printenv | grep -E 'DATABASE_URL|SECRET_KEY'

# Test Database Connectivity: Exec into a backend pod
kubectl exec -n prod -it <backend-pod> -- psql -h $RDS_ENDPOINT -U $DB_USERNAME -d $DB_NAME -c "SELECT 1;"
kubectl exec -n prod -it <backend-pod> -- psql -h $RDS_ENDPOINT -U $DB_USERNAME -d $DB_NAME -c "SELECT * FROM topics LIMIT 5;"

# Test API Health Endpoint:
kubectl exec -n prod <backend-pod> -- curl http://localhost:8000/api

# Logs:
kubectl logs -n prod <backend-pod>
kubectl logs -n prod <frontend-pod>

# Combined manifest for both apps:
cat ~/my-first-cluster-manifests/*/*.yaml > combined-manifest.yaml

# * Combined manifest for 1 app:
cat ~/my-first-cluster-manifests/devops-learning-manifest/*.yaml > combined-manifest.yaml

# * Ensure `Job` success:
kubectl logs -n prod -l job-name=devops-learning-backend-migration

# http://<ALB-DNS-NAME>/ OR
# Port forwarding Frontend:
kubectl port-forward svc/devops-learning-frontend-service 3000:80 -n prod
# - Access http://localhost:3000/

# Port forwarding Backend:
kubectl port-forward svc/devops-learning-backend-service 8000:8000 -n prod
# - Access: http://localhost:8000/api/topics
```

---

#### App Requirements (Based on [App code](https://github.com/akhileshmishrabiz/kubernetes-zero-to-hero/tree/main/3-tier-app-eks))

- **Environment Variables**:
rely on the `Secret` in your manifests (`devops-learning-secrets`) to provide `DATABASE_URL` and `SECRET_KEY`. No changes to `app/config.py` are needed.
  - `FLASK_APP`: Set to `run.py` for Flask CLI (e.g., migrations).
  - `FLASK_DEBUG`: Set to `0` for production (or `1` for debugging).
  - `DATABASE_URL`: PostgreSQL connection string (e.g., `postgresql://username:password@host:5432/dbname`).
  - `SECRET_KEY`: For session management and security.
- **Database**:
  - PostgreSQL with migrations (`flask db migrate`, `flask db upgrade`).
  - Initial data seeding (`seed_data.py`, `bulk_upload_questions.py`).
- **Networking**:
  - Runs on port `8000` (exposed in Dockerfile, served by Gunicorn).
  - Accessible via ALB (`frontend.devops-learning-platform.com/api` proxies to `http://backend.<namespace>.svc.cluster.local:8000`).
- **Secrets**:
  - Fetched from Parameter Store via ASCP and `SecretProviderClass`.
- **Health Check**:
  - The [health check endpoint for your Flask-based backend is `GET /api`](https://github.com/akhileshmishrabiz/kubernetes-zero-to-hero/blob/main/3-tier-app-eks/backend/app/routes/__init__.py)

---
# Files to edit:
1. backend/app/config.py
2. frontend/nginx.conf

- **Networking**:
  - Exposes port `8000`, matching the Dockerfile and `README.md`.
  - The frontend’s `nginx.conf` proxies `/api` requests to `http://backend.3-tier-app-eks.svc.cluster.local:8000`, but your service is in the `prod` namespace (`devops-learning-backend.prod.svc.cluster.local`). This needs alignment (see issues).

**Issues and Fixes**:
1. **Namespace Mismatch**:
   - **Issue**: Your manifests use the `prod` namespace, but the frontend’s `nginx.conf` references `backend.3-tier-app-eks.svc.cluster.local`, and the teacher’s `Secret` uses `3-tier-app-eks`. This suggests a namespace mismatch.
   - **Fix**: Update the frontend’s `nginx.conf` to proxy to the correct service:
     ```nginx
     location /api {
       proxy_pass http://devops-learning-backend.prod.svc.cluster.local:8000/;
     }
     ```
    - Fix: Fork his 3-tier code to create your own image becos of this.
   - **Action**: Confirm the backend service name with:
```bash
kubectl get svc -n prod
```
     If the service is named differently (e.g., `backend`), update `nginx.conf` accordingly.
     
---

### Features for React Frontend
1. **Deployment for React Frontend**:
   - **Object**: `Deployment`
   - **Purpose**: Manages stateless React frontend pods, serving the UI via Nginx.

2. **Service for React Frontend**:
   - **Object**: `Service`
   - **Purpose**: Exposes the frontend internally on port 80 for the ALB Ingress.
   - **The frontend** needs a stable API URL that resolves to the backend, either via Kubernetes DNS (internal) or ALB proxying (external).

3. **Ingress for React Frontend**:
   - **Object**: `Ingress`
   - **Purpose**: Routes external traffic to the frontend service via AWS ALB.

---

#### Features for RDS PostgreSQL
10. **External Service for RDS PostgreSQL**:
    - **Object**: `Service` (without selectors, using `ExternalName` or endpoints)
    - **Purpose**: Enables the Flask backend to connect to the RDS instance using Kubernetes DNS (e.g., `devops-learning-postgres.prod.svc.cluster.local:5432`).
      - Set port 5432 (PostgreSQL default).

#### Shared Features
11. **NetworkPolicy**:
    - **Object**: `NetworkPolicy`
    - **Purpose**: Restricts traffic for security.
    - **Best Practices**:
      - Allow frontend to backend (port 8000).
      - Allow backend to RDS PostgreSQL (port 5432 via `External Service`).
      - Allow ALB Ingress to frontend and backend.
      - Deny other traffic.

12. **HorizontalPodAutoscaler (HPA)**:
    - **Object**: `HorizontalPodAutoscaler`
    - **Purpose**: Scales frontend and backend pods based on CPU/memory usage.
    - **Best Practices**:
      - Set `minReplicas: 2`, `maxReplicas: 5`.
      - Target CPU utilization (e.g., 70%).

13. **PodDisruptionBudget (PDB)**:
    - **Object**: `PodDisruptionBudget`
    - **Purpose**: Ensures minimum availability during voluntary disruptions (e.g., node upgrades).
    - **Best Practices**:
      - Set `minAvailable: 1` for both frontend and backend.

14. **ServiceAccount and RBAC**:
    - **Object**: `ServiceAccount`, `Role`, `RoleBinding`
    - **Purpose**: Grants pods access to AWS Parameter Store and S3 (for backups, if needed) via IAM roles (IRSA).
    - **Best Practices**:
      - Create a `ServiceAccount` for the backend.
      - Associate an IAM role with permissions for SSM Parameter Store.
      - Use OIDC for secure authentication.

15. **CI/CD Integration**:
    - **Purpose**: Automates deployment and updates.
    - **Best Practices**:
      - **Jenkins**: Update the pipeline to handle frontend and backend images (if you rebuild them) and update manifests in `devops-learning-manifest/`.
      - **ArgoCD**: Configure an `Application` to sync `devops-learning-manifest/` to the `prod` namespace.
      - Store manifests in `my-first-cluster-manifests` for GitOps.

16. **Monitoring**:
    - **Purpose**: Tracks app performance and health.
    - **Best Practices**:
      - Use Prometheus to scrape metrics:
        - Frontend: Nginx metrics (e.g., request latency).
        - Backend: Flask metrics (e.g., `/metrics` if exposed) or pod metrics.
        - RDS: CloudWatch metrics via Prometheus exporter.
      - Create Grafana dashboards for CPU, memory, request rates, and DB connections.
      - Set alerts for pod crashes or high error rates.

17. **Route 53 Integration**:
    - **Purpose**: Maps domains to the ALB.
    - **Best Practices**:
      - Create DNS records in Route 53:
        - `frontend.devops-learning-platform.com` -> ALB hostname (frontend).
        - `api.your-domain.com` -> ALB hostname (backend, if exposed).
      - Use AWS ACM for HTTPS certificates.

---

4. **Route 53**:
   - Create DNS records after deploying the ALB:
     ```bash
     aws route53 change-resource-record-sets \
       --hosted-zone-id <your-zone-id> \
       --change-batch '{"Changes":[{"Action":"CREATE","ResourceRecordSet":{"Name":"frontend.devops-learning-platform.com","Type":"A","AliasTarget":{"HostedZoneId":"<ALB-ZONE-ID>","DNSName":"<ALB-HOSTNAME>","EvaluateTargetHealth":true}}}]}'
     ```

---
