---
title: Auto Backup Workload | KubeStash
description: An step by step guide on how to configure automatic backup for workloads.
menu:
  docs_{{ .version }}:
    identifier: auto-backup-workload
    name: Auto Backup for Workloads
    parent: auto-backup
    weight: 20
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Auto Backup for Workloads

This tutorial will show you how to configure automatic backup for Kubernetes workloads. Here, we are going to show a demo on how we can backup Deployments, StatefulSets, and DaemonSets using a common blueprint.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

- Install `KubeStash` in your cluster following the steps [here](/docs/setup/README.md).

- You should be familiar with the following KubeStash concepts:
  - [BackupStorage](/docs/concepts/crds/backupstorage/index.md)
  - [RetentionPolicy](/docs/concepts/crds/retentionpolicy/index.md)
  - [BackupConfiguration](/docs/concepts/crds/backupconfiguration/index.md)
  - [BackupBlueprint](/docs/concepts/crds/backupblueprint/index.md)
  - [BackupSession](/docs/concepts/crds/backupsession/index.md)
  - [Snapshot](/docs/concepts/crds/snapshot/index.md)

To keep things isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```bash
$ kubectl create namespace demo
namespace/demo created
```

## Prepare Backup Blueprint

We are going to use [GCS Backend](/docs/guides/backends/gcs/index.md) to store the backed up data. You can use any supported backend you prefer. You just have to configure `BackupStorage` and storage secret to match your backend. To learn which backends are supported by KubeStash and how to configure them, please visit [here](/docs/guides/backends/overview/index.md).

**Create Storage Secret:**

At first, let's create a Storage Secret for the GCS backend. `BackupStorage` and storage secret must be in the same namespace.

```bash
$ echo -n '<your-project-id>' > GOOGLE_PROJECT_ID
$ mv downloaded-sa-json.key GOOGLE_SERVICE_ACCOUNT_JSON_KEY
$ kubectl create secret generic -n demo gcs-secret \
    --from-file=./GOOGLE_PROJECT_ID \
    --from-file=./GOOGLE_SERVICE_ACCOUNT_JSON_KEY
secret/gcs-secret created
```

**Create BackupStorage**

Now, let's create a `BackupStorage` to specify the backend information where the backed up data will be stored.

Below is the YAML of the `BackupStorage` object that we are going to create,

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: BackupStorage
metadata:
  name: gcs-storage
  namespace: demo
spec:
  storage:
    provider: gcs
    gcs:
      bucket: kubestash-demo
      prefix: demo
      secretName: gcs-secret 
  usagePolicy:
    allowedNamespaces:
      from: All
  default: true
  deletionPolicy: WipeOut
```

Let’s create the above `BackupStorage`,
```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/auto-backup/workload/examples/backupstorage.yaml
backupstorage.storage.kubestash.com/gcs-storage created
```

**Create RetentionPolicy**

Now, let's create a `RetentionPolicy` to specify how the old Snapshots should be cleaned up.

Below is the YAML of the `RetentionPolicy` object that we are going to create,

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
  maxRetentionPeriod: 2mo
  successfulSnapshots:
    last: 5
  usagePolicy:
    allowedNamespaces:
      from: All
```

Let’s create the above `RetentionPolicy`,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/auto-backup/workload/examples/retentionpolicy.yaml
retentionpolicy.storage.kubestash.com/demo-retention created
```

**Create Encryption Secret:**

Let's create a secret called `encrypt-secret` with the Restic password,

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ kubectl create secret generic -n demo encrypt-secret \
    --from-file=./RESTIC_PASSWORD \
secret "encrypt-secret" created
```

**Create BackupBlueprint:**

Now, we have to create a `BackupBlueprint` CR with a blueprint for `BackupConfiguration` object.

Below is the YAML of the `BackupBlueprint` object that we are going to create,

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupBlueprint
metadata:
  name: workload-backup-blueprint
  namespace: demo
