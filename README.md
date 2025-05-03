# YouTube Clone Application and Spring PetClinic Sample Application Kubernetes Manifest


### 1. Confirming Repository Structure

Single repository with manifests organized as:
- `youtube-clone-manifest/`: Manifests for the YouTube Clone app.
- `devops-learning-manifest/`: Manifests for the 3-tier app (React frontend, Flask backend, RDS PostgreSQL).
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
│   ├── configmap.yaml            # Configuration for Flask Backend
│   ├── secret.yaml               # Secrets for RDS credentials (from AWS Parameter Store)
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

#### Naming Convention
- Follow your established best practice for `metadata.name` to ensure clarity and uniqueness:
  - **Frontend**:
    - `Deployment`: `devops-learning-frontend`
    - `Service`: `devops-learning-frontend-service`
    - `Ingress`: `devops-learning-frontend-ingress`
  - **Backend**:
    - `Deployment`: `devops-learning-backend`
    - `Service`: `devops-learning-backend-service`
    - `Ingress`: `devops-learning-backend-ingress`
    - `Job`: `devops-learning-backend-migration`
  - **RDS PostgreSQL**:
    - `External Service`: `devops-learning-postgres`
    - `ConfigMap`: `devops-learning-config`
    - `Secret`: `devops-learning-secrets`
- Use consistent `labels` for selectors (e.g., `app: devops-learning-frontend`, `app: devops-learning-backend`).

---

### Features and Kubernetes Objects to Include

3-tier application consists of:
- **React Frontend**: Served via Nginx, using the image `livingdevopswithakhilesh/devopsdozo:frontend-latest`.
- **Flask Backend**: REST API, using the image `livingdevopswithakhilesh/devopsdozo:backend-latest`.
- **RDS PostgreSQL**: Managed database in the same VPC as the EKS cluster, deployed in private subnets.


### Features for React Frontend
1. **Deployment for React Frontend**:
   - **Object**: `Deployment`
   - **Purpose**: Manages stateless React frontend pods, serving the UI via Nginx.
   - **Best Practices**:
     - Set `replicas: 2` for high availability.
     - Use `livenessProbe` and `readinessProbe` (HTTP GET on `/`, port 80).
     - Define resource `requests` and `limits` (e.g., CPU: 100m/500m, Memory: 128Mi/512Mi).
     - Use image: `livingdevopswithakhilesh/devopsdozo:frontend-latest`.
     - Set environment variable `REACT_APP_API_URL` to point to the Flask backend service (e.g., `http://devops-learning-backend-service.prod.svc.cluster.local:8000/api`).

2. **Service for React Frontend**:
   - **Object**: `Service`
   - **Purpose**: Exposes the frontend internally on port 80 for the ALB Ingress.
   - **Best Practices**:
     - Use `ClusterIP` type.
     - Map service port 80 to container port 80 (Nginx).
     - Use selector `app: devops-learning-frontend`.

3. **Ingress for React Frontend**:
   - **Object**: `Ingress`
   - **Purpose**: Routes external traffic (e.g., `frontend.your-domain.com`) to the frontend service via AWS ALB.
   - **Best Practices**:
     - Use ALB annotations (e.g., `kubernetes.io/ingress.class: alb`, `alb.ingress.kubernetes.io/scheme: internet-facing`).
     - Set health check path to `/`.
     - Integrate with Route 53 by setting `host: frontend.your-domain.com`.
     - Optionally enable HTTPS with AWS Certificate Manager (ACM).

React frontend for DevOps Learning Platform     

#### Features for Flask Backend
4. **Deployment for Flask Backend**:
   - **Object**: `Deployment`
   - **Purpose**: Manages stateless Flask API pods, serving endpoints like `/api/topics` and `/api/quiz`.
   - **Best Practices**:
     - Set `replicas: 2` for redundancy.
     - Use `livenessProbe` and `readinessProbe` (HTTP GET on `/health` or `/api/topics`, port 8000).
     - Define resource `requests` and `limits` (e.g., CPU: 200m/1000m, Memory: 256Mi/1024Mi).
     - Use image: `livingdevopswithakhilesh/devopsdozo:backend-latest`.
     - Inject RDS credentials via `Secret` (from AWS Parameter Store).

