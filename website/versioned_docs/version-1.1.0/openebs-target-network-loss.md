---
id: version-1.1.0-openebs-target-network-loss
title: OpenEBS Target Network Loss Experiment Details
sidebar_label: Target Network Loss
original_id: openebs-target-network-loss
---
------

## Experiment Metadata

<table>
  <tr>
    <th> Type </th>
    <th> Description </th>
    <th> Tested K8s Platform </th>
  </tr>
  <tr>
    <td> OpenEBS </td>
    <td> Induce network loss into the cStor target/Jiva controller container </td>
    <td> GKE, Konvoy(AWS), Packet(Kubeadm), Minikube, OpenShift(Baremetal) </td>
  </tr>
</table>

<b>Note:</b> In this example, we are using nginx as stateful application that stores static pages on a Kubernetes volume. 

## Prerequisites

- Ensure that the Kubernetes Cluster uses Docker runtime
- Ensure that the Litmus Chaos Operator is running by executing `kubectl get pods` in operator namespace (typically, `litmus`). If not, install from [here](https://raw.githubusercontent.com/litmuschaos/pages/master/docs/litmus-operator-latest.yaml)
- Ensure that the `openebs-target-network-loss` experiment resource is available in the cluster. If not, install from [here](https://hub.litmuschaos.io/charts/openebs/experiments/openebs-target-network-loss)
- The DATA_PERSISTENCE can be enabled by provide the application's info in a configmap volume so that the experiment can perform necessary checks. Currently, LitmusChaos supports data consistency checks only for MySQL and Busybox. 
    - For MYSQL data persistence check create a configmap as shown below in the application namespace (replace with actual credentials):

    ```
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: openebs-target-network-loss
    data:
      parameters.yml: | 
        dbuser: root
        dbpassword: k8sDem0
        dbname: test
    ```
    - For Busybox data persistence check create a configmap as shown below in the application namespace (replace with actual credentials):

    ```
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: openebs-target-network-loss
    data:
      parameters.yml: | 
        blocksize: 4k
        blockcount: 1024
        testfile: exampleFile
    ```
- Ensure that the chaosServiceAccount used for the experiment has cluster-scope permissions as the experiment may involve carrying out the chaos in the `openebs` namespace
  while performing application health checks in its respective namespace. 

## Entry Criteria

- Application pods are healthy before chaos injection
- Application writes are successful on OpenEBS PVs

## Exit Criteria

- Stateful application pods are healthy post chaos injection
- OpenEBS Storage target pods are healthy

If the experiment tunable DATA_PERSISTENCE is set to 'enabled':

- Application data written prior to chaos is successfully retrieved/read 
- Database consistency is maintained as per db integrity check utils 

## Details

- This scenario validates the behaviour of stateful applications and OpenEBS data plane upon high latencies/network loss in accessing the storage controller pod
- Injects network loss on the specified container in the controller pod by starting a traffic control `tc` process with `netem` rules to add egress delays
- Network loss is injected via pumba library with command `pumba netem delay` by passing the relevant network interface, network loss, chaos duration and regex filter for container name
- Can test the stateful application's resilience to loss/slow iSCSI connections

## Integrations

- Network loss is achieved using the `pumba` chaos library in case of docker runtime. Support for other other runtimes via tc direct invocation of `tc` will be added soon. 
- The desired lib image can be configured in the env variable `LIB_IMAGE`. 

## Steps to Execute the Chaos Experiment

- This Chaos Experiment can be triggered by creating a ChaosEngine resource on the cluster. To understand the values to provide in a ChaosEngine specification, refer [Getting Started](getstarted.md/#prepare-chaosengine)

- Follow the steps in the sections below to prepare the ChaosEngine & execute the experiment.

### Prepare chaosServiceAccount

Use this sample RBAC manifest to create a chaosServiceAccount in the desired (app)namespace. This example consists of the minimum necessary cluster role permissions to execute the experiment.

#### Sample Rbac Manifest

[embedmd]:# (https://raw.githubusercontent.com/litmuschaos/chaos-charts/master/charts/openebs/openebs-target-network-loss/rbac.yaml yaml)
```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: target-network-loss-sa
  namespace: default
  labels:
    name: target-network-loss-sa
---
# Source: openebs/templates/clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: target-network-loss-sa
  labels:
    name: target-network-loss-sa
rules:
- apiGroups: ["","apps","litmuschaos.io","batch","extensions","storage.k8s.io"]
  resources: ["pods","pods/exec","jobs","configmaps","secrets","services","persistentvolumeclaims","storageclasses","persistentvolumes","chaosexperiments","chaosresults","chaosengines"]
  verbs: ["create","list","get","patch","update","delete"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: target-network-loss-sa
  labels:
    name: target-network-loss-sa
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: target-network-loss-sa
subjects:
- kind: ServiceAccount
  name: target-network-loss-sa
  namespace: default
```

### Prepare ChaosEngine

- Provide the application info in `spec.appinfo`
- Provide the auxiliary applications info (ns & labels) in `spec.auxiliaryAppInfo`
- Override the experiment tunables if desired

#### Supported Experiment Tunables

<table>
  <tr>
    <th> Variables </th>
    <th> Description  </th>
    <th> Type </th>
    <th> Notes </th>
  </tr>
  <tr>
    <td> APP_PVC </td>
    <td> The PersistentVolumeClaim used by the stateful application </td>
    <td> Mandatory </td>
    <td> PVC may use either OpenEBS Jiva/cStor storage class </td>
  </tr>
  <tr>
    <td> LIB_IMAGE </td>
    <td> chaos library image used to inject the network loss </td>
    <td> Optional  </td>
    <td> Defaults to `gaiaadm/pumba:0.6.5`. Supported: `docker : gaiaadm/pumba:0.6.5` </td>
  </tr>
  <tr>
    <td> TOTAL_CHAOS_DURATION </td>
    <td> Total duration for which network loss is injected </td>
    <td> Optional </td>
    <td> Defaults to 120 seconds </td>
  </tr>
  <tr>
    <td> DEPLOY_TYPE </td>
    <td> Type of Kubernetes resource used by the stateful application </td>
    <td> Optional  </td>
    <td> Defaults to `deployment`. Supported: `deployment`, `statefulset` </td>
  </tr>
  <tr>
    <td> NETWORK_PACKET_LOSS_PERCENTAGE </td>
    <td> Total percentage for which network loss is injected </td>
    <td> Optional  </td>
    <td>  Defaults to 100 (percent) </td>
  </tr>
  <tr>
    <td> DATA_PERSISTENCE </td>
    <td> Flag to perform data consistency checks on the application </td>
    <td> Optional  </td>
    <td> Default value is disabled (empty/unset). It supports only `mysql` and `busybox`. Ensure configmap with app details are created </td>
  </tr>
</table>

#### Sample ChaosEngine Manifest

[embedmd]:# (https://raw.githubusercontent.com/litmuschaos/chaos-charts/master/charts/openebs/openebs-target-network-loss/engine.yaml yaml)
```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: target-chaos
  namespace: default
spec:
  # It can be true/false
  annotationCheck: 'false'
  # It can be active/stop
  engineState: 'active'
  #ex. values: ns1:name=percona,ns2:run=nginx 
  auxiliaryAppInfo: ''
  appinfo:
    appns: 'default'
    applabel: 'app=nginx'
    appkind: 'deployment'
  chaosServiceAccount: target-network-loss-sa
  monitoring: false
  # It can be delete/retain
  jobCleanUpPolicy: 'delete'
  experiments:
    - name: openebs-target-network-loss
      spec:
        components:
          env:
            - name: TARGET_CONTAINER
              value: 'cstor-istgt'
            - name: APP_PVC
              value: 'demo-nginx-claim'    
            - name: DEPLOY_TYPE
              value: 'deployment'       
            - name: TOTAL_CHAOS_DURATION
              value: '120000' 
```

### Create the ChaosEngine Resource

- Create the ChaosEngine manifest prepared in the previous step to trigger the Chaos.

  `kubectl apply -f chaosengine.yml`

### Watch Chaos progress

- View network loss in action by setting up a ping to the storage controller in the OpenEBS namespace
- Watch the behaviour of the application pod and the OpenEBS data replica/pool pods by setting up in a watch on the respective namespaces

  `watch -n 1 kubectl get pods -n <application-namespace>`

### Check Chaos Experiment Result

- Check whether the application is resilient to the target network loss, once the experiment (job) is completed. The ChaosResult resource naming 
  convention is: `<ChaosEngine-Name>-<ChaosExperiment-Name>`.

  `kubectl describe chaosresult target-chaos-openebs-target-network-loss -n <application-namespace>`

## OpenEBS Target Network Loss Demo [TODO]

- A sample recording of this experiment execution is provided here.