spec:
  usagePolicy:
    allowedNamespaces:
      from: All
  backupConfigurationTemplate:
    deletionPolicy: OnDelete
    backends:
      - name: gcs-backend
        storageRef:
          name: gcs-storage
          namespace: demo
        retentionPolicy:
          name: demo-retention
          namespace: demo
    sessions:
      - name: frequent-backup
        sessionHistoryLimit: 3
        scheduler:
          schedule: "*/5 * * * *"
          jobTemplate:
            backoffLimit: 1
        repositories:
          - name: ${repoName}
            backend: gcs-backend
            directory: ${namespace}/${targetName}
            encryptionSecret:
              name: encrypt-secret
              namespace: demo
        addon:
          name: workload-addon
          tasks:
            - name: logical-backup
              params:
                paths: ${paths}
        retryConfig:
          maxRetry: 2
          delay: 1m
```

Note that we have used some variables (format: `${<variable name>}`) in different fields. KubeStash will substitute these variables with values from the respective target's annotations. You're free to use any variables you like.

Let's create the `BackupBlueprint` that we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/auto-backup/workload/examples/backupblueprint.yaml
backupblueprint.core.kubestash.com/workload-backup-blueprint created
```

Now, automatic backup is configured for Kubernetes workloads (`Deployment`, `StatefulSet`, `DaemonSet` etc.). We just have to add some annotations to the targeted workload to enable periodic backup.

**Available Auto-Backup Annotations:**

You have to add the auto-backup annotations to the workload that you want to backup. The following auto-backup annotations are available:

Required Annotations:

- **BackupBlueprint Name:** Specifies the name of the `BackupBlueprint` to be used for this workload's backup.

```yaml
blueprint.kubestash.com/name: <BackupBlueprint name>
```

- **BackupBlueprint Namespace:** Specifies the namespace where the `BackupBlueprint` resides.

```yaml
blueprint.kubestash.com/namespace: <BackupBlueprint namespace>
```

Optional Annotations:

- **Session Names:** Defines which sessions from the `BackupBlueprint` should be used for the `BackupConfiguration`. If you don’t specify this annotation, all sessions from the `BackupBlueprint` will be used. For multiple sessions, you can provide comma (,) seperated sessions name.

```yaml
 blueprint.kubestash.com/sessions: <Sessions name>
```

- **Variables:** Allows you to customize the `BackupConfiguration` by providing values for variables defined within the `BackupBlueprint`. You can define as many annotations as needed for variables.

```yaml
variables.kubestash.com/<variable-name>: <Variable value>
```

Example:

Assuming you are using a variable named `schedule` for a session schedule in the `BackupBlueprint`, you can configure different backup schedules for each target application using annotations like this:
```yaml
variables.kubestash.com/schedule: "*/5 * * * *"
```

## Backup StatefulSet

Now, we are going to backup a StatefulSet using the blueprint we have configured earlier.

**Create StatefulSet:**

We are going to create a StatefulSet with 3 replicas. We are going to configure the StatefulSet to generate sample data in each replica.

Below is the YAML of the StatefulSet that we are going to create,

```yaml
apiVersion: v1
kind: Service
metadata:
  name: busybox
  namespace: demo
spec:
  ports:
    - name: http
      port: 80
      targetPort: 0
  selector:
    app: demo-busybox
  clusterIP: None
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sample-sts
  namespace: demo
  labels:
    app: demo-busybox
  annotations:
    blueprint.kubestash.com/name: workload-backup-blueprint
    blueprint.kubestash.com/namespace: demo
    variables.kubestash.com/targetName: sample-sts
    variables.kubestash.com/namespace: demo
    variables.kubestash.com/repoName: sts-repo
    variables.kubestash.com/paths: /source/data
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-busybox
  serviceName: busybox
  template:
    metadata:
      labels:
        app: demo-busybox
    spec:
      containers:
        - name: busybox
          image: busybox
          command: ["/bin/sh", "-c","echo $(POD_NAME) > /source/data/pod-name.txt && sleep 3000"]
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          volumeMounts:
            - name: source-data
              mountPath: "/source/data"
          imagePullPolicy: IfNotPresent
  volumeClaimTemplates:
    - metadata:
        name: source-data
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 256Mi
```

Notice the `metadata.annotations` field, we have specified the automatic backup specific annotations. We have specified BackupBlueprint name `workload-backup-blueprint` and namespace `demo` for creating `BackupConfiguration` for this StatefulSet.

