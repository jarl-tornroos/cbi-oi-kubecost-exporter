# Kubecost exporter for Flexera CCO

Kubecost Flexera Exporter is a utility to collect cost allocation data. It is a command line tool that automates the transfer of Kubernetes cluster cost allocation data to Cloud Cost Optimization. This tool generates a CSV file for each day of the current (and optionally previous) month in a format compatible with the Flexera One platform and then uploads files into Cloud Cost Optimization via CBI connect.
Kubecost Flexera Exporter utilizes Kubecost Allocation API to request cost allocation data. The majority of Kubecost Allocation API parameters are exposed as exporter settings, matching Kubecost API parameters are listed in the exporter setting descriptions.

## Installation

The application can be installed from golang sources, from a docker image or via the helm package manager.

### Go sources

This app requires Go version 1.16 or higher. To install, run:

```bash
go install github.com/flexera-public/cbi-oi-kubecost-exporter
```

#### Settings
The app is configured using environment variables defined in a .env file. The following configuration options are available:
| Environment Variable | Kubecost API | Description |
|---------|---------|-------------|
| FILE_PATH | - | The path where the generated CSV files are stored. Default is "/var/kubecost"
| FILE_ROTATION | - | Indicates whether to delete files generated for previous months. Default is true. Note: current and previous months data is kept.
| BILL_CONNECT_ID | - | The ID of the bill connect to which to upload the data. Default value is "cbi-oi-kubecost-1". To learn more about Bill Connect, and how to obtain your BILL_CONNECT_ID, please refer to [Creating Kubecost CBI Bill Connect](https://docs.flexera.com/flexera/EN/Optima/CreateKubecostBillConnect.htm) in the Flexera documentation.
| ORG_ID | - | The ID of your Flexera One organization, please refer to [Organization ID Unique Identifier](https://docs.flexera.com/flexera/EN/FlexeraAPI/APIKeyConcepts.htm#gettingstarted_2697534192_1120261) in the Flexera documentation.
| REFRESH_TOKEN | - | The refresh token used to obtain an access token for the Flexera One API. Please refer to [Generating a Refresh Token](https://docs.flexera.com/flexera/EN/FlexeraAPI/GenerateRefreshToken.htm) in the Flexera documentation.
| SHARD | - | The zone of your Flexera One account. Valid values are NAM, EU or AU.
| INCLUDE_PREVIOUS_MONTH | - | Indicates whether to collect and export previous month. Default is false.
| REQUEST_TIMEOUT | - | Indicates the timeout per each request in minutes.
| KUBECOST_HOST | - | The hostname of the Kubecost instance. Default is "kubecost-cost-analyzer.kubecost.svc.cluster.local:9090".
| KUBECOST_API_PATH | - | The base path for the Kubecost API endpoint. Default is "/model/"
| AGGREGATION | aggregate | The level of granularity to use when aggregating the cost data. Valid values are namespace, controller, or pod. Default is pod. Note: Exporter collects namespace labels regardless of set aggregation level and includes them into entity labels.
| IDLE | idle | Indicates whether to include cost of idle resources. Valid values are true and false. Default is true.
| IDLE_BY_NODE | idleByNode | Indicates whether idle allocations are created on a per node basis. Valid values are true and false. Default is false.
| SHARE_IDLE | shareIdle | Indicates whether allocate idle cost proportionally across non-idle resources. Default is false.
| SHARE_NAMESPACES | shareNamespaces | Comma-separated list of namespaces to share costs with the remaining non-idle, unshared allocations. Default = kube-system,cadvisor
| SHARE_TENANCY_COSTS | shareTenancyCosts | Indicates whether to share the cost of cluster overhead assets across tenants of those resources. Default is true.
| MULTIPLIER | - | Optional multiplier for costs. Default is 1.


#### Execution
To use this app, run:

```bash
flexera-kubecost-exporter
```

### Kubecost exporter helm chart for Kubernetes

There are two different ways to transfer custom Helm configuration values to the kubecost-exporter:

#### 1. Pass exact parameters via --set command-line flags:

```
helm install kubecost-exporter cbi-oi-kubecost-exporter \
    --repo https://flexera-public.github.io/cbi-oi-kubecost-exporter/ \
    --namespace kubecost-exporter --create-namespace \
    --set flexera.refreshToken="Ek-aGVsbUBrdWJlY29zdC5jb20..." \
    --set flexera.orgId="1105" \
    --set flexera.billConnectId="cbi-oi-kubecost-..." \
    ...
```

#### 2. Pass exact parameters via custom values.yaml file:

2.1 Create a **values.yaml** file and add the necessary settings to it as below:

```yml
flexera:
    refreshToken: "Ek-aGVsbUBrdWJlY29zdC5jb20..."
    orgId: "1105"
    billConnectId: "cbi-oi-kubecost-..."

kubecost:
    aggregation: "pod"
```

2.2 Apply this file when installing kubecost-exporter:

```
helm install kubecost-exporter cbi-oi-kubecost-exporter \
    --repo https://flexera-public.github.io/cbi-oi-kubecost-exporter/ \
    --namespace kubecost-exporter --create-namespace \
    --values values.yaml
```

### Verifying configuration

After successfully installing the helm chart, you can trigger the CronJob manually to ensure that everything is working as expected:

1. Check the schedule of the CronJob:

```
kubectl get cronjobs -n <your-namespace>
```

The `SCHEDULE` column should reflect the schedule you have set (default: "0 \*/6 \* \* \*"). The `NAME` column shows the name of your CronJob.

2. Manually create a job from the CronJob:

```
kubectl create job --from=cronjob/<your-cronjob-name> manual-001 -n <your-namespace>
```

Replace `<your-cronjob-name>` with the name of your CronJob you obtained from the previous command.

3. Monitor the logs of the job:

First, get the name of the pod associated with the job you have just created:

```
kubectl get pods -n <your-namespace>
```

Look for the pod that starts with the name "manual-001".

Then, fetch the logs:

```
kubectl logs <your-pod-name> -n <your-namespace>
```

Replace `<your-pod-name>` with the name of your pod.

You should see 200/201s in the logs, which indicates that the exporter is working as expected. This means that the CronJob will also run successfully according to its schedule.

### Helm configuration Values

| Key | Type | Kubecost API Parameter | Default | Description |
|-----|------|---------|----------|-------------|
| cronSchedule | string | - | `"0 */6 * * *"` | Setting up a cronJob scheduler to run an export task at the desired time. |
| env | object | - | `{}` | Pod environment variables |
| filePath | string | - | `"/var/kubecost"` | File path to mount persistent volume. |
| fileRotation | bool | - | `true` | Indicates whether to delete files generated for previous months. Default is true. Note: current and previous months data is kept. |
| flexera.billConnectId | string | - | `"cbi-oi-kubecost-1"` | The ID of the bill connect to which to upload the data. To learn more about Bill Connect, and how to obtain your BILL_CONNECT_ID, please refer to [Creating Kubecost CBI Bill Connect](https://docs.flexera.com/flexera/EN/Optima/CreateKubecostBillConnect.htm) in the Flexera documentation. |
| flexera.orgId | string | - | `""` | The ID of your Flexera One organization, please refer to [Organization ID Unique Identifier](https://docs.flexera.com/flexera/EN/FlexeraAPI/APIKeyConcepts.htm#gettingstarted_2697534192_1120261) in the Flexera documentation. |
| flexera.refreshToken | string | - | `""` | The refresh token used to obtain an access token for the Flexera One API. Please refer to [Generating a Refresh Token](https://docs.flexera.com/flexera/EN/FlexeraAPI/GenerateRefreshToken.htm) in the Flexera documentation. You can provide the refresh token in two ways: 1. Directly as a string:    refreshToken: "your_token_here" 2. Reference it from a Kubernetes secret:    refreshToken:      valueFrom:        secretKeyRef:          name: flexera-secrets  # Name of the Kubernetes secret          key: refresh_token     # Key in the secret containing the refresh token |
| flexera.shard | string | - | `"NAM"` | The zone of your Flexera One account. Valid values are NAM, EU or AU. |
| image.pullPolicy | string | - | `"Always"` |  |
| image.repository | string | - | `"public.ecr.aws/flexera/cbi-oi-kubecost-exporter"` |  |
| image.tag | string | - | `"1.9"` |  |
| imagePullSecrets | list | - | `[]` |  |
| includePreviousMonth | bool | - | `false` | Indicates whether to collect and export previous month. |
| kubecost.aggregation | string | aggregate | `"pod"` | The level of granularity to use when aggregating the cost data. Valid values are namespace, controller, or pod. |
| kubecost.apiPath | string | - | `"/model/"` | The base path for the Kubecost API endpoint. |
| kubecost.host | string | - | `"kubecost-cost-analyzer.kubecost.svc.cluster.local:9090"` | Default kubecost-cost-analyzer service host on the current cluster. For current cluster is serviceName.namespaceName.svc.cluster.local |
| kubecost.idle | bool | idle | `true` | Indicates whether to include cost of idle resources. |
| kubecost.idleByNode | bool | idleByNode | `false` | Indicates whether idle allocations are created on a per node basis. |
| kubecost.multiplier | float | - | `1` | Optional multiplier for costs. |
| kubecost.shareIdle | bool | shareIdle | `false` | Indicates whether allocate idle cost proportionally across non-idle resources. |
| kubecost.shareNamespaces | string | shareNamespaces | `"kube-system,cadvisor"` | Comma-separated list of namespaces to share costs with the remaining non-idle, unshared allocations. |
| kubecost.shareTenancyCosts | bool | shareTenancyCosts | `true` | Indicates whether to share the cost of cluster overhead assets across tenants of those resources. |
| persistentVolume.enabled | bool | - | `true` | Enable Persistent Volume. Recommended setting is true to prevent loss of historical data. |
| persistentVolume.size | string | - | `"1Gi"` | Persistent Volume size. |
| requestTimeout | int | - | `5` | Indicates the timeout per each request in minutes. |

## License

This tool is licensed under the MIT license. See the LICENSE file for more details.
