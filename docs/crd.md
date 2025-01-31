# Flink Application Custom Resource Definition
The [flinkapplication](https://github.com/lyft/flinkk8soperator/blob/master/deploy/crd.yaml) is a [kubernetes custom resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/). Once the *flinkapplication* custom resource is created in Kubernetes, the FlinkK8sOperator watches the resource and tries to move it through a series of states until the desired state is reached.

[FlinkApplication Custom Resource Example](https://github.com/lyft/flinkk8soperator/blob/master/examples/wordcount/flink-operator-custom-resource.yaml)

The type information is available here [FlinkApplication Type](https://github.com/lyft/flinkk8soperator/blob/master/pkg/apis/app/v1beta1/types.go#L25)

Below is the list of fields in the custom resource and their description

* **spec** `type:FlinkApplicationSpec required=True`
  Contains the entire specification of the flink application.

  * **image** `type:string required=True`
    The image name format should be registry/repository[:tag] to pull by tag, or registry/repository[@digest] to pull by digest

  * **imagePullPolicy** `type:v1.PullPolicy`
    The default pull policy is IfNotPresent which causes the Kubelet to skip pulling an image if it already exists.

  * **imagePullSecrets** `type:[]v1.LocalObjectReference`
    Indicates name of Secrets, Kubernetes should get the credentials from.

  * **taskManagerConfig** `type:TaskManagerConfig required=true`
    Configuration for the Flink task manager

    * **resources** `type:ResourceRequirements`
      Resources for the task manager. This includes cpu, memory, storage, and ephemeral-storage. If empty the operator will
      use a default value for cpu and memory.

    * **envConfig** `type:EnvironmentConfig`
      Configuration for setting environment variables in the task manager.

    * **taskSlots** `type:int32 required=true`
      Number of task slots per task manager

    * **offHeapMemoryFraction** `type:float64`
      A value between 0 and 1 that represents % of container memory dedicated to system / off heap. The
      remaining memory is allocated for heap.

    * **nodeSelector** `type:map[string]string`
      Configuration for the node selectors used for the task manager

  * **jobManagerConfig** `type:JobManagerConfig`
    Configuration for the Flink job manager

    * **resources** `type:ResourceRequirements`
      Resources for the job manager. This includes cpu, memory, storage, and ephemeral-storage. If empty the operator will
      use a default value for cpu and memory.

    * **envConfig** `type:EnvironmentConfig`
      Configuration for setting environment variables in the job manager.

    * **replicas** `type:int32 required=true`
      Number of job managers for the flink cluster. If multiple job managers are provided, the user has to ensure that
      correct environment variables are set for High availability mode.

    * **offHeapMemoryFraction** `type:float64`
      A value between 0 and 1 that represents % of container memory dedicated to system / off heap. The
      remaining memory is allocated for heap.

    * **nodeSelector** `type:map[string]string`
      Configuration for the node selectors used for the job manager

  * **jarName** `type:string required=true`
    Name of the jar file to be run. The application image needs to ensure that the jar file is present at the right location, as
    the operator uses the Web API to submit jobs.

  * **parallelism** `type:int32 required=true`
    Job level parallelism for the Flink Job

  * **entryClass** `type:string`
    Entry point for the Flink job

  * **programArgs** `type:string`
    External configuration parameters to be passed as arguments to the job like input and output sources, etc
    
  * **allowNonRestoredState** `type:boolean`
    Skips savepoint operator state that cannot be mapped to the new program version  

  * **savepointInfo** `type:SavepointInfo`
    Optional Savepoint info that can be passed in to indicate that the Flink job must resume from the corresponding savepoint.

  * **flinkVersion** `type:string required=true`
    The version of Flink to be managed. This version must match the version in the image.

  * **flinkConfig** `type:FlinkConfig`
    Optional map of flink configuration, which passed on to the deployment as environment variable with `OPERATOR_FLINK_CONFIG`

  * **deploymentMode** `type:DeploymentMode`
    Indicates the type of deployment that operator should perform if the custom resource is updated. Currently only Dual is supported.

    `Dual` This deployment mode is intended for applications where downtime during deployment needs to be as minimal as possible. In this deployment mode, the operator brings up a second Flink cluster with the new image, while the original Flink cluster is still active. Once the pods and containers in the new flink cluster are ready, the Operator cancels the job in the first Cluster with savepoint, deletes the cluster and starts the job in the second cluster. (More information in the state machine section below). This mode is suitable for real time processing applications.

  * **deleteMode** `type:DeleteMode`
    Indicates how Flink jobs are torn down when the FlinkApplication resource is deleted

    `Savepoint` (default) The operator will take a final savepoint before cancelling the job, and will not tear down the cluster until a savepoint has succeeded.

    `ForceCancel` The operator will force cancel the job before tearing down the cluster

    `None` The operator will immediately tear down the cluster

  * **restartNonce** `type:string`
    Can be set or modified to force a restart of the cluster

  * **volumes** `type:[]v1.Volume`
    Represents a named volume in a pod that may be accessed by any container in the pod.

  * **volumeMounts** `type:[]v1.VolumeMount`
    Describes a mounting of a Volume within a container.
    
  * **ForceRollback** `type:bool`
    Can be set to true to force rollback a deploy/update. The rollback is **not** performed when the application is in a **RUNNING** phase.
    If an application is successfully rolled back, it is moved to a *DeployFailed* phase. Un-setting or setting `ForceRollback` to `False` will allow updates to progress normally.
    