Let's create the above StatefulSet,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/auto-backup/workload/examples/statefulset.yaml
service/busybox created
statefulset.apps/sample-sts created
```

If everything goes well, KubeStash will create a `BackupConfiguration` with the name in the following format: `<workload-kind>-<workload-name>`. If the `BackupConfiguration` is created in different namespace other than targeted workload's namespace, KubeStash will create a `BackupConfiguration` with the name in the following format: `<workload-namespace>-<workload-kind>-<workload-name>`

**Verify BackupConfiguration:**

If everything goes well, KubeStash should create a `BackupConfiguration` for our StatefulSet and the phase of that `BackupConfiguration` should be `Ready`. Verify that the `BackupConfiguration` has been created by the following command,

```bash
$ kubectl get backupconfiguration -n demo
NAME                    PHASE   PAUSED   AGE
statefulset-sample-sts  Ready            19s
```

Let's check the YAML of this `BackupConfiguration`,

```bash
$ kubectl get backupconfiguration -n demo statefulset-sample-sts -o yaml
```

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  labels:
    app.kubernetes.io/managed-by: kubestash.com
    kubestash.com/invoker-name: workload-backup-blueprint
    kubestash.com/invoker-namespace: demo
  name: statefulset-sample-sts
  namespace: demo
  ...
spec:
  backends:
    - name: gcs-backend
      retentionPolicy:
        name: demo-retention
        namespace: demo
      storageRef:
        name: gcs-storage
        namespace: demo
  sessions:
    - addon:
        name: workload-addon
        tasks:
          - name: logical-backup
            params:
              paths: /source/data
      failurePolicy: Fail
      name: frequent-backup
      repositories:
        - backend: gcs-backend
          directory: demo/sample-sts
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
          name: sts-repo
      retryConfig:
        delay: 1m0s
        maxRetry: 2
      ...
        schedule: '*/5 * * * *'
      sessionHistoryLimit: 3
  target:
    apiGroup: apps
    kind: StatefulSet
    name: sample-sts
    namespace: demo
status:
  ...
```

Notice that the `spec.target` is pointing to the `sample-sts` StatefulSet. Also, notice that the `spec.sessions[*].addon.tasks[*].params.paths`, `spec.sessions[*].repositories[*].name` and `spec.sessions[*].repositories[*].directory` fields have been populated with the information we had provided as annotation of the StatefulSet.

**Wait for BackupSession:**

Now, wait for the next backup schedule. Run the following command to watch `BackupSession` CR:

```bash
$ watch kubectl get backupsession -n demo
Every 2.0s: kubectl get backupsession -n demo                                   AppsCode-PC-03: Fri Jan  5 15:06:20 2024

NAME                                                INVOKER-TYPE          INVOKER-NAME             PHASE       DURATION   AGE
statefulset-sample-sts-frequent-backup-1704431400   BackupConfiguration   statefulset-sample-sts   Succeeded              61s
```

**Verify Backup:**

When backup session is completed, KubeStash will update the respective `Repository` to reflect the latest state of backed up data.

Run the following command to check if the snapshots are stored in the backend,

