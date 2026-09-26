# Debug Operations in Kubernetes

## Overview

After you deploy a workload, you might notice that a Pod or container is not behaving as expected. A Pod or container might restart unexpectedly or fail to reach a healthy state. This can happen shortly after deployment or later while the workload is operating normally. When this happens, you need to determine the Pod's current state and gather information that can help you identify the problem. You can use the Kubernetes command-line tool `kubectl` to perform debugging operations. `kubectl` communicates with the Kubernetes API server to manage and inspect resources such as Pods and containers.

This page covers four operations that are useful for investigating a workload: checking a Pod's status, retrieving container logs, executing commands inside an existing container, and creating a debugging environment with `kubectl debug`. You don't need to perform these operations in a fixed sequence.

## `kubectl get pods`

`kubectl get pods` lists the Pods in a namespace and shows their current status. Each row shows the Pod's `NAME`, `READY` count, `STATUS`, `RESTARTS`, and `AGE`.

Use this command when you need a quick view of the current state of the Pods associated with a workload. Look for a `STATUS` value that differs from the state you expect for that workload. Not every status other than `Running` is a problem. For example, `Succeeded` means the Pod's containers completed successfully and won't restart. A high or increasing `RESTARTS` value is also worth investigating.

### Syntax and Example

To list Pods in a specific namespace, specify the namespace:

```bash
kubectl get pods --namespace <namespace>
```

The following example specifies the `production` namespace:

```bash
kubectl get pods --namespace production
```

### Example Output

```text
NAME                         READY   STATUS    RESTARTS   AGE
web-app-7d9c8f6b5d-x2k9m     1/1     Running   0          2h
api-6f84c7d8b9-p4r7t         1/1     Running   3          2h
```

The example output shows both Pods currently active and ready. The `api` Pod has restarted three times. A restart count does not necessarily indicate a current failure, but an increasing count can indicate a recurring container problem.

### Important Considerations

- By default, `kubectl get pods` shows only the Pods in your current namespace. Use `--all-namespaces` (or `-A`) to list Pods across every namespace in the cluster.
- Add `-o wide` to see additional columns, such as the Pod's IP address and the node that hosts the Pod.
- The `STATUS` column and the Pod phase represent different information. Kubernetes defines a small set of phases: `Pending`, `Running`, `Succeeded`, `Failed`, and `Unknown`. The `STATUS` column can display other values, such as `CrashLoopBackOff`, that provide additional information about the Pod's condition.

## `kubectl logs`

`kubectl logs` prints the logs written by a container in a Pod. Logs can show you error messages, startup output, or other information a container has reported.

Use this command when you need to see what a container has actually reported, such as when a Pod shows an unexpected `STATUS` or a high or increasing `RESTARTS` value.

### Syntax and Example

To retrieve logs for a Pod in a specific namespace, specify the Pod name and namespace:

```bash
kubectl logs <pod-name> --namespace <namespace>
```

The following example retrieves logs for the `api` Pod in the `production` namespace:

```bash
kubectl logs api-6f84c7d8b9-p4r7t --namespace production
```

### Example Output

```text
2026-09-24T10:15:32Z INFO  Starting api server on port 8080
2026-09-24T10:15:33Z ERROR Failed to connect to database: connection refused
2026-09-24T10:15:33Z INFO  Retrying in 5s...
2026-09-24T10:15:38Z ERROR Failed to connect to database: connection refused
```

The output shows that the application reports repeated failures to connect to its database. The errors provide information that may help investigate the earlier restart count.

### Important Considerations

- If a Pod has more than one container, specify which container's logs to retrieve with `-c <container-name>`.
- Add `--previous` (or `-p`) to retrieve logs from the previous instance of a container, if one exists. This is useful when you need to investigate why a container restarted.
- Add `-f` (or `--follow`) to stream new log output as it is written, rather than viewing a single snapshot.

## `kubectl exec`

`kubectl exec` executes a command inside an existing container. You can execute a single command and see its output, or open an interactive shell inside the container.

Use this command when you need to inspect a container's environment directly, such as checking configuration values, environment variables, or connectivity from inside the container.

### Syntax and Example

To execute a command inside a container, specify the Pod, namespace, and the command to execute:

```bash
kubectl exec <pod-name> --namespace <namespace> -- <command>
```

The following example checks the environment variables inside the `api` Pod's container:

```bash
kubectl exec api-6f84c7d8b9-p4r7t --namespace production -- env
```

### Example Output

```text
HOSTNAME=api-6f84c7d8b9-p4r7t
DB_HOST=db-service
DB_PORT=5432
```

The output shows that the `api` container is configured to connect to `db-service` on port `5432`. These values provide configuration details to help you investigate the database connection failures shown in the container logs.

### Important Considerations

- Add `-it` and specify an available shell, such as `/bin/sh`, to open an interactive shell inside the container.
- If a Pod has more than one container, specify which one to execute the command in with `-c <container-name>`.
- `kubectl exec` executes the command in the existing container. If the container image doesn't include a shell or the tools you need, you won't be able to investigate further this way.

## `kubectl debug`

`kubectl debug` provides additional debugging capabilities for Pods and nodes. Depending on how you use it, the command can add an ephemeral container to an existing Pod, create a modified copy of a Pod, or create a debugging environment for a node.

Use this command when the existing container does not include the shell or diagnostic tools you need, or when you need a separate environment for deeper investigation.

### Syntax and Example

To add an ephemeral debugging container to an existing Pod, specify a container image and the container you want to target:

```bash
kubectl debug <pod-name> --namespace <namespace> --image=<debug-image> --target=<container-name>
```

The following example adds a `busybox` debugging container to the `api` Pod and targets the `api` container's process namespace:

```bash
kubectl debug api-6f84c7d8b9-p4r7t --namespace production --image=busybox --target=api -it -- /bin/sh
```

### Example Output

```text
Defaulting debug container name to debugger-8xjvq.
If you don't see a command prompt, try pressing enter.
/ #
```

This adds a new `busybox` debugging container to the `api` Pod and opens a shell inside it. From here, you can use tools such as `nslookup` to check whether `db-service` resolves, without modifying the `api` container itself.

### Important Considerations

- `kubectl debug` supports more than one mode. Besides adding an ephemeral container to an existing Pod, you can create a debugging copy of a Pod with `--copy-to`, or debug a node directly with `kubectl debug node/<node-name>`.
- The debugging container uses the image you specify with `--image`. Choose an image that includes the tools you actually need, such as networking utilities.
- An ephemeral container added to an existing Pod this way can't be removed once created. It persists until the Pod itself is deleted, so consider using `--copy-to` if you'd rather not modify the original Pod at all.
- Specify `--target=<container-name>` if you need visibility into the processes in the target container. This functionality is available only if the container runtime supports process namespace targeting.

## References

- [kubectl Commands](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-strong-getting-started-strong-)
- [What is Kubernetes](https://kubernetes.io/docs/concepts/overview/)
