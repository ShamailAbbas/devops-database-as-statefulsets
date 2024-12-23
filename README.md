# Deploying a Database StatefulSet with AWS EBS Storage on Kubernetes

This guide walks through deploying a database (e.g., MySQL) in a Kubernetes StatefulSet, using AWS EBS volumes for persistent storage. AWS credentials are securely managed via Kubernetes Secrets.

---

## Prerequisites

1. Kubernetes cluster (self-managed or non-EKS).
2. AWS account with access to IAM and EC2 services.
3. Installed tools:
   - [kubectl](https://kubernetes.io/docs/tasks/tools/)
   - [Helm](https://helm.sh/docs/intro/install/)

---

## Step 1: Create an AWS IAM User and Attach Permissions

1. **Create an IAM User**:
   - Go to the [IAM Console](https://console.aws.amazon.com/iam/).
   - Create a user with programmatic access.
   - Save the **AWS Access Key ID** and **AWS Secret Access Key**.

2. **Attach Permissions**:
   - Attach the `AmazonEBSCSIDriverPolicy` to the user.
   - Alternatively, use the custom policy below:

     ```json
     {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Effect": "Allow",
           "Action": [
             "ec2:CreateVolume",
             "ec2:AttachVolume",
             "ec2:DeleteVolume",
             "ec2:DetachVolume",
             "ec2:DescribeVolumes",
             "ec2:DescribeVolumeAttribute",
             "ec2:DescribeVolumeStatus",
             "ec2:DescribeInstances",
             "ec2:DescribeInstanceAttribute",
             "ec2:DescribeInstanceStatus"
           ],
           "Resource": "*"
         }
       ]
     }
     ```

---

## Step 2: Install the AWS EBS CSI Driver

1. **Add the Helm Repository**:
   ```bash
   helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
   helm repo update
   ```

2. **Install the Driver**:
   ```bash
   helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
     --namespace kube-system \
     --set controller.serviceAccount.create=false
   ```

---

## Step 3: Configure AWS Credentials Using Kubernetes Secrets

1. **Create a Kubernetes Secret**:
   Replace `<AWS_ACCESS_KEY_ID>` and `<AWS_SECRET_ACCESS_KEY>` with your credentials:

   ```bash
   kubectl create secret generic aws-secret \
     --from-literal=AWS_ACCESS_KEY_ID=<AWS_ACCESS_KEY_ID> \
     --from-literal=AWS_SECRET_ACCESS_KEY=<AWS_SECRET_ACCESS_KEY> \
     -n kube-system
   ```

2. **Verify the Secret**:
   ```bash
   kubectl get secret aws-secret -n kube-system
   ```

3. **Edit the EBS CSI Driver Deployment**:
   Update the `ebs-csi-controller` Deployment:

   ```bash
   kubectl edit deployment ebs-csi-controller -n kube-system
   ```

   Add the following environment variables:

   ```yaml
   env:
     - name: AWS_ACCESS_KEY_ID
       valueFrom:
         secretKeyRef:
           name: aws-secret
           key: AWS_ACCESS_KEY_ID
     - name: AWS_SECRET_ACCESS_KEY
       valueFrom:
         secretKeyRef:
           name: aws-secret
           key: AWS_SECRET_ACCESS_KEY
   ```

---

## Step 4: Create a StorageClass for EBS

1. **Create a StorageClass**:
   Save the following YAML as `ebs-storageclass.yaml`:

   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: ebs-gp3
   provisioner: ebs.csi.aws.com
   parameters:
     type: gp3
     fsType: ext4
     encrypted: "true"
   reclaimPolicy: Delete
   volumeBindingMode: WaitForFirstConsumer
   ```

2. **Apply the StorageClass**:
   ```bash
   kubectl apply -f ebs-storageclass.yaml
   ```

---

## Step 5: Deploy the StatefulSet

1. **Create a StatefulSet**:
   Save the following YAML as `mysql-statefulset.yaml`:

   ```yaml
   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     name: mysql
     namespace: default
   spec:
     serviceName: "mysql"
     replicas: 1
     selector:
       matchLabels:
         app: mysql
     template:
       metadata:
         labels:
           app: mysql
       spec:
         containers:
           - name: mysql
             image: mysql:8
             env:
               - name: MYSQL_ROOT_PASSWORD
                 value: "rootpassword"
             ports:
               - containerPort: 3306
             volumeMounts:
               - name: mysql-data
                 mountPath: /var/lib/mysql
     volumeClaimTemplates:
       - metadata:
           name: mysql-data
         spec:
           accessModes: ["ReadWriteOnce"]
           resources:
             requests:
               storage: 10Gi
           storageClassName: ebs-gp3
   ```

2. **Apply the StatefulSet**:
   ```bash
   kubectl apply -f mysql-statefulset.yaml
   ```

---

## Step 6: Verify the Deployment

1. **Check StatefulSet Pods**:
   ```bash
   kubectl get pods
   ```

2. **Verify Persistent Volume Claims**:
   ```bash
   kubectl get pvc
   ```

3. **Inspect the Persistent Volume**:
   ```bash
   kubectl describe pv
   ```

4. **Validate on AWS**:
   - Navigate to **EC2 > Elastic Block Store > Volumes**.
   - Verify the EBS volume is created and attached.

---

## Additional Notes

- Ensure that your Kubernetes nodes are in the same AWS region and availability zone as the EBS volumes.
- The `WaitForFirstConsumer` volume binding mode ensures the volume is provisioned only when a pod is scheduled.
- Update MySQL credentials or other environment variables as per your requirements.

---

Feel free to contribute to or suggest improvements for this guide!