```bash
$ kubectl get repository -n demo sts-repo
NAME       INTEGRITY   SNAPSHOT-COUNT   SIZE        PHASE   LAST-SUCCESSFUL-BACKUP   AGE
sts-repo   true        1                2.487 KiB   Ready   23s                      23s
```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot` which represents the state of a backup run for an application.

```bash
$ kubectl get snapshots -n demo -l=kubestash.com/repo-name=sts-repo
NAME                                                         REPOSITORY   SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       VERIFICATION-STATUS   AGE
sts-repo-statefulset-sample-sts-frequent-backup-1704431400   sts-repo     frequent-backup   2024-01-05T05:10:09Z   Delete            Succeeded                         72s
```

> Note: KubeStash creates a `Snapshot` with the following labels:
> - `kubestash.com/app-ref-kind: <workload-kind>`
> - `kubestash.com/app-ref-name: <workload-name>`
> - `kubestash.com/app-ref-namespace: <workload-namespace>`
> - `kubestash.com/repo-name: <repository-name>`
>
> These labels can be used to watch only the `Snapshot`s related to our desired Workload or `Repository`.

If we check the YAML of the `Snapshot`, we can find the information about the backed up components of the StatefulSet.

```bash
$ kubectl get snapshots -n demo sts-repo-statefulset-sample-sts-frequent-backup-1704431400 -oyaml
```

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  labels:
    kubestash.com/app-ref-kind: StatefulSet
    kubestash.com/app-ref-name: sample-sts
    kubestash.com/app-ref-namespace: demo
    kubestash.com/repo-name: sts-repo
  name: sts-repo-statefulset-sample-sts-frequent-backup-1704431400
  namespace: demo
spec:
  ...
status:
  components:
    dump-pod-0:
      driver: Restic
      duration: 6.232547841s
      integrity: true
      path: repository/v1/frequent-backup/pod-0
      phase: Succeeded
      resticStats:
      - hostPath: /source/data
        id: 07abd0bd6d28406141709d01b343dba35fea3198321cd60dbe1710088c239153
        size: 26 B
        uploaded: 1.397 KiB
      size: 848 B
    dump-pod-1:
      driver: Restic
      duration: 6.384220849s
      integrity: true
      path: repository/v1/frequent-backup/pod-1
      phase: Succeeded
      resticStats:
      - hostPath: /source/data
        id: 06d34f5fa7a2251408a7d846fb0616d5156a387106fe344854ab4a095c79bfa3
        size: 26 B
        uploaded: 1.397 KiB
      size: 849 B
    dump-pod-2:
      driver: Restic
      duration: 6.18345431s
      integrity: true
      path: repository/v1/frequent-backup/pod-2
      phase: Succeeded
      resticStats:
      - hostPath: /source/data
        id: caa69ec25c20fcfe26624ecd30032054b0b68789ada42f2c8eb8554e60d5fc3a
        size: 26 B
        uploaded: 1.395 KiB
      size: 850 B
 ...
```
> For StatefulSet, KubeStash takes backup from every pod of the StatefulSet. Since we are using three replicas, three components have been backed up. For logical backup, KubeStash uses `dump-pod-<ordinal-value>` as the component name where `<ordinal-value>` corresponds to the pod's ordinal number for the StatefulSet.

Now, if we navigate to the GCS bucket, we will see the backed up data stored in the `demo/demo/sample-sts/repository/v1/frequent-backup/dump-pod-<ordinal-value>` directory. KubeStash also keeps the backup for `Snapshot` YAMLs, which can be found in the `demo/demo/sample-sts/snapshots` directory.

> KubeStash keeps all the dumped data encrypted in the backup directory meaning the dumped files won't contain any readable data until decryption.

## Backup Deployment

Now, we are going to backup a Deployment with the same blueprint we have used to backup StatefulSet in the previous section.

**Create Deployment:**

We are going to create a Deployment.

Below is the YAML of the Deployment that we are going to create,

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kubestash-pvc-1
  namespace: demo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kubestash-pvc-2
  namespace: demo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubestash-demo
  name: kubestash-demo
  namespace: demo
  annotations:
    blueprint.kubestash.com/name: workload-backup-blueprint
    blueprint.kubestash.com/namespace: demo
    variables.kubestash.com/targetName: kubestash-demo
    variables.kubestash.com/namespace: demo
    variables.kubestash.com/repoName: dep-repo
    variables.kubestash.com/paths: /source/data-1,/source/data-2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubestash-demo
  template:
    metadata:
      labels:
        app: kubestash-demo
      name: busybox
    spec:
      containers:
        - args:
            - sleep
            - "3600"
          image: busybox
          imagePullPolicy: IfNotPresent
          name: busybox
          volumeMounts:
            - mountPath: /source/data-1
              name: source-data-1
            - mountPath: /source/data-2
              name: source-data-2
      restartPolicy: Always
      volumes:
        - name: source-data-1
          persistentVolumeClaim:
            claimName: kubestash-pvc-1
        - name: source-data-2
          persistentVolumeClaim:
            claimName: kubestash-pvc-2
```

Notice the `metadata.annotations` field. We have specified automatic backup specific annotations similarly as we had specified in the StatefulSet in the previous section.

Let's create the Deployment we have created above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/auto-backup/workload/examples/deployment.yaml
persistentvolumeclaim/kubestash-pvc-1 created
persistentvolumeclaim/kubestash-pvc-2 created
deployment.apps/kubestash-demo created
```

