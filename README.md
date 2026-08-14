# WordPress on Kubernetes with AWS EBS & EFS

A Kubernetes-based WordPress deployment that demonstrates persistent storage using **Amazon EBS** for MySQL and **Amazon EFS** for WordPress shared content.

The project uses Kubernetes Deployments, Services, Secrets, PersistentVolumeClaims, PersistentVolumes, and StorageClasses, together with the AWS EBS and EFS CSI drivers.

## Architecture

```text
                         ┌──────────────────────┐
                         │      WordPress        │
                         │   Deployment (2 Pods) │
                         └──────────┬───────────┘
                                    │
                           Service: LoadBalancer
                                    │
                                    ▼
                           External Access

                 ┌─────────────────────────────────┐
                 │        WordPress Storage        │
                 │                                 │
                 │ PVC: wordpress-efs-pvc          │
                 │ PV:  wordpress-efs-pv           │
                 │ Access: ReadWriteMany (RWX)     │
                 │ Backend: Amazon EFS              │
                 └─────────────────────────────────┘

                                    │
                                    │ MySQL connection
                                    ▼
                 ┌─────────────────────────────────┐
                 │            MySQL                 │
                 │      Deployment (1 Pod)          │
                 └──────────────┬──────────────────┘
                                │
                         Service: mysql-svc
                                │
                 ┌──────────────▼──────────────────┐
                 │         MySQL Storage            │
                 │                                  │
                 │ PVC: mysql-pvc                   │
                 │ StorageClass: mysql-sc            │
                 │ Access: ReadWriteOnce (RWO)      │
                 │ Backend: Amazon EBS               │
                 └──────────────────────────────────┘
```

## Key Features

- WordPress deployed on Kubernetes.
- MySQL 5.6 deployed as a Kubernetes Deployment.
- **Amazon EBS CSI driver** used for MySQL persistent storage.
- **Amazon EFS CSI driver** used for WordPress shared storage.
- WordPress storage uses **ReadWriteMany (RWX)**, allowing multiple WordPress replicas to use the same filesystem.
- MySQL storage uses **ReadWriteOnce (RWO)**.
- Kubernetes Secret used to store the MySQL password.
- WordPress exposed through a Kubernetes `LoadBalancer` Service.
- WordPress can be scaled horizontally to multiple replicas.

## Technologies

| Technology | Purpose |
|---|---|
| Kubernetes | Container orchestration |
| Amazon EBS | Persistent storage for MySQL |
| Amazon EFS | Shared persistent storage for WordPress |
| EBS CSI Driver | Connects Kubernetes to EBS volumes |
| EFS CSI Driver | Connects Kubernetes to EFS |
| MySQL 5.6 | WordPress database |
| WordPress | Web application |

## Project Structure

```text
wordpress-project/
│
├── mysql-secret.yaml
├── mysql-storageclass.yaml
├── mysql-pvc.yaml
├── mysql-deployment.yaml
├── mysql-service.yaml
│
├── wordpress-storageclass.yaml
├── wordpress-pv.yaml
├── wordpress-pvc.yaml
├── wordpress-deployment.yaml
└── wordpress-service.yaml
```
## 🤖 How AI Helped

AI tools were used as a productivity assistant during the development and documentation of this project.

They helped me with:

- Structuring and writing the project README documentation.
- Improving the clarity and organization of Kubernetes deployment steps.
- Explaining Kubernetes and AWS concepts when needed during implementation.
- Troubleshooting configuration and deployment issues by analyzing errors and suggesting possible solutions.
- Reviewing YAML manifests for consistency and readability.
- Improving technical explanations and documentation quality.

The Kubernetes manifests, AWS configuration, deployment process, and overall implementation were tested and validated by me.

AI was used as a supporting tool for learning, troubleshooting, documentation, and improving productivity—not as a replacement for understanding or implementing the infrastructure.

## Prerequisites

Before deploying the project, the Kubernetes cluster must have the AWS storage CSI drivers available.

Verify them with:

```bash
kubectl get csidriver
```

Expected drivers:

```text
ebs.csi.aws.com
efs.csi.aws.com
```

The cluster nodes should also report both drivers:

```bash
kubectl get csinode
```

## AWS Preparation