5. **Service for Flask Backend**:
   - **Object**: `Service`
   - **Purpose**: Exposes the backend internally on port 8000 for the frontend and direct API access (if needed).
   - **Best Practices**:
     - Use `ClusterIP` type.
     - Map service port 8000 to container port 8000 (Flask).
     - Use selector `app: devops-learning-backend`.

6. **Ingress for Flask Backend** (Optional):
   - **Object**: `Ingress`
   - **Purpose**: Routes external traffic (e.g., `api.your-domain.com`) to the backend service for direct API access (e.g., for testing or admin tools).
   - **Best Practices**:
     - Use ALB annotations.
     - Set health check path to `/health` or `/api/topics`.
     - Set `host: api.your-domain.com` and integrate with Route 53.
     - Consider skipping this if the frontend handles all API traffic internally.

7. **Job for Database Migrations**:
   - **Object**: `Job`
   - **Purpose**: Runs `flask db migrate` and `flask db upgrade` to initialize the RDS PostgreSQL schema before the backend starts.
   - **Best Practices**:
     - Use the same backend image (`livingdevopswithakhilesh/devopsdozo:backend-latest`).
     - Set `restartPolicy: OnFailure` to retry failed migrations.
     - Inject RDS credentials via `Secret`.
     - Run as a one-time job before the backend `Deployment` (use `initContainer` or manual execution if preferred).

8. **ConfigMap for Backend Configuration**:
   - **Object**: `ConfigMap`
   - **Purpose**: Stores Flask configuration (e.g., `FLASK_DEBUG`, `SECRET_KEY`) and avoids hardcoding in the image.
   - **Best Practices**:
     - Mount as environment variables or a config file.
     - Example:
       ```yaml
       data:
         FLASK_DEBUG: "0"
         SECRET_KEY: "your-secret-key"
       ```

9. **Secret for RDS Credentials**:
   - **Object**: `Secret`
   - **Purpose**: Stores PostgreSQL credentials (username, password, database name) from AWS Parameter Store.
   - **Best Practices**:
     - Sync credentials from Parameter Store (e.g., `/devops-learning/db-username`, `/devops-learning/db-password`).
     - Inject as environment variables (e.g., `DATABASE_URL=postgresql://user:pass@host:5432/db`).
     - Use OIDC and IAM roles (IRSA) for secure access to Parameter Store.

#### Features for RDS PostgreSQL
10. **External Service for RDS PostgreSQL**:
    - **Object**: `Service` (without selectors, using `ExternalName` or endpoints)
    - **Purpose**: Enables the Flask backend to connect to the RDS instance using Kubernetes DNS (e.g., `devops-learning-postgres.prod.svc.cluster.local:5432`).
    - **Best Practices**:
      - Use an `ExternalName` service pointing to the RDS endpoint (e.g., `my-rds-instance.abcdef123456.us-east-1.rds.amazonaws.com`).
      - Alternatively, use a `Service` with manual endpoints if the RDS DNS name changes.
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
        - `frontend.your-domain.com` -> ALB hostname (frontend).
        - `api.your-domain.com` -> ALB hostname (backend, if exposed).
      - Use AWS ACM for HTTPS certificates.

---

### Additional Guidance for Writing Manifests

When writing your manifests in `~/my-first-cluster-manifests/devops-learning-manifest/`, keep these best practices in mind:
- **Namespace**: Set `metadata.namespace: prod` for all resources.
- **Naming**:
  - Use `metadata.name: devops-learning-frontend`, `devops-learning-backend`, etc.
  - Add suffixes (e.g., `-service`, `-ingress`) for clarity.
- **Labels**:
  - Use `labels: { app: devops-learning-frontend }` for frontend, `app: devops-learning-backend` for backend.
  - Ensure selectors match (e.g., `selector.matchLabels` in `Deployment` and `Service`).
- **Probes**:
  - Frontend: HTTP GET on `/` (port 80).
  - Backend: HTTP GET on `/health` or `/api/topics` (port 8000).
- **Resources**: Define `requests` and `limits` to prevent resource contention.
- **Secrets**:
  - Sync RDS credentials from Parameter Store (e.g., `/devops-learning/db-username`).
  - Use `Secret` for `DATABASE_URL` and `SECRET_KEY`.
- **Comments**: Add comments in manifests (e.g., `# React frontend for DevOps Learning Platform`).
- **Validation**: Test manifests locally:
  ```bash
  kubectl apply -f ~/my-first-cluster-manifests/devops-learning-manifest/ --dry-run=client
  ```

