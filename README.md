# Bottlerocket Update Operator

The Bottlerocket update operator (or, for short, Brupop) is a [Kubernetes operator](https://Kubernetes.io/docs/concepts/extend-Kubernetes/operator/) that coordinates Bottlerocket updates on hosts in a cluster.
When installed, the Bottlerocket update operator starts a controller deployment on one node, an agent daemon set on every Bottlerocket node, and an Update Operator API Server deployment.
The controller orchestrates updates across your cluster, while the agent is responsible for periodically querying for Bottlerocket updates, draining the node, and performing the update when asked by the controller.
The agent performs all cluster object mutation operations via the API Server, which performs additional authorization using the Kubernetes TokenReview API -- ensuring that any request associated with a node is being made by the agent pod running on that node.
Further, `cert-manager` is required in order for the API server to use a CA certificate to communicate over SSL with the agents.
Updates to Bottlerocket are rolled out in [waves](https://github.com/bottlerocket-os/bottlerocket/tree/develop/sources/updater/waves) to reduce the impact of issues; the nodes in your cluster may not all see updates at the same time.

For a deep dive on installing Brupop, how it works, and its integration with Bottlerocket, [check out this design deep dive document!](./design/1.0.0-release.md)

## Getting Started

### Version Support Policy

As per our policy, Brupop follows the semantic versioning (semver) principles, ensuring that any updates in minor versions do not introduce any breaking or backward-incompatible changes. However, please note that we only provide security patches for the latest minor version. Therefore, it is highly recommended to always keep your Brupop installation up to date with the latest available version.

For example: If `v1.3.0` is the latest Brupop release, then, `v1.3` (latest minor version) will be considered as supported and `v1.3.0` (latest available version) will be the recommended version of Brupop to be installed. When `v1.3.1` is released, then that version will be considered as recommended.

### Installation

1. Brupop uses [cert-manager](https://cert-manager.io) to manage self-signed certificates used by the Bottlerocket update operator. To install cert-manager:

```sh
kubectl apply -f \
  https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```

Or if you're using helm:

```sh
# Add the cert-manager helm chart
helm repo add jetstack https://charts.jetstack.io

# Update your local chart cache with the latest
helm repo update

# Install the cert-manager (including its CRDs)
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.8.2 \
  --set installCRDs=true
```

2. The Bottlerocket update operator can then be installed using helm:

```sh
# Add the bottlerocket-update-operator chart
helm repo add brupop https://bottlerocket-os.github.io/bottlerocket-update-operator

# Update your local chart cache with the latest updates
helm repo update

# Create a namespace
kubectl create namespace brupop-bottlerocket-aws

# Install the brupop CRD
helm install brupop-crd brupop/bottlerocket-shadow

# Install the brupop operator
helm install brupop-operator brupop/bottlerocket-update-operator
```

This will create the custom resource definition, roles, deployments, etc., and use the latest update operator image available in [Amazon ECR Public](https://gallery.ecr.aws/bottlerocket/bottlerocket-update-operator).

Alternatively, you can use the pre-baked manifest, with all the default values, [found at the root of this repository (named `bottlerocket-update-operator.yaml`).](./bottlerocket-update-operator.yaml)
This YAML manifest file is also attached to each release [found in the GitHub releases page.](https://github.com/bottlerocket-os/bottlerocket-update-operator/releases)

Note: The generated manifests points to the latest version of the Update Operator, `v1.1.0`.
Be sure to use the release for the Update Operator release that you plan to deploy.

3. Label nodes with `bottlerocket.aws/updater-interface-version=2.0.0` to indicate they should be automatically updated.
Only bottlerocket nodes with this label will be updated. For more information on labeling, refer to the [Label nodes section of this readme.](https://github.com/bottlerocket-os/bottlerocket-update-operator#label-nodes)

```sh
kubectl label node MY_NODE_NAME bottlerocket.aws/updater-interface-version=2.0.0
```

### Configuration

#### Configure via Helm values yaml file

You can use a values file when installing brupop with helm (via the `--values / -f` flag) to configure how brupop functions:

```yaml
# Default values for bottlerocket-update-operator.

# The namespace to deploy the update operator into
namespace: "brupop-bottlerocket-aws"

# The image to use for brupop
# This defaults to the image built alongside a particular helm chart, but you can override it by uncommenting this line:
# image: "public.ecr.aws/bottlerocket/bottlerocket-update-operator:v1.4.0"

# Placement controls
# See the Kubernetes documentation about placement controls for more details:
# * https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
# * https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector
# * https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity
placement:
  agent:
    # The agent is a daemonset, so the only controls that apply to it are tolerations.
    tolerations: []

  controller:
    tolerations: []
    nodeSelector: {}
    podAffinity: {}
    podAntiAffinity: {}

  apiserver:
    tolerations: []
    nodeSelector: {}
    podAffinity: {}
    # By default, apiserver pods prefer not to be scheduled to the same node.
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          podAffinityTerm:
            labelSelector:
              matchExpressions:
              - key: brupop.bottlerocket.aws/component
                operator: In
                values:
                - apiserver
            topologyKey: kubernetes.io/hostname

# If testing against a private image registry, you can set the pull secret to fetch images.
# This can likely remain as `brupop` so long as you run something like the following:
# kubectl create secret docker-registry brupop \
#  --docker-server 109276217309.dkr.ecr.us-west-2.amazonaws.com \
#  --docker-username=AWS \
#  --docker-password=$(aws --region us-west-2 ecr get-login-password) \
#  --namespace=brupop-bottlerocket-aws
#imagePullSecrets:
#  - name: "brupop"

# External load balancer setting.
# When `exclude_from_lb_wait_time_in_sec` is set to a positive value
# brupop will exclude the node from load balancing
# and will wait for `exclude_from_lb_wait_time_in_sec` seconds before draining nodes.
# Under the hood, this uses the `node.kubernetes.io/exclude-from-external-load-balancers` label
# to exclude those nodes from load balancing.
exclude_from_lb_wait_time_in_sec: "0"

# Concurrent update nodes setting.
# When `max_concurrent_updates` is set to a positive integer value,
# brupop will concurrently update max `max_concurrent_updates` nodes.
# When `max_concurrent_updates` is set to "unlimited",
# brupop will concurrently update all nodes with respecting `PodDisruptionBudgets`
# Note: the "unlimited" option does not work well with `exclude_from_lb_wait_time_in_sec`
# option, which could potential exclude all nodes from load balancer at the same time.
max_concurrent_updates: "1"

# DEPRECATED: use the scheduler settings
# Start and stop times for update window
# Brupop will operate node updates within update time window.
# when you set up time window start and stop time, you should use UTC (24-hour time notation).
update_window_start: "0:0:0"
update_window_stop: "0:0:0"

# Scheduler setting
# Brupop will operate node updates on scheduled maintenance windows by using cron expressions.
# When you set up the scheduler, you should follow cron expression rules.
# ┌───────────── seconds (0 - 59)
# │ ┌───────────── minute (0 - 59)
# │ │ ┌───────────── hour (0 - 23)
# │ │ │ ┌───────────── day of the month (1 - 31)
# │ │ │ │ ┌───────────── month (Jan, Feb, Mar, Apr, Jun, Jul, Aug, Sep, Oct, Nov, Dec)
# │ │ │ │ │ ┌───────────── day of the week (Mon, Tue, Wed, Thu, Fri, Sat, Sun)
# │ │ │ │ │ │ ┌───────────── year (formatted as YYYY)
# │ │ │ │ │ │ │
# │ │ │ │ │ │ │
# * * * * * * *
scheduler_cron_expression: "* * * * * * *"

# API server ports
# The port the API server uses for its own operations. This is accessed by the controller,
# the bottlerocket-shadow daemonset, etc.
apiserver_internal_port: "8443"
# API server internal address where the CRD version conversion webhook is served
apiserver_service_port: "443"

logging:
  # Formatter for the logs emitted by brupop.
  # Options are:
  # * full - Human-readable, single-line logs
  # * compact - A variant of full optimized for shorter line lengths
  # * pretty - "Excessively pretty" logs optimized for human-readable terminal output.
  # * json - Newline-delimited JSON-formatted logs.
  formatter: "pretty"
  # Whether or not to enable ANSI colors on log messages.
  # Makes the output "pretty" in terminals, but may add noise to web-based log utilities.
  ansi_enabled: "true"

  # Controls the filter for tracing/log messages.
  # This can be as simple as a log-level (e.g. "info", "debug", "error"), but also supports more complex directives.
  # See https://docs.rs/tracing-subscriber/0.3.17/tracing_subscriber/filter/struct.EnvFilter.html#directives
  controller:
    tracing_filter: "info"
  agent:
    tracing_filter: "info"
  apiserver:
    tracing_filter: "info"

# Provide pod level labels for the brupop resources.
podLabels: {}
```

#### Configuration via Kubernetes yaml

##### Configure API server ports

If you'd like to configure what ports the API server uses,
adjust the value that is consumed in the container environment:

```yaml
      ...
      containers:
        - command:
            - "./api-server"
          env:
            - name: APISERVER_INTERNAL_PORT
              value: 999
```

You'll then also need to adjust the various "port" entries in the YAML manifest
to correctly reflect what port the API server starts on and expects for its service port:

```yaml
    ...
    webhook:
      clientConfig:
        service:
          name: brupop-apiserver
          namespace: brupop-bottlerocket-aws
          path: /crdconvert
          port: 123
```

The default values are generated from the [default values yaml file](https://github.com/bottlerocket-os/bottlerocket-update-operator/blob/develop/deploy/bottlerocket-update-operator/values.yaml) file:
- `apiserver_internal_port: "8443"` - This is the container port the API server starts on.
  If this environment variable is _not_ found, the Brupop API server will fail to start.
- `apiserver_service_port: "443"` - This is the port Brupop's Kubernetes Service uses to target the internal API Server port.
  It is used by the node agents to access the API server.
  If this environment variable is _not_ found, the Brupop agents will fail to start.

##### Resource Requests & Limits

The `bottlerocket-update-operator.yaml` manifest makes several default recommendations for
Kubernetes resource requests and limits. In general, the update operator and its components are lite-weight
and shouldn't consume more than 10m CPU (which is roughly equivalent to 1/100th of a CPU core)
and 50Mi (which is roughly equivalent to 0.05 GB of memory).
If this limit is breached, the Kubernetes API will restart the faulting container.

Note that your mileage with these resource requests and limits may vary.
Any number of factors may contribute to varying results in resource utilization (different compute instance types, workload utilization, API ingress/egress, etc).
[The Kubernetes documentation for Resource Management of Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
is an excellent resource for understanding how various compute resources are utilized
and how Kubernetes manages these resources.

If resource utilization by the brupop components is not a concern,
removing the `resources` fields in the manifest will not affect the functionality of any components.

##### Exclude Nodes from Load Balancers Before Draining
This configuration uses Kubernetes [ServiceNodeExclusion](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) feature.
`EXCLUDE_FROM_LB_WAIT_TIME_IN_SEC` can be used to enable the feature to exclude the node from load balancer before draining.
When `EXCLUDE_FROM_LB_WAIT_TIME_IN_SEC` is 0 (default), the feature is disabled.
When `EXCLUDE_FROM_LB_WAIT_TIME_IN_SEC` is set to a positive integer, bottlerocket update operator will exclude the node from
load balancer and then wait `EXCLUDE_FROM_LB_WAIT_TIME_IN_SEC` seconds before draining the pods on the node.

To enable this feature, set the `exclude_from_lb_wait_time_in_sec` value in your helm values yaml file to a positive integer. For example,
`exclude_from_lb_wait_time_in_sec: "100"`.

Otherwise, go to `bottlerocket-update-operator.yaml` and change `EXCLUDE_FROM_LB_WAIT_TIME_IN_SEC` to a positive integer value.
For example:
```yaml
      ...
      containers:
        - command:
            - "./agent"
          env:
            - name: MY_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: EXCLUDE_FROM_LB_WAIT_TIME_IN_SEC
              value: "180"
      ...
```

##### Set Up Max Concurrent Update

`MAX_CONCURRENT_UPDATE` can be used to specify the max concurrent updates during updating.
When `MAX_CONCURRENT_UPDATE` is a positive integer, the bottlerocket update operator
will concurrently update up to `MAX_CONCURRENT_UPDATE` nodes respecting [PodDisruptionBudgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/).
When `MAX_CONCURRENT_UPDATE` is set to `unlimited`, bottlerocket update operator
will concurrently update all nodes respecting [PodDisruptionBudgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/).

Note: The `MAX_CONCURRENT_UPDATE` configuration does not work well with `EXCLUDE_FROM_LB_WAIT_TIME_IN_SEC` 
configuration, especially when `MAX_CONCURRENT_UPDATE` is set to `unlimited`, it could potentially exclude all 
nodes from load balancer at the same time.

To enable this feature, set the `max_concurrent_updates` value in your helm values yaml file to a positive integer value or `unlimited`. For example,
`max_concurrent_updates: "1"` or `max_concurrent_updates: "unlimited"`.

Otherwise, go to `bottlerocket-update-operator.yaml` and change `MAX_CONCURRENT_UPDATE` to a positive integer value or `unlimited`.
For example:
```yaml
      containers:
        - command:
            - "./controller"
          env:
            - name: MY_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: MAX_CONCURRENT_UPDATE
              value: "1"
```

##### Set scheduler
`SCHEDULER_CRON_EXPRESSION` can be used to specify the scheduler in which updates are permitted.
When `SCHEDULER_CRON_EXPRESSION` is "* * * * * * *" (default), the feature is disabled.

To enable this feature, set the `scheduler_cron_expression` value in your helm values yaml file.
This value should be a valid cron expression.
A cron expression can be configured to a time window or a specific trigger time.
When users specify cron expressions as a time window, the bottlerocket update operator will operate node updates within that update time window.
When users specify cron expression as a specific trigger time, brupop will update and complete all waitingForUpdate nodes on the cluster when that time occurs.
```
# ┌───────────── seconds (0 - 59)
# | ┌───────────── minute (0 - 59)
# | │ ┌───────────── hour (0 - 23)
# | │ │ ┌───────────── day of the month (1 - 31)
# | │ │ │ ┌───────────── month (Jan, Feb, Mar, Apr, Jun, Jul, Aug, Sep, Oct, Nov, Dec)
# | │ │ │ │ ┌───────────── day of the week (Mon, Tue, Wed, Thu, Fri, Sat, Sun)
# | │ │ │ │ │ ┌───────────── year (formatted as YYYY)
# | │ │ │ │ │ |
# | │ │ │ │ │ |
# * * * * * * *
```

Note: brupop uses Coordinated Universal Time(UTC), please convert your local time to Coordinated Universal Time (UTC). This tool [Time Zone Converter](https://www.timeanddate.com/worldclock/converter.html) can help you find your desired time window on UTC.
For example (schedule to run update operator at 03:00 PM on Monday ):
```yaml
      containers:
        - command:
            - "./controller"
          env:
            - name: MY_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: SCHEDULER_CRON_EXPRESSION
              value: "* * * * * * *"
```

##### Set an Update Time Window - DEPRECATED

**Note**: these settings are deprecated and will be removed in a future release.
Time window settings cannot be used in combination with the preferred cron expression format and will be ignored.

If you still decide to use these settings, please use "hour:00:00" format only instead of "HH:MM:SS".

`UPDATE_WINDOW_START` and `UPDATE_WINDOW_STOP` can be used to specify the time window in which updates are permitted.

To enable this feature, set the `update_window_start` and `update_window_stop` values in your helm values yaml file to a `hour:minute:second` formatted value (UTCE 24-hour time notation).
For example: `update_window_start: "08:0:0"` and `update_window_stop: "12:30:0"`.

Otherwise, go to `bottlerocket-update-operator.yaml` and change `UPDATE_WINDOW_START` and `UPDATE_WINDOW_STOP` to a `hour:minute:second` formatted value (UTC (24-hour time notation)). 

Note that `UPDATE_WINDOW_START` is inclusive and `UPDATE_WINDOW_STOP` is exclusive.

Note: brupop uses UTC (24-hour time notation), please convert your local time to UTC.
For example:
```yaml
      containers:
        - command:
            - "./controller"
          env:
            - name: MY_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: UPDATE_WINDOW_START
              value: "09:00:00"
            - name: UPDATE_WINDOW_STOP
              value: "21:00:00"
```

### Label nodes

By default, each Workload resource constrains scheduling of the update operator by limiting pods to Bottlerocket nodes based on their labels.
These labels are not applied on nodes automatically and will need to be set on each desired node using `kubectl`.
The agent relies on each node's updater components and schedules its pods based on their interface supported.
The node indicates its updater interface version in a label called `bottlerocket.aws/updater-interface-version`.
Agent deployments, respective to the interface version, are scheduled using this label and target only a single version in each.

For versions > `0.2.0` of the Bottlerocket update operator, only `update-interface-version` `2.0.0` is supported, which uses Bottlerocket's [update API](https://github.com/bottlerocket-os/bottlerocket/blob/develop/sources/updater/README.md#update-api) to dispatch updates.
For this reason, only Bottlerocket OS versions > `v0.4.1` are supported.

For the `2.0.0` `updater-interface-version`, this label looks like:

``` text
bottlerocket.aws/updater-interface-version=2.0.0
```

With [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) configured for the desired cluster, you can use the below command to get all nodes:

```sh
kubectl get nodes
```
Make a note of all the node names that you would like the Bottlerocket update operator to manage.

Next, add the `updater-interface-version` label to the nodes. 
For each node, use this command to add `updater-interface-version` label. 
Make sure to change `NODE_NAME` with the name collected from the previous command:

```sh
kubectl label node NODE_NAME bottlerocket.aws/updater-interface-version=2.0.0
```

If all nodes in the cluster are running Bottlerocket and require the same `updater-interface-version`, you can label all at the same time by running this:
```sh
kubectl label node $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}') bottlerocket.aws/updater-interface-version=2.0.0
```

#### Automatic labeling via Bottlerocket user-data

You can automatically [add Kubernetes labels to your Bottlerocket nodes via the settings provided in user data](https://github.com/bottlerocket-os/bottlerocket#kubernetes-settings)
when your nodes are provisioned:

```toml
# Configure the node-labels Bottlerocket setting
[settings.kubernetes.node-labels]
"bottlerocket.aws/updater-interface-version" = "2.0.0"
```

#### Automatic labeling via `eksctl`

If you're using `eksctl`, you can automatically add labels to nodegroups. For example:

```yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: bottlerocket-cluster
  region: us-west-2
  version: '1.17'

nodeGroups:
  - name: ng-bottlerocket
    labels: { bottlerocket.aws/updater-interface-version: 2.0.0 }
    instanceType: m5.large
    desiredCapacity: 3
    amiFamily: Bottlerocket
```

### Uninstalling

If you remove the `bottlerocket.aws/updater-interface-version=2.0.0` label from a node,
the Brupop controller will remove the resources from that node (including the `bottlerocketshadow` CRD associated with that node).

Deleting the custom resources, definitions, and deployments will then fully remove the bottlerocket update operator from your cluster:

```sh
helm uninstall brupop
helm uninstall brupop-crd
```

## Operation

### Overview

The update operator controller and agent processes communicate using Kubernetes [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/), with one being created for each node managed by the operator.
The Custom Resource created by the update operator is called a BottlerocketShadow resource, or otherwise shortened to `brs`.
The Custom Resource's Spec is configured by the controller to indicate a desired state, which guides the agent components.
The update operator's agent component keeps the Custom Resource Status updated with the current state of the node.
More about Spec and Status can be found in the [Kubernetes documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/#object-spec-and-status).

Additionally, the update operator's controller and apiserver components expose metrics which can be configured to be [collected by Prometheus](#monitoring-cluster-history-and-metrics-with-prometheus).

### Observing State

#### Monitoring Custom Resources

The current state of the cluster from the perspective of the update operator can be summarized by querying the Kubernetes API for BottlerocketShadow objects.
This view will inform you of the current Bottlerocket version of each node managed by the update operator, as well as the current ongoing update status of any node with an update in-progress.
The following command requires `kubectl` to be configured for the desired cluster to be monitored:

``` sh
kubectl get bottlerocketshadows --namespace brupop-bottlerocket-aws 
```

You can shorten this with:

``` sh
kubectl get brs --namespace brupop-bottlerocket-aws 
```

You should see output akin to the following:

```
$ kubectl get brs --namespace brupop-bottlerocket-aws 
NAME                                               STATE   VERSION   TARGET STATE   TARGET VERSION
brs-node-1                                         Idle    1.5.2     Idle           
brs-node-2                                         Idle    1.5.1     StagedUpdate   1.5.2
```

#### Monitoring Cluster History and Metrics with Prometheus

The update operator provides metrics endpoints which can be scraped by [Prometheus](https://prometheus.io/).
This allows you to monitor the history of update operations using popular metrics analysis and visualization tools.

We provide a [sample configuration](./deploy/examples/prometheus-resources.yaml) which demonstrates a Prometheus deployment into the cluster that is configured to gather metrics data from the update operator.

To deploy the sample configuration, you can use `kubectl`:

```sh
kubectl apply -f ./deploy/telemetry/prometheus-resources.yaml
```

Now that Prometheus is running in the cluster, you can use the UI provided to visualize the cluster's history.
Get the Prometheus pod name (e.g. `prometheus-deployment-5554fd6fb5-8rm25`):

```sh
kubectl get pods --namespace brupop-bottlerocket-aws 
```

Set up port forwarding to access Prometheus on the cluster:

```sh
kubectl port-forward $prometheus-pod-name 9090:9090 --namespace brupop-bottlerocket-aws 
```

Point your browser to `localhost:9090/graph` to access the sample Prometheus UI.

Search for:
* `brupop_hosts_state` to check how many hosts are in each state. 
* `brupop_hosts_version` to check how many hosts are in each Bottlerocket version.


### Image Region

`bottlerocket-update-operator.yaml` pulls operator images from Amazon ECR Public.
You may also choose to pull from regional Amazon ECR repositories such as the following.

  - `917644944286.dkr.ecr.af-south-1.amazonaws.com`
  - `375569722642.dkr.ecr.ap-east-1.amazonaws.com`
  - `328549459982.dkr.ecr.ap-northeast-1.amazonaws.com`
  - `328549459982.dkr.ecr.ap-northeast-2.amazonaws.com`
  - `328549459982.dkr.ecr.ap-northeast-3.amazonaws.com`
  - `328549459982.dkr.ecr.ap-south-1.amazonaws.com`
  - `328549459982.dkr.ecr.ap-southeast-1.amazonaws.com`
  - `328549459982.dkr.ecr.ap-southeast-2.amazonaws.com`
  - `386774335080.dkr.ecr.ap-southeast-3.amazonaws.com`
  - `328549459982.dkr.ecr.ca-central-1.amazonaws.com`
  - `328549459982.dkr.ecr.eu-central-1.amazonaws.com`
  - `328549459982.dkr.ecr.eu-north-1.amazonaws.com`
  - `586180183710.dkr.ecr.eu-south-1.amazonaws.com`
  - `328549459982.dkr.ecr.eu-west-1.amazonaws.com`
  - `328549459982.dkr.ecr.eu-west-2.amazonaws.com`
  - `328549459982.dkr.ecr.eu-west-3.amazonaws.com`
  - `553577323255.dkr.ecr.me-central-1.amazonaws.com`
  - `509306038620.dkr.ecr.me-south-1.amazonaws.com`
  - `328549459982.dkr.ecr.sa-east-1.amazonaws.com`
  - `328549459982.dkr.ecr.us-east-1.amazonaws.com`
  - `328549459982.dkr.ecr.us-east-2.amazonaws.com`
  - `328549459982.dkr.ecr.us-west-1.amazonaws.com`
  - `328549459982.dkr.ecr.us-west-2.amazonaws.com`
  - `388230364387.dkr.ecr.us-gov-east-1.amazonaws.com`
  - `347163068887.dkr.ecr.us-gov-west-1.amazonaws.com`
  - `183470599484.dkr.ecr.cn-north-1.amazonaws.com.cn`
  - `183901325759.dkr.ecr.cn-northwest-1.amazonaws.com.cn`

Example regional image URI:
```
328549459982.dkr.ecr.us-west-2.amazonaws.com/bottlerocket-update-operator:v1.1.0
```

### Current Limitations

- Monitoring on newly-rebooted nodes is limited.
  We are considering an approach in which custom health checks can be configured to run after reboots. (https://github.com/bottlerocket-os/bottlerocket/issues/503)
- Node labels are not automatically applied to allow scheduling (https://github.com/bottlerocket-os/bottlerocket/issues/504)

## Troubleshooting

When installed with the [default deployment](./bottlerocket-update-operator.yaml), the logs can be fetched through Kubernetes deployment logs.
Because mutations to a node are orchestrated through the API server component, searching those deployment logs for a node ID can be useful.
To get logs for the API server, run the following:

```sh
kubectl logs deployment/brupop-apiserver --namespace brupop-bottlerocket-aws 
```

The controller logs will usually not help troubleshoot issues about the state of updates in a cluster, but they can similarly be fetched:

```sh
kubectl logs deployment/brupop-controller-deployment --namespace brupop-bottlerocket-aws 
```

### Why are updates stuck in my cluster?
The bottlerocket update operator only installs updates on one node at a time.
If a node's update becomes stuck, it can prevent the operator from proceeding with updates across the cluster.

The update operator uses the [Kubernetes Eviction API](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/) to safely drain pods from a node.
The eviction API will respect [PodDisruptionBudgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/), refusing to remove a pod from a node if it would cause a PDB not to be satisfied.
It is possible to mistakenly configure a Kubernetes cluster in such a way that a Pod can never be deleted while still maintaining the conditions of a PDB.
In this case, the operator may become stuck waiting for the PDB to allow an eviction to proceed.

Similarly, if the Node in question is repeatedly encountering issues while updating, it may cause updates across the cluster to become stuck.
Such issues can be troubleshooted by requesting the update operator agent logs from the node.
First, list the agent pods and select the pod residing on the node in question:

```sh
kubectl get pods --selector=brupop.bottlerocket.aws/component=agent -o wide --namespace brupop-bottlerocket-aws
```

Then fetch the logs for that agent:

```sh
kubectl logs brupop-agent-podname --namespace brupop-bottlerocket-aws 
```

### Why are my bottlerocket nodes egressing to `https://updates.bottlerocket.aws`?

The [Bottlerocket updater API](https://github.com/bottlerocket-os/bottlerocket/blob/develop/sources/updater/README.md)
utilizes [TUF repository signed metadata](https://theupdateframework.io/metadata/)
served from a public URL to query and check for updates.
The URL is `https://updates.bottlerocket.aws` and Bottlerocket's updater system requires
access to this endpoint in order to perform updates.

Cluster nodes running in production environments with limited network access
may not be able to reach this metadata endpoint
and you may see failures when updates are available or an update statuses that appears "stuck".

To troubleshoot and validate this, [access one of your Bottlerocket nodes on your cluster](https://github.com/bottlerocket-os/bottlerocket/blob/9adbe40c3fced559b2d05829e62902060c5ecc9c/README.md#exploration),
and manually execute `apiclient update check`. If you see errors that indicate `Failed to check for updates`
and network timeouts, ensure that your nodes on cluster have access to `https://updates.bottlerocket.aws`.

If it is unacceptable to give your Bottlerocket nodes outside network access,
you may consider creating a self managed proxy within the cluster
that can periodically scrape the TUF repository from `https://updates.bottlerocket.aws`.
This would also require updating [the `settings.updates.metadata-base-url` and `settings.updates.targets-base-url` settings](https://github.com/bottlerocket-os/bottlerocket#updates-settings)
to point to your proxy.
[Typically, this is done via `tuftool download` or `tuftool clone`.](https://github.com/awslabs/tough/tree/develop/tuftool#download-tuf-repo)

Users who are building _their own Bottlerocket variants and TUF repositories_
will also need to update their Bottlerocket `settings.updates` settings to point to their custom TUF repository.
But since Brupop simply interfaces with the node's `apiclient` via the daemonset,
no further configurations or changes are required in Brupop.
An in depth discussion on [building your own TUF repos can be found in the core Bottlerocket repo.](https://github.com/bottlerocket-os/bottlerocket/blob/develop/PUBLISHING.md#build-a-repo)

### Why do only some of my Bottlerocket instances have an update available?

Updates to Bottlerocket are rolled out in [waves](https://github.com/bottlerocket-os/bottlerocket/tree/develop/sources/updater/waves) to reduce the impact of issues; the container instances in your cluster may not all see updates at the same time.
You can check whether an update is available on your instance by running the `apiclient update check` command from within the [control](https://github.com/bottlerocket-os/bottlerocket#control-container) or [admin](https://github.com/bottlerocket-os/bottlerocket#admin-container) container.

### Why do new container instances launch with older Bottlerocket versions?

The Bottlerocket update operator performs in-place updates for instances in your Kubernetes cluster.
The operator does not influence how those instances are launched.
If you use an [auto-scaling group](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html) to launch your instances, you can update the AMI ID in your [launch configuration](https://docs.aws.amazon.com/autoscaling/ec2/userguide/LaunchConfiguration.html) or [launch template](https://docs.aws.amazon.com/autoscaling/ec2/userguide/LaunchTemplates.html) to use a newer version of Bottlerocket.

## How to Contribute and Develop Changes

Working on the update operator requires a fully configured and working Kubernetes cluster.
Because the agent component relies on the Bottlerocket API to properly function, we suggest a cluster which is running Bottlerocket nodes.
The `integ` crate can currently be used to launch Bottlerocket nodes into an [Amazon EKS](https://aws.amazon.com/eks/) cluster to observe update-operator behavior.

Have a look at the [design](./design/DESIGN.md) to learn more about how the update operator functions.
Please feel free to open an issue with an questions!

### Building and Deploying a Development Image

To re-generate the yaml manifest found at the root of this reposiroy, simply run:
```
make manifest
```
Note: this requires `helm` to be installed.

Targets in the `Makefile` can assist in creating an image.
The following command will build and tag an image using the local Docker daemon:

```sh
make brupop-image
```

Once this image is pushed to a container registry, you can set environment variables to regenerate a `.yaml` file suitable for deploying the image to Kubernetes.
Firstly, modify the `.env` file to contain the desired image name, as well as a secret for pulling the image if necessary.
Then run the following to regenerate the `.yaml` resource definitions:

```sh
cargo build -p deploy
```

These can of course be deployed using `kubectl apply` or the automatic integration testing tool [integ](https://github.com/bottlerocket-os/bottlerocket-update-operator/tree/develop/integ).

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This project is dual licensed under either the Apache-2.0 License or the MIT license, your choice.
