---
title: Hooks Examples | KubeStash
menu:
  docs_{{ .version }}:
    identifier: configuring-hooks
    name: Configuring Hooks
    parent: hooks
    weight: 40
product_name: kubestash
menu_name: docs_{{ .version }}
section_menu_id: guides
---

# Configuring Different Types of Hooks

In this guide, we are going to discuss how to configure different types of hooks in `HookTemplate`. Here, we will give some examples of different configurations.

## Based on Action

Based on action of a hook, we can configure them for the following use cases.

### HTTPGet

`httpGet` hook allows sending an HTTP GET request to an HTTP server before and after the backup/restore process. The following configurable fields are available for `httpGet` hook.

| Field         | Field Type | Data Type             | Usage                                                                                                                                                                                                                                                                                    |
|---------------|------------|-----------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `host`        | `Optional` | `String`              | Specify the name of the host where the GET request will be sent. If you don't specify this field, KubeStash will use the hook executor Pod IP as host.                                                                                                                                   |
| `port`        | `Required` | `Integer` or `String` | Specify the port where the HTTP server is listening. You can specify a numeric port or the name of the port in case of the HTTP server is running in the same pod where the hook will be executed. If you specify the port name, you must specify the `containerName` field of the hook. |
| `path`        | `Required` | `String`              | Specify the path to access on the HTTP server.                                                                                                                                                                                                                                           |
| `scheme`      | `Required` | `String`              | Specify the scheme to use to connect with the host.                                                                                                                                                                                                                                      |
| `httpHeaders` | `Optional` | Array of Objects      | Specify a list of custom headers to set in the request.                                                                                                                                                                                                                                  |

**Examples:**

- Hook with `host` and `port` are specified:

```yaml
action:
  httpGet:
    host: my-host.demo.svc
    port: 8080
    path: "/"
    scheme: HTTP
```

- Hook to extract `host` and `port` from the hook executor pod:

```yaml
action:
  httpGet:
    port: my-port
    path: "/"
    scheme: HTTP
  containerName: my-container
```

- Hook for sending custom header in the HTTP request:

```yaml
action:
  httpGet:
    host: my-host.demo.svc
    port: 8080
    path: "/"
    scheme: HTTP
    httpHeaders:
    - name: "User-Agent"
      value: "kubestash/0.5.x"
```

### HTTPPost

`httpPost` hook allows sending an HTTP POST request to an HTTP server before and after the backup/restore process. The following configurable fields are available for `httpPost` hook.

| Field         | Field Type | Data Type             | Usage                                                                                                                                                                                                                                                                             |
|---------------|------------|-----------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `host`        | `Optional` | `String`              | Specify the name of the host where the POST request will be sent. If you don't specify this field, KubeStash will use the hook executor Pod IP as host.                                                                                                                           |
| `port`        | `Required` | `Integer` or `String` | Specify the port where the HTTP server is listening. You can specify a numeric port or the name of the port if your HTTP server is running in the same pod where the hook will be executed. If you specify the port name, you must specify the `containerName` field of the hook. |
| `path`        | `Required` | `String`              | Specify the path to access on the HTTP server.                                                                                                                                                                                                                                    |
| `scheme`      | `Required` | `String`              | Specify the scheme to use to connect with the host.                                                                                                                                                                                                                               |
| `httpHeaders` | `Optional` | Array of Objects      | Specify a list of custom headers to set in the request.                                                                                                                                                                                                                           |
| `body`        | `Optional` | `String`              | Specify the data to send in the request body.                                                                                                                                                                                                                                     |
| `form`        | `Optional` | Array of Objects      | Specify the data to send as URL encoded form with the request.                                                                                                                                                                                                                    |

**Examples:**

- Send JSON data in the request body:

```yaml
action:
  httpPost:
    host: my-service.mynamespace.svc
    path: /demo
    port: 8080
    scheme: HTTP
    httpHeaders:
    - name: Content-Type
      value: application/json
    body: '{
            "name": "john doe",
            "age": "28"
           }'
```

- Send URL encoded form in the request:

```yaml
action:
  httpPost:
    host: my-service.mynamespace.svc
    path: /demo
    port: 8080
    scheme: HTTP
    form:
    - key: "name"
      values: ["john doe"]
    - key: "age"
      values: ["28"]
```

### TCPSocket