**Verify BackupConfiguration:**

Verify that a `BackupConfiguration` has been created and in `Ready` Phase for this Deployment using the following command,

```bash
$ kubectl get backupconfiguration -n demo
NAME                        PHASE   PAUSED   AGE
deployment-kubestash-demo   Ready            2m
statefulset-sample-sts      Ready            1h
```

Here, `deployment-kubestash-demo` has been created for the Deployment `kubestash-demo`. You can check the YAML of this `BackupConfiguration` to see that the target field is pointing to this Deployment.

```bash
$ kubectl get backupconfiguration -n demo deployment-kubestash-demo -o yaml
```

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: deployment-kubestash-demo
  namespace: demo
spec:
  ...
  sessions:
  - addon:
      name: workload-addon
      tasks:
      - name: logical-backup
        params:
          paths: /source/data-1,/source/data-2
    failurePolicy: Fail
    name: frequent-backup
    repositories:
    - backend: gcs-backend
      directory: demo/kubestash-demo
      encryptionSecret:
        name: encrypt-secret
        namespace: demo
      name: dep-repo
    retryConfig:
      delay: 1m0s
      maxRetry: 2
    ...
    sessionHistoryLimit: 3
  target:
    apiGroup: apps
    kind: Deployment
    name: kubestash-demo
    namespace: demo
status:
  ...
```

**Wait for BackupSession:**

Now, wait for the next backup schedule. Watch the `BackupSession` of the BackupConfiguration `deployment-kubestash-demo` using the following command,

```bash
$ watch kubectl get backupsession -n demo -l=kubestash.com/invoker-name=deployment-kubestash-demo
Every 1.0s: kubectl get backupsession -n demo -l=kubestash.com/invoker-na...  workstation: AppsCode-PC-03: Fri Jan  5 19:30:24 2024

NAME                                                   INVOKER-TYPE          INVOKER-NAME                PHASE       DURATION   AGE
deployment-kubestash-demo-frequent-backup-1704460503   BackupConfiguration   deployment-kubestash-demo   Succeeded              2m21s
```

**Verify Backup:**

Once the backup session is completed, verify that the `Repository` has been updated to reflect the backup using the following command,

```bash
$ kubectl get repository -n demo dep-repo
NAME       INTEGRITY   SNAPSHOT-COUNT   SIZE        PHASE   LAST-SUCCESSFUL-BACKUP   AGE
dep-repo   true        1                1.386 KiB   Ready   16m                      46m
```
At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot` which represents the state of a backup run for an application.

```bash
$ kubectl get snapshots -n demo -l=kubestash.com/repo-name=dep-repo
NAME                                                            REPOSITORY   SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       VERIFICATION-STATUS   AGE
dep-repo-deployment-kubestash-demo-frequent-backup-1704460503   dep-repo     frequent-backup   2024-01-05T13:15:07Z   Delete            Succeeded                         72s
```

> Note: KubeStash creates a `Snapshot` with the following labels:
> - `kubestash.com/app-ref-kind: <workload-kind>`
> - `kubestash.com/app-ref-name: <workload-name>`
> - `kubestash.com/app-ref-namespace: <workload-namespace>`
> - `kubestash.com/repo-name: <repository-name>`
>
> These labels can be used to watch only the `Snapshot`s related to our desired Workload or `Repository`.

If we check the YAML of the `Snapshot`, we can find the information about the backed up components of the Deployment.

```bash
$ kubectl get snapshots -n demo dep-repo-deployment-kubestash-demo-frequent-backup-1704460503 -oyaml
```

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  labels:
    kubestash.com/app-ref-kind: Deployment
    kubestash.com/app-ref-name: kubestash-demo
    kubestash.com/app-ref-namespace: demo
    kubestash.com/repo-name: dep-repo
  name: dep-repo-deployment-kubestash-demo-frequent-backup-1704460503
  namespace: demo
spec:
  ...
