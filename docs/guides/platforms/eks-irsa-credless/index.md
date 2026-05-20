---
title: Using IRSA with KubeStash on Amazon EKS
description: A guide on how to use EKS IRSA with KubeStash
menu:
  docs_{{ .version }}:
    identifier: platforms-eks-irsa-credless
    name: EKS IRSA
    parent: platforms
    weight: 10
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Using IRSA with KubeStash on Amazon EKS

This guide walks you through using [IAM Roles for Service Accounts (IRSA)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) on [Amazon EKS](https://aws.amazon.com/eks/) with KubeStash, eliminating the need to manage long-lived AWS access keys inside the cluster. We will:

1. Provision the IAM identities required by KubeStash and `aws-credential-manager`.
2. Install `aws-credential-manager` so backup and restore Jobs receive scoped credentials automatically.
3. Back up a sample DaemonSet to an [Amazon S3](https://aws.amazon.com/s3/) bucket.
4. Restore the workload from that backup.

## Before You Begin

- An EKS cluster with [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) enabled. If you don't have one, create it from the [Amazon EKS console](https://aws.amazon.com/eks/).
- `KubeStash` installed in the cluster. See the [setup guide](/docs/setup/README.md).
- An [Amazon S3 bucket](https://aws.amazon.com/s3/) for backup storage.
- The [`eksctl`](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html) and `aws` CLIs configured against your account.
- Familiarity with the following KubeStash concepts:
  - [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md)
  - [BackupSession](/docs/concepts/crds/backupsession/index.md)
  - [RestoreSession](/docs/concepts/crds/restoresession/index.md)
  - [BackupStorage](/docs/concepts/crds/backupstorage/index.md)
  - [Snapshot](/docs/concepts/crds/snapshot/index.md)

To keep resources isolated, this guide uses a dedicated namespace called `demo`:

```bash
$ kubectl create namespace demo
namespace/demo created
```

## Architecture Overview

`aws-credential-manager` is a Kubernetes controller that watches `ServiceAccount` objects carrying a *seed* IRSA annotation and provisions the underlying AWS IAM trust policy on demand. When KubeStash creates a Job for a `BackupSession` or `RestoreSession`, its short-lived `ServiceAccount` inherits the seed annotation. The controller then either reuses an existing managed role or creates a new one (e.g. `KubestashRole-0`, `KubestashRole-1`, …) and updates the trust policy so the Job pod can authenticate to AWS via IRSA.

This means:

- **No AWS access keys** are stored anywhere in the cluster.
- Each Job receives its own least-privilege IAM identity, scoped to its `ServiceAccount`.
- Trust policies are managed automatically; you never edit them by hand after initial setup.

## 1. Configure IAM for KubeStash

KubeStash needs an IAM role (the *seed role*) with permission to read and write objects in your S3 bucket. The `aws-credential-manager` will then derive per-Job roles from this seed role at runtime.

Throughout this guide we use the following placeholders. Replace them with your own values:

| Placeholder | Example                            |
| --- |------------------------------------|
| `<aws-account-id>` | `453903318604`                     |
| `<oidc-issuer-id>` | `Q0ZSA571J0QD87D390GCC678A40480HE` |
| `<region>` | `us-east-1`                        |
| `<bucket-name>` | `kubestash-backups`                |

### 1.1 Create the S3 access policy

Save the following document as `KubestashPolicy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation",
        "s3:ListBucket",
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::<bucket-name>",
        "arn:aws:s3:::<bucket-name>/*"
      ]
    }
  ]
}
```

Create the policy:

```bash
$ aws iam create-policy \
    --policy-name KubestashPolicy \
    --policy-document file://KubestashPolicy.json
```

### 1.2 Create the KubeStash seed role

Save the following trust policy as `KubestashRole.json`. The `sub` condition restricts which `ServiceAccount` can assume the role; replace `<kubestash-namespace>` and `<kubestash-operator-service-account-name>` with the actual values from your cluster.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<aws-account-id>:oidc-provider/oidc.eks.<region>.amazonaws.com/id/<oidc-issuer-id>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": {
          "oidc.eks.<region>.amazonaws.com/id/<oidc-issuer-id>:sub": [
            "system:serviceaccount:<kubestash-namespace>:<kubestash-operator-service-account-name>"
          ],
          "oidc.eks.<region>.amazonaws.com/id/<oidc-issuer-id>:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

Create the role and attach the policy:

```bash
$ aws iam create-role \
    --role-name KubestashRole \
    --assume-role-policy-document file://KubestashRole.json

$ aws iam attach-role-policy \
    --role-name KubestashRole \
    --policy-arn arn:aws:iam::<aws-account-id>:policy/KubestashPolicy
```

## 2. Install `aws-credential-manager`

`aws-credential-manager` is a controller that automatically reconciles IAM trust policies for any `ServiceAccount` annotated with a seed IRSA role. Install it via Helm:

```bash
$ helm upgrade --install aws-credential-manager appscode/aws-credential-manager \
    --namespace kubeops \
    --create-namespace \
    --set image.pullPolicy=Always \
    --set serviceAccount.create=true \
    --set serviceAccount.name=aws-credential-manager \
    --set apiserver.enableMutatingWebhook=true \
    --set apiserver.servingCerts.generate=true
```

Verify the controller pod is running:

```bash
$ kubectl get pods -n kubeops -l app.kubernetes.io/name=aws-credential-manager
NAME                                      READY   STATUS    RESTARTS   AGE
aws-credential-manager-77c5999bbf-vlzr6   1/1     Running   0          52s
```

### 2.1 Discover the cluster's OIDC issuer

```bash
$ aws eks describe-cluster \
    --name <cluster-name> \
    --region <region> \
    --query "cluster.identity.oidc.issuer" \
    --output text
https://oidc.eks.us-east-1.amazonaws.com/id/F0DFD593D0CD87D390ECC672A40480AE
```

### 2.2 Create the manager policy

The controller needs IAM permissions to create roles, attach policies, and update trust documents on your behalf. Save the following as `ManagerPolicy.json` (scope down further if your security posture requires it):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:GetRole",
        "iam:UpdateAssumeRolePolicy",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:DeleteRole",
        "iam:ListAttachedRolePolicies"
      ],
      "Resource": "arn:aws:iam::<aws-account-id>:role/KubestashRole*"
    }
  ]
}
```

Create the policy:

```bash
$ aws iam create-policy \
    --policy-name ManagerPolicy \
    --policy-document file://ManagerPolicy.json
