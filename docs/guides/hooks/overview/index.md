---
title: Hooks Overview | KubeStash
menu:
  docs_{{ .version }}:
    identifier: hooks-overview
    name: Overview
    parent: hooks
    weight: 10
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# KubeStash Backup and Restore Hooks

KubeStash supports executing custom commands before and after backup or restore process. This is called `hook` in KubeStash. This guide will give you an overview of what kind of hooks you can execute, how the hooks get executed, and how the hooks behave in different scenarios.

## Types of Hooks

We can categorize KubeStash backup and restore hooks based on the action they perform and based on their execution order.

### Based on Action

Based on the action of a hook, we can categorize them into four different categories. These are the followings:

- **HTTPGet:** Executes an HTTP GET request before/after the backup/restore process. The hook is considered successful if the return code is between `200` and `400`.

- **HTTPPost:** Executes an HTTP POST request before/after the backup/restore process. Like `HTTPGet`, the hook is considered successful if the return code is between `200` and `400`.

- **TCPSocket:** Performs a TCP check against the provided URL on a specific port before/after the backup/restore process. The hook is considered successful if the targeted port is open.

- **Exec:** Executes commands inside a targeted container before/after the backup/restore process. The hook is considered successful if the command executes with exit code 0.

### Based on Backup & Restore

Based on the backup & restore, we can categorize the hooks into two different categories. These are the followings:

- **Backup Hook:** `Backup Hook` specifies the hook that will be executed before and/or after backup. Based on the execution order, we can categorize each backup hook into two different phases. These are the followings:
  
  - **PreBackup Hook:** `PreBackup Hook` specifies the hook that will be executed before backup.
  
  - **PostBackup Hook:** `PostBackup Hook`  specifies the hook that will be executed after backup

- **Restore Hook:** `Restore Hook` specifies the hook that will be executed before and/or after restore. Based on the execution order, we can categorize each restore hook into two different phases. These are the followings:

  - **PreRestore Hook:** `PreRestore Hook` specifies the hook that will be executed before restore.

  - **PostRestore Hook:** `PostRestore Hook`  specifies the hook that will be executed after restore.


### Based on Hook Executor

Based on the hook executor, we can categorize the hooks into three different categories. These are the followings:

- **Pod Executor Hook:** In pod executor hook, the hook is executed by the selected Pod(s). We can specify the criteria in [PodHookExecutorSpec](/docs/concepts/crds/hooktemplate/index.md#specexecutor) that will be used to select the pod which will execute the hook. 

- **Operator Executor Hook:** In operator executor hook, the hook is executed by the KubeStash operator. 

- **Function Executor Hook:** In function executor hook, the hook is executed by a Job. We can create a [Function](/docs/concepts/crds/function/index.md) and define the `Function` and its parameters in [FunctionHookExecutorSpec](/docs/concepts/crds/hooktemplate/index.md#specexecutor) that will be used to create hook executor job. 

## Hook's Behaviors

Now, we are going to discuss what will happen when a hook fails or backup/restore process fails.

### `preBackup` or `preRestore` hook

If a `preBackup` or `preRestore` hook fails to execute, the rest of the backup/restore process will be skipped and the respective `BackupSession`/`RestoreSession` will be marked as `Failed`.

### `postBackup` or `postRestore` hook

If the backup or restore process fails then the respective `postBackup` or `postRestore` hook will be executed according to the policy specified in the `executionPolicy` field of the respective hook in `BackupConfiguration` / `RestoreSession`. The current acceptable values and behaviors are:

- `Always`: The hook will be executed after the backup/restore process no matter the backup/restore has failed or succeeded. This is the default behavior.
- `OnSuccess`: The hook will be executed after the backup/restore process only if the backup/restore has succeeded.
- `OnFailure`: The hook will be executed after the backup/restore process only if the backup/restore has failed.

If the `postBackup` or `postRestore` hook fails, the respective BackupSession or RestoreSession will be marked as `Failed`.
