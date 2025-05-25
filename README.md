# EKS

Perfect! Let's walk through **every single step**, including how to create IAM **Users**, **Groups**, **Policies**, and map them to your **EKS cluster** using `aws-auth` and Kubernetes **RBAC**.

---

# ✅ Complete Guide: Add IAM Users/Groups to EKS Cluster with Different Permissions

---

## 🧱 SCENARIO

You have an EKS cluster and want to:

1. Create IAM Users in AWS
2. Add them to IAM Groups (e.g., `eks-dev-read`, `eks-admin`)
3. Map IAM users/groups to EKS via `aws-auth` ConfigMap
4. Assign permissions using Kubernetes RBAC

---

## 🧰 Tools Required

* AWS CLI
* `kubectl`
* Access to AWS Console (IAM and EKS)
* EKS cluster already created
* `eksctl` (optional but helpful)

---

## ✅ STEP 1: Create IAM Users & Groups in AWS

### 🔹 Step 1.1: Create IAM Group (e.g., `eks-dev-read`)

#### 📍 Via AWS Console:

* Go to **IAM > Groups > Create Group**
* Name it: `eks-dev-read`
* Skip attaching policies (not needed for EKS Auth)
* Create the group

#### 📍 Or via AWS CLI:

```bash
aws iam create-group --group-name eks-dev-read
```

---

### 🔹 Step 1.2: Create IAM User (e.g., `dev-user`) and Add to Group

#### 📍 Via AWS Console:

* Go to **IAM > Users > Add user**
* Name: `dev-user`
* Select: **Programmatic access** (optional for CLI) or **Console access**
* Skip policy attachment
* Add to group: `eks-dev-read`
* Finish creation

#### 📍 Or via AWS CLI:

```bash
aws iam create-user --user-name dev-user
aws iam add-user-to-group --user-name dev-user --group-name eks-dev-read
```

---

## ✅ STEP 2: Get IAM User or Group ARN

### For user:

```bash
aws iam get-user --user-name dev-user
```

Output:

```json
"Arn": "arn:aws:iam::123456789012:user/dev-user"
```

### For group:

```bash
aws iam get-group --group-name eks-dev-read
```

Output:

```json
"Arn": "arn:aws:iam::123456789012:group/eks-dev-read"
```

> But EKS only supports mapping **users and roles**, not groups directly.

---

## ✅ STEP 3: Add IAM User to EKS Cluster using `aws-auth`

### 🔹 Step 3.1: Edit aws-auth ConfigMap

```bash
kubectl edit configmap aws-auth -n kube-system
```

### 🔹 Step 3.2: Add the IAM user mapping

Example:

```yaml
mapUsers:
  - userarn: arn:aws:iam::123456789012:user/dev-user
    username: dev-user
    groups:
      - eks-readonly
```

> `eks-readonly` is a **K8s RBAC group** — we will define its permissions next.

✅ Save and exit the editor.

---

## ✅ STEP 4: Create RBAC Role and Bind it to Group

### 🔹 Step 4.1: Create Role for Read-Only Access

```yaml
# eks-read-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: eks-read-role
  namespace: dev
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
```

Apply:

```bash
kubectl apply -f eks-read-role.yaml
```

---

### 🔹 Step 4.2: Create RoleBinding for Group

```yaml
# eks-read-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: eks-read-binding
  namespace: dev
subjects:
  - kind: Group
    name: eks-readonly
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: eks-read-role
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```bash
kubectl apply -f eks-read-binding.yaml
```

---

## ✅ STEP 5: Allow IAM User to Access Cluster

On IAM user system (or in your profile):

```bash
aws eks update-kubeconfig --name my-cluster --region us-east-1 --profile dev-user
```

Now test:

```bash
kubectl get pods -n dev              ✅ Allowed
kubectl get pods -n kube-system      ❌ Forbidden
kubectl create deployment nginx ...  ❌ Forbidden
```

---

## ✅ OPTIONAL: Admin Access for Another User

If you want to grant **admin** access to another IAM user (e.g., `platform-admin`):

### 1. Get ARN:

```bash
arn:aws:iam::123456789012:user/platform-admin
```

### 2. Add to aws-auth:

```yaml
mapUsers:
  - userarn: arn:aws:iam::123456789012:user/platform-admin
    username: platform-admin
    groups:
      - system:masters
```

> `system:masters` is the built-in **admin group** in Kubernetes.

No need to define RBAC roles separately.

---

## ✅ Recap Summary Table

| IAM User         | IAM Group      | K8s Group        | Access Level       | Namespace |
| ---------------- | -------------- | ---------------- | ------------------ | --------- |
| `dev-user`       | `eks-dev-read` | `eks-readonly`   | Read-Only          | `dev`     |
| `qa-deployer`    | `eks-qa-team`  | `eks-qa-deploy`  | Create/Update Pods | `qa`      |
| `platform-admin` | N/A            | `system:masters` | Full Admin         | All       |

---

## ✅ Best Practices

* Use **IAM Roles with STS AssumeRole** for automation (better than static users)
* Create RBAC groups like `eks-dev`, `eks-admin`, etc.
* Keep `aws-auth` entries consistent across environments using GitOps or Terraform
* Audit IAM and RBAC regularly

---

Would you like me to give you **Terraform** or **CloudFormation** setup for automating this process?