```

### 2.3 Create the manager role

Save the following trust policy as `ManagerRole.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<aws-account-id>:oidc-provider/oidc.eks.<region>.amazonaws.com/id/<oidc-issuer-id>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": {
          "oidc.eks.<region>.amazonaws.com/id/<oidc-issuer-id>:sub": [
            "system:serviceaccount:kubeops:aws-credential-manager"
          ],
          "oidc.eks.<region>.amazonaws.com/id/<oidc-issuer-id>:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

Create the role and attach the manager policy:

```bash
$ aws iam create-role \
    --role-name ManagerRole \
    --assume-role-policy-document file://ManagerRole.json

$ aws iam attach-role-policy \
    --role-name ManagerRole \
    --policy-arn arn:aws:iam::<aws-account-id>:policy/ManagerPolicy
```

### 2.4 Bind the role to the controller's `ServiceAccount`

Annotate the controller's `ServiceAccount` with the role ARN, then restart the pod so EKS injects the IRSA token:

```bash
$ kubectl annotate -n kubeops sa aws-credential-manager \
    eks.amazonaws.com/role-arn=arn:aws:iam::<aws-account-id>:role/ManagerRole

$ kubectl rollout restart -n kubeops deployment aws-credential-manager
```

After the restart, confirm IRSA is wired up. The relevant environment variables and projected token volume should appear on the controller pod:

```bash
$ kubectl get pod -n kubeops -l app.kubernetes.io/name=aws-credential-manager -o yaml
```

```yaml
spec:
  containers:
  - env:
    - name: AWS_STS_REGIONAL_ENDPOINTS
      value: regional
    - name: AWS_DEFAULT_REGION
      value: us-east-1
    - name: AWS_REGION
      value: us-east-1
    - name: AWS_ROLE_ARN
      value: arn:aws:iam::<aws-account-id>:role/ManagerRole
    - name: AWS_WEB_IDENTITY_TOKEN_FILE
      value: /var/run/secrets/eks.amazonaws.com/serviceaccount/token
    volumeMounts:
    - mountPath: /var/run/secrets/eks.amazonaws.com/serviceaccount
      name: aws-iam-token
      readOnly: true
  volumes:
  - name: aws-iam-token
    projected:
      sources:
      - serviceAccountToken:
          audience: sts.amazonaws.com
          expirationSeconds: 86400
          path: token
```