### 1. EBS / EFS permissions

The project requires AWS permissions for the EBS and EFS CSI components. The source setup uses permissions associated with:

- `AmazonEBSCSIDriverPolicy`
- `AmazonElasticFileSystemFullAccess`

The EFS CSI setup also uses permissions for describing EFS resources, creating and tagging access points, and deleting tagged access points.

> **Security recommendation:** do not commit AWS access keys or secrets to GitHub. Use IAM roles, instance profiles, IRSA, or another secure credential mechanism appropriate for your cluster.

### 2. Create the EFS filesystem

For WordPress persistent shared storage, create an Amazon EFS filesystem and an EFS Access Point.

The source setup uses these Access Point settings:

```text
Root directory: /wordpress
POSIX User ID: 1000
POSIX Group ID: 1000
Root Owner User ID: 1000
Root Group ID: 1000
POSIX Permissions: 777
```

Update `wordpress-pv.yaml` with your own EFS filesystem and access point IDs.

Example:

```yaml
volumeHandle: fs-XXXXXXXX::fsap-XXXXXXXX
```

The EFS security group must allow the required NFS traffic between the Kubernetes nodes and the EFS mount targets.

## Kubernetes Resources

### MySQL

#### Secret

`mysql-secret.yaml` stores the MySQL root password as a Kubernetes Secret.

#### StorageClass

`mysql-storageclass.yaml` uses:

```yaml
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
```

This allows the EBS volume to be provisioned with awareness of the Pod's scheduling requirements.

#### PersistentVolumeClaim

`mysql-pvc.yaml` requests:

```text
5Gi
ReadWriteOnce
```

#### Deployment

`mysql-deployment.yaml` runs:

```text
mysql:5.6
```

The persistent volume is mounted at:

```text
/var/lib/mysql
```

#### Service

`mysql-service.yaml` exposes MySQL internally as:

```text
mysql-svc:3306
```

### WordPress

#### StorageClass

`wordpress-storageclass.yaml` uses the AWS EFS CSI driver:

```yaml
provisioner: efs.csi.aws.com
```

#### PersistentVolume

`wordpress-pv.yaml` statically references the EFS filesystem and access point.

The volume is configured with:

```text
Access Mode: ReadWriteMany
Reclaim Policy: Retain
Capacity: 5Gi
```

#### PersistentVolumeClaim

`wordpress-pvc.yaml` binds to the EFS-backed PV using `ReadWriteMany`.

#### Deployment

`wordpress-deployment.yaml` uses:

```text
wordpress:php7.1-apache
```

The WordPress content directory is mounted at:

```text
/var/www/html
```

The database host is configured as:

```text
mysql-svc
```

#### Service

`wordpress-service.yaml` exposes WordPress through a Kubernetes `LoadBalancer` Service on port `80`.

## Deployment

Apply the resources in the following order.

### 1. MySQL

```bash
kubectl apply -f mysql-secret.yaml
kubectl apply -f mysql-storageclass.yaml
kubectl apply -f mysql-pvc.yaml
kubectl apply -f mysql-deployment.yaml
kubectl apply -f mysql-service.yaml
```

Check the resources:

```bash
kubectl get sc
kubectl get pvc
kubectl get pv
kubectl get pods
kubectl get svc
```

The MySQL PVC initially may remain `Pending` because the StorageClass uses `WaitForFirstConsumer`. Once the MySQL Pod is scheduled, Kubernetes provisions the EBS volume and the PVC should become `Bound`.

### 2. WordPress

```bash
kubectl apply -f wordpress-storageclass.yaml
kubectl apply -f wordpress-pv.yaml
kubectl apply -f wordpress-pvc.yaml
kubectl apply -f wordpress-deployment.yaml
kubectl apply -f wordpress-service.yaml
```

Verify the deployment:

```bash
kubectl get pods
kubectl get deploy
kubectl get pvc
kubectl get pv
kubectl get svc
```

## Scaling WordPress

Because WordPress uses an EFS-backed `ReadWriteMany` volume, the deployment can be scaled to multiple replicas while sharing the same persistent filesystem.

Example:

```bash
kubectl scale deployment wordpress --replicas=2
```

Verify:

```bash
kubectl get deployment wordpress
kubectl get pods
```

## Accessing WordPress

The WordPress Service is configured as:

```yaml
type: LoadBalancer
```

Check the service:

```bash
kubectl get svc wordpress
```

Depending on the Kubernetes environment and cloud integration, an external endpoint may be assigned.

The source setup also demonstrates access through the node IP and assigned service port when applicable.

## Useful Commands

Check all resources:

```bash
kubectl get all
```

Check storage:

```bash
kubectl get sc
kubectl get pv
kubectl get pvc
```

Check CSI drivers:

```bash
kubectl get csidriver
kubectl get csinode
```

Check WordPress logs:

```bash
kubectl logs deployment/wordpress
```

Check MySQL logs:

```bash
kubectl logs deployment/mysql-app
```

Describe a resource when troubleshooting:

```bash
kubectl describe pod <pod-name>
kubectl describe pvc <pvc-name>
kubectl describe pv <pv-name>
```

## Storage Design

One of the main goals of this project is to demonstrate the difference between block storage and shared file storage in Kubernetes.

### MySQL → Amazon EBS

MySQL uses EBS because the database requires persistent block storage and the workload is attached to a single Pod through a `ReadWriteOnce` claim.

```text
MySQL Pod
   │
   ▼
mysql-pvc
   │
   ▼
mysql-sc
   │
   ▼
Amazon EBS
```

### WordPress → Amazon EFS

WordPress uses EFS because multiple WordPress replicas need to access the same application files.

```text
WordPress Pod 1 ─┐
                 │
WordPress Pod 2 ─┼──> wordpress-efs-pvc
                 │
                 └──> Amazon EFS
```

## Project Validation

The original setup successfully demonstrated:

- MySQL Pod reaching `Running` state.
- MySQL PVC becoming `Bound`.
- EBS-backed PV being dynamically provisioned.
- WordPress EFS PV becoming `Bound`.
- WordPress Pod reaching `Running` state.
- WordPress deployment scaling from 1 replica to 2 replicas.
- WordPress and MySQL Services being created successfully.

## Troubleshooting

### MySQL PVC is Pending

This is expected initially because the MySQL StorageClass uses:

```yaml
volumeBindingMode: WaitForFirstConsumer
```

Check:

```bash
kubectl get pvc
kubectl get pods
kubectl describe pvc mysql-pvc
```

### WordPress PVC is Pending

Check that:

- The EFS PV exists.
- The EFS Access Point is valid.
- `storageClassName` matches `efs-sc`.
- The EFS CSI driver is installed.
- Network/security group rules allow NFS connectivity.

Commands:

```bash
kubectl get pv
kubectl get pvc
kubectl get csidriver
kubectl describe pvc wordpress-efs-pvc
```

### WordPress Service has no External IP

Check:

```bash
kubectl get svc wordpress
kubectl describe svc wordpress
```

The assignment of an external address depends on the Kubernetes environment and its cloud/load-balancer integration.

## Lessons Learned

This project demonstrates several practical Kubernetes and AWS concepts:

1. Persistent storage is different depending on the workload requirements.
2. EBS is suitable for block storage workloads such as MySQL.
3. EFS provides shared storage suitable for multiple WordPress replicas.
4. CSI drivers connect Kubernetes storage abstractions with AWS storage services.
5. `ReadWriteOnce` and `ReadWriteMany` solve different application requirements.
6. Kubernetes StorageClasses can dynamically provision storage instead of manually creating every volume.
7. Persistent storage is essential when scaling stateful or semi-stateful workloads.

## Security Notes

The project uses Kubernetes Secrets for the MySQL password, but the example repository should not contain real credentials.

For a production deployment, consider:

- IAM roles instead of long-lived AWS access keys.
- Restricting EFS security-group rules to the required sources.
- Avoiding `777` permissions for production EFS access points.
- Using a currently supported WordPress image and MySQL version.
- Applying Kubernetes NetworkPolicies and stronger workload security controls.

## Important Notes

This README reflects the configuration and workflow in the provided project source. Before deploying in another environment, replace environment-specific AWS identifiers such as the EFS filesystem and access point IDs.

## Author

**Mahmoud Saeed**

AWS / Cloud / DevOps Learner