status:
  components:
    dump:
      driver: Restic
      duration: 12.241343157s
      integrity: true
      path: repository/v1/frequent-backup/deployment
      phase: Succeeded
      resticStats:
      - hostPath: /source/data-1
        id: ac086307614333b6af56e0b70c23fc1d4990552d3b414e947e2f86c23e74e9af
        size: 0 B
        uploaded: 994 B
      - hostPath: /source/data-2
        id: 61060b3e893f53400fd4f36e9e692267d883d1220184eb6a6fcaf51cb8f7d41d
        size: 0 B
        uploaded: 994 B
      size: 1.386 KiB
  ...
```

> For Deployment, KubeStash takes backup only one pod of the Deployment. So only one component has been backed up. For logical backup, KubeStash uses `dump` as the component name for Deployment.

Now, if we navigate to the GCS bucket, we will see the backed up data stored in the `demo/demo/sample-sts/repository/v1/frequent-backup/dump` directory. KubeStash also keeps the backup for `Snapshot` YAMLs, which can be found in the `demo/demo/sample-sts/snapshots` directory.

## Backup DaemonSet

Now, we are going to use the same blueprint to backup a DaemonSet. We are going to mount volume in `/source/data` directory. Then, we are going to backup this directory using automatic backup.

**Create DaemonSet:**

Below is the YAML of the DaemonSet that we are going to create,

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: ks-demo
  name: ks-demo
  namespace: demo
  annotations:
    blueprint.kubestash.com/name: workload-backup-blueprint
    blueprint.kubestash.com/namespace: demo
    variables.kubestash.com/targetName: ks-demo
    variables.kubestash.com/namespace: demo
    variables.kubestash.com/repoName: daemon-repo
    variables.kubestash.com/paths: /source/data
spec:
  selector:
    matchLabels:
      app: ks-demo
  template:
    metadata:
      labels:
        app: ks-demo
      name: busybox
    spec:
      containers:
        - args: ["echo sample_data > /source/data/data.txt && sleep 3000"]
          command: ["/bin/sh", "-c"]
          image: busybox
          imagePullPolicy: IfNotPresent
          name: busybox
          volumeMounts:
            - mountPath: /source/data
              name: source-data
      restartPolicy: Always
      volumes:
        - name: source-data
          hostPath:
            path: /kubestash/source/data
```

Notice the `metadata.annotations` field. We have specified automatic backup specific annotations to backup our desired file path.

Let's create the DaemonSet we have shown above,

```bash
$ kubectl apply -f https://github.com/kubestash/docs/raw/{{< param "info.version" >}}/docs/guides/auto-backup/workload/examples/daemonset.yaml
daemonset.apps/ks-demo created
```

**Verify BackupConfiguration:**
If everything goes well, KubeStash should create a `BackupConfiguration` for our DaemonSet and the phase of that `BackupConfiguration` should be `Ready`. Verify the `BackupConfiguration` CR by the following command,

```bash
$  kubectl get backupconfiguration -n demo
NAME                        PHASE   PAUSED   AGE
daemonset-ks-demo           Ready            4m24s
deployment-kubestash-demo   Ready            3h
statefulset-sample-sts      Ready            4h
```

Here, `daemonset-ks-demo` has been created for the DaemonSet `ks-demo`. You can check the YAML of this `BackupConfiguration` to see that the target field is pointing to this DaemonSet.

```bash
$ kubectl get backupconfiguration -n demo daemonset-ks-demo -o yaml
```

```yaml
apiVersion: core.kubestash.com/v1alpha1
kind: BackupConfiguration
metadata:
  name: daemonset-ks-demo
  namespace: demo
spec:
  ...
  sessions:
    - addon:
        name: workload-addon
        tasks:
          - name: logical-backup
            params:
              paths: /source/data
      failurePolicy: Fail
      name: frequent-backup
      repositories:
        - backend: gcs-backend
          directory: demo/ks-demo
          encryptionSecret:
            name: encrypt-secret
            namespace: demo
          name: daemon-repo
      retryConfig:
        delay: 1m0s
        maxRetry: 2
      ...
  target:
    apiGroup: apps
    kind: DaemonSet
    name: ks-demo
    namespace: demo
status:
  ...
```

**Wait for BackupSession:**