## 3. Configure the KubeStash Operator

Annotate the KubeStash operator `ServiceAccount` with the seed role you created in section 1, then restart the operator and webhook so IRSA takes effect:

```bash
$ kubectl annotate -n <kubestash-namespace> sa <kubestash-operator-service-account-name> \
    eks.amazonaws.com/role-arn=arn:aws:iam::<aws-account-id>:role/KubestashRole

$ kubectl delete pod -n <kubestash-namespace> --all
```

Verify that IRSA credentials are projected into the operator pod:

```bash
$ kubectl get pod -n <kubestash-namespace> -l component=operator -o yaml
```

```yaml
spec:
  containers:
  - env:
    - name: AWS_ROLE_ARN
      value: arn:aws:iam::<aws-account-id>:role/KubestashRole
    - name: AWS_WEB_IDENTITY_TOKEN_FILE
      value: /var/run/secrets/eks.amazonaws.com/serviceaccount/token
    volumeMounts:
    - mountPath: /var/run/secrets/eks.amazonaws.com/serviceaccount
      name: aws-iam-token
      readOnly: true
```

## 4. Prepare the Backend

### 4.1 Create the BackupStorage

`BackupStorage` describes where backed-up data lands. Because we are authenticating via IRSA, no AWS access key or secret key is required.

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: BackupStorage
metadata:
  name: s3-storage
  namespace: demo
spec:
  storage:
    provider: s3
    s3:
      endpoint: https://s3.us-east-1.amazonaws.com
      bucket: kubestash-backups
      prefix: ace/production/backups/demo
      region: us-east-1
  usagePolicy:
    allowedNamespaces:
      from: All
  default: true
  deletionPolicy: WipeOut
```

```bash
$ kubectl apply -f backupstorage.yaml
backupstorage.storage.kubestash.com/s3-storage created

$ kubectl get backupstorage -n demo
NAME         PROVIDER   DEFAULT   DELETION-POLICY   TOTAL-SIZE   PHASE   AGE
s3-storage   s3         true      WipeOut                        Ready   7s
```

### 4.2 Create the RetentionPolicy

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: RetentionPolicy
metadata:
  name: demo-retention
  namespace: demo
spec:
  default: true
  failedSnapshots:
    last: 2
  successfulSnapshots:
    last: 5
  maxRetentionPeriod: 2mo
  usagePolicy:
    allowedNamespaces:
      from: All
```

```bash
$ kubectl apply -f retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

## 5. Backup

### 5.1 Create the encryption secret

KubeStash uses Restic under the hood and encrypts backups with the password you supply:

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encrypt-secret \
    --from-file=./RESTIC_PASSWORD
secret/encrypt-secret created
```

### 5.2 Create the BackupConfiguration

The following `BackupConfiguration` schedules a backup of the `ks-demo` DaemonSet every five minutes:

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: sample-backup-daemon
  namespace: demo
spec:
  target:
    apiGroup: apps
    kind: DaemonSet
    name: ks-demo
    namespace: demo
  backends:
  - name: s3-backend
    storageRef:
      name: s3-storage
      namespace: demo
    retentionPolicy:
      name: demo-retention
      namespace: demo
  sessions:
  - name: demo-session
    scheduler:
      schedule: "*/5 * * * *"
      jobTemplate:
        backoffLimit: 1
    repositories:
    - name: s3-demo-repo
      backend: s3-backend
      directory: /data/ks-demo
      encryptionSecret:
        name: encrypt-secret
        namespace: demo
    addon:
      name: workload-addon
      tasks:
      - name: logical-backup
        params:
          paths: /source/data
    retryConfig:
      maxRetry: 2
      delay: 1m
```

```bash
$ kubectl apply -f backupconfiguration.yaml
backupconfiguration.core.kubestash.com/sample-backup-daemon created
```

### 5.3 Verify the backup setup

The `BackupConfiguration` should reach the `Ready` phase, and KubeStash should provision the `Repository`:

```bash
$ kubectl get backupconfiguration -n demo
NAME                   PHASE   PAUSED   AGE
sample-backup-daemon   Ready            61s