`tcpSocket` hook allows running a check against a TCP port to verify whether it is open or not. The following configurable fields are available for `tcpSocket` hook.

| Field  | Field Type | Data Type             | Usage                                                                                                                                                                                                                                                |
|--------|------------|-----------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `host` | `Optional` | `String`              | Specify the name of the host where the server is running. If you don't specify this field, KubeStash will use the hook executor Pod IP as host.                                                                                                      |
| `port` | `Required` | `Integer` or `String` | Specify the port to check. You can specify a numeric port or the name of the port when your server is running in the same pod where the hook will be executed. If you specify the port name, you must specify the `containerName` field of the hook. |

**Examples:**

- Hook with `host` and `port` field specified.

```yaml
action:
  tcpSocket:
    host: my-host.demo.svc
    port: 8080
```

- Hook to extract `host` and `port` from the hook executor pod.

```yaml
action:
  tcpSocket:
    port: my-port
  containerName: my-container
```

### Exec

`exec` hook allows running commands inside a container before or after the backup/restore process. The following configurable fields are available for `exec` hook.

| Field     | Field Type | Data Type        | Usage                                  |
|-----------|------------|------------------|----------------------------------------|
| `command` | `Required` | Array of Strings | Specify a list of commands to execute. |

> If you don't specify `containerName` for `exec` hook, the command will be executed into the 0th container of the respective pod.

**Examples:**

- Hook to remove old corrupted data:

```yaml
action:
  exec:
    command: ["/bin/sh","-c","rm -rf /corrupted/data/directory"]
  containerName: my-container
```

- Hook to make a MySQL database read-only:

```yaml
action:
  exec:
    command:
      - /bin/sh
      - -c
      - mysql -u root --password=$MYSQL_ROOT_PASSWORD -e "SET GLOBAL super_read_only = ON;"
  containerName: mysql
```

- Hook to make a MySQL database writable:

```yaml
action:
  exec:
    command:
      - /bin/sh
      - -c
      - mysql -u root --password=$MYSQL_ROOT_PASSWORD -e "SET GLOBAL super_read_only = OFF;"
  containerName: mysql
```

- Hook to remove corrupted MySQL database:

```yaml
action:
  exec:
    command:
      - /bin/sh
      - -c
      - mysql -u root --password=$MYSQL_ROOT_PASSWORD -e "DROP DATABASE companyRecord;"
  containerName: mysql
```

- Hook to apply some migration after restoring a MySQL database:

```yaml
action:
  exec:
    command:
      - /bin/sh
      - -c
      - mysql -u root --password=$MYSQL_ROOT_PASSWORD -e "RENAME TABLE companyRecord.employee TO companyRecord.salaryRecord;"
  containerName: mysql
```

## Based on Executor

Based on executor of a hook, we can configure them in the following ways.

### Pod Executor

Provide list of key value pair that will be used as label selector to select the desired pods. You can use comma to separate multiple labels (i.e. `"app=my-app,env=prod"`).

**Examples:**

- Execute hook on all the selected pods (only filtered by `selector`):

```yaml
executor:
  type: Pod
  pod:
    selector: env=dev, app=web
    strategy: ExecuteOnAll
```

- Execute hook on only one of the selected pods (only filtered by `selector`). This is default behavior:

```yaml
executor:
  type: Pod
  pod:
    selector: env=dev, app=web
```

- Execute hook on all the selected pods (filtered by `selector` and `owner`):

```yaml
executor:
  type: Pod
  pod:
    selector: env=dev, app=web
    owner:
      apiVersion: apps/v1
      kind: Deployment
      name: kubestash-op
    strategy: ExecuteOnAll
```

### Operator Executor

KubeStash operator itself will execute the hook. 

**Example:**

```yaml
executor:
  type: Operator
```

### Function Executor

Hook is executed by a Job. We need to create a [Function](/docs/concepts/crds/function/index.md) define the `Function` and its parameters in [FunctionHookExecutorSpec](/docs/concepts/crds/hooktemplate/index.md#specexecutor) that will be used to create hook executor job.

**Example:**

```yaml
executor:
  type: Function
  function:
    name: hook-function
```

Here we can provide a list of environment variables that will be passed to the executor container. We can also provide volumes and the volumes mounts for the executor container.

If the above hooks do not cover your use cases, or you have any questions regarding the hooks, please feel free to open an issue [here](https://github.com/kubestash/).