Now, wait for the next backup schedule. Watch the `BackupSession` of the BackupConfiguration `daemonset-ks-demo` using the following command,

```bash
$ watch kubectl get backupsession -n demo -l=kubestash.com/invoker-name=deployment-kubestash-demo
Every 1.0s: kubectl get backupsession -n demo -l=kubestash.com/invoker-na...  workstation: AppsCode-PC-03: Fri Jan  5 19:30:24 2024

NAME                                                   INVOKER-TYPE          INVOKER-NAME                PHASE       DURATION   AGE
daemonset-ks-demo-frequent-backup-1704705120           BackupConfiguration   daemonset-ks-demo           Succeeded              2m21s
```

**Verify Backup:**

Once the backup session is completed, verify that the `Repository` has been updated to reflect the backup using the following command,

```bash
$ kubectl get repository -n demo daemon-repo
NAME          INTEGRITY   SNAPSHOT-COUNT   SIZE    PHASE   LAST-SUCCESSFUL-BACKUP   AGE
daemon-repo   true        1                804 B   Ready   55m                      7m
```

At this moment we have one `Snapshot`. Run the following command to check the respective `Snapshot` which represents the state of a backup run for an application.

```bash
$ kubectl get snapshots -n demo -l=kubestash.com/repo-name=daemon-repo
NAME                                                       REPOSITORY    SESSION           SNAPSHOT-TIME          DELETION-POLICY   PHASE       VERIFICATION-STATUS   AGE
daemon-repo-daemonset-ks-demo-frequent-backup-1704705120   daemon-repo   frequent-backup   2024-01-08T09:12:08Z   Delete            Succeeded                         7m
```

> Note: KubeStash creates a `Snapshot` with the following labels:
> - `kubestash.com/app-ref-kind: <workload-kind>`
> - `kubestash.com/app-ref-name: <workload-name>`
> - `kubestash.com/app-ref-namespace: <workload-namespace>`
> - `kubestash.com/repo-name: <repository-name>`
>
> These labels can be used to watch only the `Snapshot`s related to our desired Workload or `Repository`.

If we check the YAML of the `Snapshot`, we can find the information about the backed up components of the DaemonSet.

```bash
$ kubectl get snapshots -n demo daemon-repo-daemonset-ks-demo-frequent-backup-1704705120 -oyaml
```

```yaml
apiVersion: storage.kubestash.com/v1alpha1
kind: Snapshot
metadata:
  labels:
    kubestash.com/app-ref-kind: DaemonSet
    kubestash.com/app-ref-name: ks-demo
    kubestash.com/app-ref-namespace: demo
    kubestash.com/repo-name: daemon-repo
  name: daemon-repo-daemonset-ks-demo-frequent-backup-1704705120
  namespace: demo
spec:
  ...
status:
  components:
    dump-kind-control-plane:
      driver: Restic
      duration: 6.948057521s
      integrity: true
      path: repository/v1/frequent-backup/kind-control-plane
      phase: Succeeded
      resticStats:
      - hostPath: /source/data
        id: 9a682fd821d7ee19770d5f8db34b5fff57252088d7351d883979675e8b6b5340
        size: 12 B
        uploaded: 1.049 KiB
      size: 804 B
  ...
```

> For DaemonSet, KubeStash takes backup from every daemon pod running on different nodes. Since we are using a single node cluster (Kind), only one component has been backed up. For logical backup, KubeStash uses `dump-<node-name>` as the component name, with `<node-name>` representing the name of the node where DaemonSet's pod is deployed.

Now, if we navigate to the GCS bucket, we will see the backed up data stored in the `demo/demo/sample-sts/repository/v1/frequent-backup/dump-<node-name>` directory. KubeStash also keeps the backup for `Snapshot` YAMLs, which can be found in the `demo/demo/sample-sts/snapshots` directory.

## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete -n demo statefulset/sample-sts
kubectl delete -n demo deployment/kubestash-demo
kubectl delete -n demo daemonset/ks-demo

kubectl delete -n demo backupstorage --all
kubectl delete -n demo secret/gcs-secret
kubectl delete -n demo backupblueprint/workload-backup-blueprint
```

If you would like to uninstall KubeStash operator, please follow the steps [here](/docs/setup/README.md).