$ kubectl get repository -n demo
NAME           INTEGRITY   SNAPSHOT-COUNT   SIZE     PHASE   LAST-SUCCESSFUL-BACKUP   AGE
s3-demo-repo                0                0 B      Ready                            3m
```

### 5.4 Observe `aws-credential-manager` provisioning the Job's IAM identity

When the first scheduled `BackupSession` fires, KubeStash creates a Job whose `ServiceAccount` inherits the seed annotation. `aws-credential-manager` reconciles that `ServiceAccount`, derives a managed role (`KubestashRole-0`), and updates the trust policy:

```bash
$ kubectl logs -f -n kubeops deploy/aws-credential-manager
I0506 12:39:48.381 irsa.go:154] service account stash/kubestash-kubestash-operator successfully reconciled for role arn:aws:iam::<aws-account-id>:role/KubestashRole
I0506 12:55:49.550 serviceaccount_controller.go:62] reconciling for service account demo/sample-backup-daemon-demo-session-1778072138
I0506 12:55:49.718 irsa.go:454] creating role KubestashRole-0 with OIDC arn:aws:iam::<aws-account-id>:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/<oidc-issuer-id> for demo/sample-backup-daemon-demo-session-1778072138
I0506 12:55:49.903 irsa.go:508] attached policy arn:aws:iam::<aws-account-id>:policy/KubestashPolicy to role KubestashRole-0
I0506 12:55:49.903 irsa.go:445] role KubestashRole-0 has space for service account demo/sample-backup-daemon-demo-session-1778072138
I0506 12:55:50.003 irsa.go:152] service account demo/sample-backup-daemon-demo-session-1778072138 has seed role annotation, added AWS IRSA role annotation
I0506 12:55:50.003 irsa.go:154] service account demo/sample-backup-daemon-demo-session-1778072138 successfully reconciled for role arn:aws:iam::<aws-account-id>:role/KubestashRole-0
```

Confirm the trust policy on `KubestashRole-0` now includes the Job's `ServiceAccount`:

```bash
$ aws iam get-role \
    --role-name KubestashRole-0 \
    --query "Role.AssumeRolePolicyDocument" \
    --output json
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<aws-account-id>:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/<oidc-issuer-id>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": {
          "oidc.eks.us-east-1.amazonaws.com/id/<oidc-issuer-id>:sub": [
            "system:serviceaccount:demo:sample-backup-daemon-demo-session-1778072138",
            "system:serviceaccount:demo:retention-policy-sample-backup-daemon-demo-session-1778072402"
          ],
          "oidc.eks.us-east-1.amazonaws.com/id/<oidc-issuer-id>:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

### 5.5 Verify the BackupSession succeeded

```bash
$ kubectl get backupsession -n demo
NAME                                           INVOKER-TYPE          INVOKER-NAME           PHASE       DURATION   AGE
sample-backup-daemon-demo-session-1778072138   BackupConfiguration   sample-backup-daemon   Succeeded   48s        6m44s
sample-backup-daemon-demo-session-1778072402   BackupConfiguration   sample-backup-daemon   Succeeded   42s        2m31s
```

The backup pod runs an `aws-bucket-access-test` init container that validates IRSA-derived credentials before the main backup container starts:

```bash
$ kubectl logs -n demo -c aws-bucket-access-test sample-backup-daemon-demo-session-1778072138-0-xspwf
2026-05-06 12:55:58 === Attempt 1 (elapsed: 0s) ===
2026-05-06 12:55:58 Testing permissions on bucket: kubestash-backups
2026-05-06 12:55:58 Uploading test object to s3://kubestash-backups/permission-check/test-object-1778072158-1
2026-05-06 12:55:59 Verifying existence of test object
2026-05-06 12:56:01 Deleting test object
2026-05-06 12:56:03 S3 permission check PASSED for bucket kubestash-backups
```

The main backup container completes successfully and reports the snapshot stats:

```bash
$ kubectl logs -n demo sample-backup-daemon-demo-session-1778072138-0-xspwf -c logical-backup-0
I0506 12:56:08.827 credentials.go:113] Successfully fetched S3 credentials for demo/s3-storage
I0506 12:56:13.547 commands.go:163] Backing up target data
[golang-sh]$ restic backup /source/data --quiet --json --host dump --cache-dir /kubestash-tmp/restic-cache
{"message_type":"summary","files_new":4821,"files_changed":0,"files_unmodified":0,"data_added":3623874560,"data_added_packed":2587294310,"total_files_processed":4821,"total_bytes_processed":4118528000,"total_duration":182.41,"snapshot_id":"106554e1a07251aaf046e61606f3a4f3d9573010493118b9f778db5698bea391"}
I0506 12:59:18.671 commands.go:321] Reading repository status
[golang-sh]$ restic stats --quiet --json --mode raw-data --no-lock
{"total_size":2587294310,"total_uncompressed_size":3623874560,"compression_ratio":1.40,"compression_progress":100,"compression_space_saving":28.6,"total_blob_count":18432,"snapshots_count":1}
```

Inspect the `Repository` to confirm the data size was committed to S3:

```bash
$ kubectl get repository -n demo s3-demo-repo
NAME           INTEGRITY   SNAPSHOT-COUNT   SIZE       PHASE   LAST-SUCCESSFUL-BACKUP   AGE
s3-demo-repo   true        1                2.41 GiB   Ready   3m36s                    4m59s
```

List the resulting `Snapshot`:

```bash
$ kubectl get snapshot -n demo -l kubestash.com/repo-name=s3-demo-repo
NAME                                                        REPOSITORY     SESSION        SNAPSHOT-TIME          DELETION-POLICY   PHASE       AGE
s3-demo-repo-sample-backup-daemon-demo-session-1778072138   s3-demo-repo   demo-session   2026-05-06T12:55:49Z   Delete            Succeeded   5m23s
```

> KubeStash labels each `Snapshot` with `kubestash.com/app-ref-kind`, `kubestash.com/app-ref-name`, `kubestash.com/app-ref-namespace`, and `kubestash.com/repo-name`. Use these labels to scope queries to a specific workload or repository.

For a DaemonSet, KubeStash backs up each pod independently. Inspecting the `Snapshot` reveals one component per node, plus a roll-up size:

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  name: s3-demo-repo-sample-backup-daemon-demo-session-1778072138
  namespace: demo
status:
  components:
    dump-ip-192-168-2-154.us-east-1.compute.internal:
      driver: Restic
      duration: 64.21s
      integrity: true
      phase: Succeeded
      resticStats:
      - hostPath: /source/data
        size: 1.4 GiB
        uploaded: 982.7 MiB
      size: 1.4 GiB
    dump-ip-192-168-16-32.us-east-1.compute.internal:
      driver: Restic
      duration: 58.97s
      integrity: true
      phase: Succeeded
      resticStats:
      - hostPath: /source/data
        size: 512 MiB
        uploaded: 367.9 MiB
      size: 512 MiB
    dump-ip-192-168-53-129.us-east-1.compute.internal:
      driver: Restic
      duration: 59.23s
      integrity: true
      phase: Succeeded
      resticStats:
      - hostPath: /source/data
        size: 524 MiB
        uploaded: 1.21 GiB
      size: 524 MiB
```

> Backed-up data in S3 is encrypted at rest with the Restic password supplied via `encrypt-secret`. It cannot be read without the password — store it securely.

## 6. Restore

This section restores the DaemonSet from the most recent `Snapshot`. The same IRSA flow you saw on the backup path applies here: when the restore Job is created, `aws-credential-manager` reconciles its `ServiceAccount` and grants it the necessary permissions.

### 6.1 Simulate data loss

```bash
$ kubectl exec -it -n demo ks-demo-kjn7t -- sh
/ # rm /source/data/data.txt
/ # cat /source/data/data.txt
cat: can't open '/source/data/data.txt': No such file or directory
/ # exit
```

### 6.2 Create the RestoreSession

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: RestoreSession
metadata:
  name: app-restore
  namespace: demo
spec:
  dataSource:
    repository: s3-demo-repo
    snapshot: latest          # or the specific snapshot ID you want
    encryptionSecret:
      name: encrypt-secret
      namespace: demo
  addon:
    name: workload-addon
    tasks:
      - name: logical-backup-restore
```

```bash
$ kubectl apply -f restoresession.yaml
restoresession.core.kubestash.com/app-restore created
```

### 6.3 Observe `aws-credential-manager` granting the restore `ServiceAccount` permission