---

### AWS Setup Notes

To support the RDS PostgreSQL and Parameter Store, set up these AWS resources:

1. **RDS PostgreSQL Instance**:
   - Deploy in the same VPC as your EKS cluster, using **private subnets** (from your Terraform setup).
   - Example AWS CLI command:
     ```bash
     aws rds create-db-instance \
       --db-instance-identifier devops-learning-postgres \
       --db-instance-class db.t3.micro \
       --engine postgres \
       --allocated-storage 20 \
       --master-username postgres \
       --master-user-password <password> \
       --vpc-security-group-ids <your-eks-sg> \
       --db-subnet-group-name <your-private-subnet-group> \
       --no-publicly-accessible
     ```
   - Note the RDS endpoint (e.g., `devops-learning-postgres.abcdef123456.us-east-1.rds.amazonaws.com:5432`).

2. **Parameter Store for Credentials**:
   - Store RDS credentials:
     ```bash
     aws ssm put-parameter --name /devops-learning/db-username --value postgres --type SecureString
     aws ssm put-parameter --name /devops-learning/db-password --value <password> --type SecureString
     aws ssm put-parameter --name /devops-learning/db-name --value devops_learning --type SecureString
     ```

3. **IAM Role for Parameter Store**:
   - Update the `ServiceAccount`’s IAM role (IRSA) to include SSM permissions:
     ```json
     {
       "Effect": "Allow",
       "Action": [
         "ssm:GetParameter",
         "ssm:GetParameters"
       ],
       "Resource": "arn:aws:ssm:us-east-1:<account-id>:parameter/devops-learning/*"
     }
     ```

4. **Route 53**:
   - Create DNS records after deploying the ALB:
     ```bash
     aws route53 change-resource-record-sets \
       --hosted-zone-id <your-zone-id> \
       --change-batch '{"Changes":[{"Action":"CREATE","ResourceRecordSet":{"Name":"frontend.your-domain.com","Type":"A","AliasTarget":{"HostedZoneId":"<ALB-ZONE-ID>","DNSName":"<ALB-HOSTNAME>","EvaluateTargetHealth":true}}}]}'
     ```

---

### Next Steps

1. **Set Up the Repository**:
   - Ensure `~/my-first-cluster-manifests/` has `youtube-clone-manifest/` and `devops-learning-manifest/`.
   - Push to GitHub:
     ```bash
     cd ~/my-first-cluster-manifests
     git add devops-learning-manifest/
     git commit -m "Add devops-learning-manifest directory"
     git push origin main
     ```

2. **Create AWS Resources**:
   - Deploy the RDS PostgreSQL instance in private subnets.
   - Store credentials in Parameter Store.
   - Update the IAM role for SSM access.

3. **Write Manifests**:
   - Create files in `~/my-first-cluster-manifests/devops-learning-manifest/`:
     ```bash
     touch ~/my-first-cluster-manifests/devops-learning-manifest/frontend-deployment.yaml
     touch ~/my-first-cluster-manifests/devops-learning-manifest/backend-deployment.yaml
     # Add more as needed
     ```
   - Use the YouTube Clone manifests as a reference, adapting for frontend (port 80) and backend (port 8000).

4. **Share Manifests for Review**:
   - When ready, paste the manifests or share a GitHub link.
   - I’ll cross-check for correctness, best practices, and integration with EKS/ALB/CI/CD.

5. **Continue CI/CD**:
   - Extend the Jenkins pipeline to update `devops-learning-manifest/` (if you rebuild images).
   - Configure ArgoCD to sync `devops-learning-manifest/`.

---

### Clarifications and Notes

- **Single Repo**: `my-first-cluster-manifests` with `youtube-clone-manifest/` and `devops-learning-manifest/` is ideal for your setup.
- **Prod Namespace**: Using `prod` aligns with your RBAC and CI/CD requirements.
- **Naming**: Your convention (e.g., `devops-learning-frontend`, `devops-learning-backend-service`) is perfect.
- **RDS**: The `External Service` simplifies connectivity without running PostgreSQL in the cluster.
- **Learning Focus**: Writing manifests manually will deepen your Kubernetes skills.

I’ll wait for you to write and share the manifests for the 3-tier app. Let me know if you have questions about specific objects (e.g., `Job`, `ExternalName Service`) or AWS setup. You’re doing fantastic, and I’m excited to review your work! 🚀