KubeStash creates a restore Job with its own `ServiceAccount` (e.g. `app-restore-restore`). `aws-credential-manager` picks it up, attaches it to a managed role with the `KubestashPolicy`, and updates the trust policy so the restore pod can authenticate to S3 via IRSA:

```bash
$ kubectl logs -f -n kubeops deploy/aws-credential-manager
I0506 13:14:02.211 serviceaccount_controller.go:62] reconciling for service account demo/app-restore-restore
I0506 13:14:02.234 serviceaccount_controller.go:100] found eks auth mode:
W0506 13:14:02.234 serviceaccount_controller.go:105] no auth mode specified, trying with irsa ...
I0506 13:14:02.296 mutate_job.go:63] ServiceAccount has seed role annotation but missing required AWS arn annotations, suspending Job demo/app-restore-restore
I0506 13:14:02.297 mutate_job.go:76] Mutating Job for creation demo/app-restore-restore
I0506 13:14:02.297 mutate_job.go:82] Successfully injected init container into demo/app-restore-restore
I0506 13:14:02.418 irsa.go:445] role KubestashRole-0 has space for service account demo/app-restore-restore
I0506 13:14:02.418 irsa.go:139] reconciling service account demo/app-restore-restore for role KubestashRole-0
I0506 13:14:02.487 irsa.go:278] role KubestashRole-0 trust policy updated to include demo/app-restore-restore
I0506 13:14:02.541 irsa.go:152] service account demo/app-restore-restore has seed role annotation, added AWS IRSA role annotation
I0506 13:14:02.541 irsa.go:154] service account demo/app-restore-restore successfully reconciled for role arn:aws:iam::<aws-account-id>:role/KubestashRole-0
```

The restore `ServiceAccount` now appears in `KubestashRole-0`'s trust policy alongside the backup `ServiceAccount`s:

```bash
$ aws iam get-role \
    --role-name KubestashRole-0 \
    --query "Role.AssumeRolePolicyDocument.Statement[0].Condition.StringLike" \
    --output json
{
  "oidc.eks.us-east-1.amazonaws.com/id/<oidc-issuer-id>:sub": [
    "system:serviceaccount:demo:sample-backup-daemon-demo-session-1778072138",
    "system:serviceaccount:demo:sample-backup-daemon-demo-session-1778072402",
    "system:serviceaccount:demo:app-restore-restore"
  ],
  "oidc.eks.us-east-1.amazonaws.com/id/<oidc-issuer-id>:aud": "sts.amazonaws.com"
}
```

### 6.4 Verify the restore

```bash
$ kubectl get restoresession -n demo
NAME          REPOSITORY     FAILURE-POLICY   PHASE       DURATION   AGE
app-restore   s3-demo-repo                    Succeeded   38s        53s
```

Confirm the deleted file has been restored:

```bash
$ kubectl exec -it -n demo ks-demo-kjn7t -- cat /source/data/data.txt
sample_data
```

## 7. Cleanup

Remove the resources created by this guide:

```bash
$ kubectl delete -n demo backupconfiguration sample-backup-daemon
$ kubectl delete -n demo restoresession app-restore
$ kubectl delete -n demo secret encrypt-secret
$ kubectl delete -n demo backupstorage s3-storage
$ kubectl delete -n demo retentionpolicy demo-retention
$ kubectl delete -n demo daemonset ks-demo
$ kubectl delete namespace demo
```

If you no longer need the IAM resources, remove them in this order (managed roles first, then policies):

```bash
$ for role in $(aws iam list-roles --query "Roles[?starts_with(RoleName, 'KubestashRole-')].RoleName" --output text); do
    aws iam detach-role-policy --role-name "$role" --policy-arn arn:aws:iam::<aws-account-id>:policy/KubestashPolicy
    aws iam delete-role --role-name "$role"
  done

$ aws iam detach-role-policy --role-name KubestashRole --policy-arn arn:aws:iam::<aws-account-id>:policy/KubestashPolicy
$ aws iam delete-role --role-name KubestashRole

$ aws iam detach-role-policy --role-name ManagerRole --policy-arn arn:aws:iam::<aws-account-id>:policy/ManagerPolicy
$ aws iam delete-role --role-name ManagerRole

$ aws iam delete-policy --policy-arn arn:aws:iam::<aws-account-id>:policy/KubestashPolicy
$ aws iam delete-policy --policy-arn arn:aws:iam::<aws-account-id>:policy/ManagerPolicy
```
