


# Optimizing GPU Utilization: GPU Sharing with MIG on RHOAI
Maximizing GPU utilization is a game-changer in modern computing, especially for AI and ML, where GPUs take the spotlight for their unmatched ability to handle parallel tasks and crunch massive datasets at lightning speed. With thousands of cores firing in parallel, modern GPUs excel at complex model training and real-time data analysis—tasks that traditional CPUs simply can’t keep up with.

By unlocking the full power of GPUs, organizations can supercharge MLOps workflows, accelerate time-to-insight, and boost infrastructure efficiency. The result? Reduced hardware costs, lower operational overheads, and a future-proof system that's agile enough to scale with ever-evolving demands. One powerful approach to optimizing GPU resources is sharing them across multiple workloads.

So, how do we squeeze every drop of performance from GPUs? The answer lies in **sharing**—leveraging **Virtual GPUs (vGPU), Multi-Instance GPUs (MIG), and GPU Time-Slicing** to allocate resources dynamically and maximize efficiency. Ready to push your GPUs to the limit? Let’s dive in!

In this article, we will focus on Multi-Instance GPU (MIG)

## Multi-Instance GPU (MIG)

The **Multi-Instance GPU (MIG)** feature in NVIDIA GPUs allows secure partitioning of a single GPU into **multiple isolated instances**, enabling multiple users to share GPU resources efficiently. This is ideal for workloads that don’t fully utilize a GPU, allowing parallel execution for maximum efficiency.

Each MIG instance has dedicated compute resources, memory pathways, and isolated L2 cache and DRAM bandwidth, ensuring **consistent performance with predictable throughput and latency**. By partitioning streaming multiprocessors (SMs) and other GPU engines, MIG provides **fault isolation and guaranteed QoS** for virtual machines (VMs), containers, and processes—allowing multiple workloads to run in parallel on a single physical GPU. With partitioning at the hardware level, it delivers improved performance with lower overhead and enhanced security.

Note - MIG is primarily available on GPUs based on the **NVIDIA Ampere architecture** and newer. Like NVIDIA A30, A40, A100, NVIDIA H100 etc.

![MIG Architecture (image credit: NVIDIA)](https://raw.githubusercontent.com/rohitralhan/GPUSharingMIG/refs/heads/main/images/mig-overview.jpg)
<p align=center>MIG Architecture (image credit: <a style="text-decoration:none" href="https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html">Nvidia</a>)
</p>

---

Consider a scenario where a large application with high memory and compute demands runs alongside smaller programs on the same GPU, the small programs experience significant slowdowns. The larger program saturates the GPU, causing the small programs that were expected to finish in 5 hours to take 15 hours instead. This results in poor user experience, as the smaller task’s delay is more noticeable, even though both programs are affected. 

The solution is as shown in the image below where the large model is give the right size/amount of MIG slices to complete their work without effecting the smaller programs which now have their own isolated GPU slice.


The following figure illustrates how MIG technology partitions an A100 GPU into smaller, virtual GPUs. These "smaller GPUs" have reduced memory and fewer computing cores compared to the original GPU.

![Workloads on MIG Partitions (image credit: NVIDIA)](https://raw.githubusercontent.com/rohitralhan/GPUSharingMIG/refs/heads/main/images/mig-multiWorkLoad.png)
<p align=center>Workloads on MIG Partitions<font size="2px"> (image credit: <a style="text-decoration:none" href="https://www.nvidia.com/en-gb/technologies/multi-instance-gpu/">NVIDIA</a>)</font>
</p>

---

### Key Features
 - **Hardware-Level Partitioning** - enables a single NVIDIA GPU to be split into **several separate GPU instances**, each with its own dedicated resources like memory and compute cores.
 - **Resource Customization** - **MIG slices** can be configured with varying amounts of **memory** and **compute power** based on workload requirements, ensuring optimal resource allocation.
 - **Security and Isolation** - hardware isolation guarantees **predictable performance** and **prevents data leakage** or potential security breaches between instances, offering a secure computing environment.
 - **Efficient Resource Utilization** - By enabling multiple workloads to share a single GPU, MIG maximizes GPU usage, improving **performance efficiency** and optimizing hardware costs


## Enable MIG on Red Hat OpenShift

### Prerequisite
 - Access to Red Hat OpenShift Cluster with MIG supported GPUs available
 - NVIDIA GPU Operator is Installed and Configured (GPU Cluster Policy etc.)
 - Cluster administrator access to your OpenShift cluster
 - Access to oc CLI tool for accessing the cluster from the command line

### MIG geometry
---
Geometry refers to how the user can create various partitions on the GPU. Partition size depends on the GPU being used and the driver version. For instance, the NVIDIA A30 24GB, offers multiple partitioning options:

-   **1g.6gb**: 1 Compute Instance (CI), 6GB memory
-   **2g.12gb**: 2 CIs, 12GB memory
-   **4g.24gb**: 4 CIs, 24GB memory

The NVIDIA A100 40GB, offers the following options:
-   **1g.5gb**: 1 Compute Instance (CI), 5GB memory
-   **2g.10gb**: 2 CIs, 10GB memory
-   **3g.20gb**: 3 CIs, 20GB memory
-   **4g.20gb**: 4 CIs, 20GB memory
-   **7g.40gb**: 7 CIs, 40GB memory

### MIG advertisement strategies
---
There are two MIG strategies: the **`Single`**, where all GPUs on the node are configured to use **the same MIG profile** (i.e., same number and size of MIG instances), and the **`Mixed`** MIG strategy, GPU can be partitioned into multiple instances with varying sizes, allowing different workloads to run concurrently on the same GPU. Set `mig strategy` to `mixed` when MIG mode is not enabled on all GPUs on a node.

### MIG Setup on Red Hat OpenShift
---
For each MIG configuration, you define a strategy and a MIG configuration label. By default, MIG is disabled and set to the **`single`** strategy. This example demonstrates how to configure a **`mixed`** strategy with the **`all-balanced`** configuration on an NVIDIA A30-24GB, using 2 GPUs across two worker nodes.

#### Procedure

The following steps will demonstrate how to apply ``` mixed ``` MIG strategy with MIG configuration label ``` all-balanced ```. Run the following command, this will get all the MIG related labels from all nodes 

```
oc describe node | grep nvidia.com/mig
```
Sample Output
```
nvidia.com/mig.capable=true
nvidia.com/mig.config=all-disabled
nvidia.com/mig.config.state=success
nvidia.com/mig.strategy=single
```

 1. Set the following environment variables for the NODE_NAME (for each node), STRATEGY and MIG_CONFIGURATION 
	```
	NODE_NAME1=worker1.host.com
	STRATEGY=mixed
	MIG_CONFIGURATION=all-balanced
	```
 2. To apply the strategy run the following command
	```
	oc patch clusterpolicy/gpu-cluster-policy --type='json' \
	 -p='[{"op": "replace", "path": "/spec/mig/strategy", "value": '$STRATEGY'}]'
	``` 
 3. Next, label the node with the configuration label
	```
	oc label node $NODE_NAME nvidia.com/mig.config=$MIG_CONFIGURATION --overwrite
	```

	The MIG manager assigns a `mig.config.state` label to the GPU, then terminates all GPU pods to enable MIG mode and configure the GPU according to the specified settings.

 4. Verify that MIG manager configured the GPUs:
	```
	oc describe node | grep nvidia.com/mig.config
	nvidia.com/mig.config=all-balanced
	nvidia.com/mig.config.state=success
	``` 
 5. Next we will verify the GPU configuration
	```
	oc describe node | egrep "Name:|Roles:|Capacity|nvidia.com/gpu: \
	|nvidia.com/mig-.* |Allocatable:|Requests +Limits"
	```
	```
	Name: worker1.host.com
	Roles:  worker
	Capacity:
		nvidia.com/gpu:  		1
		nvidia.com/mig-1g.6gb: 		2
		nvidia.com/mig-2g.12gb:  	1
	Allocatable:
		nvidia.com/gpu:  		0
		nvidia.com/mig-1g.6gb: 		2
		nvidia.com/mig-2g.12gb:  	1
		Resource  				Requests  Limits
		nvidia.com/mig-1g.6gb 		0 		0
		nvidia.com/mig-2g.12gb  	0 		0
	```
 6. Alternatively(Optional): Start a pod to run the `nvidia-smi` command and display the GPU resources.
	 1. Start a pod
		```
		cat <<EOF | oc apply -f -
		apiVersion: v1
		kind: Pod
		metadata:
		 name: nvidia-smi-pod
		spec:
		 restartPolicy: Never
		 containers:
		 - name: cuda-container
		 image: nvcr.io/nvidia/cuda:12.1.0-base-ubi8
		 command: ["/bin/sh","-c"]
		 args: ["nvidia-smi"]
		EOF
		```
		This pod will start and run the `nvidia-smi` command which will show us the MIG configuration with 3 MIG devices.
		
	 2. List the pod
		```
		$ oc get pods
		NAME                 READY   STATUS      RESTARTS   AGE
		nvidia-smi-pod   0/1     Completed   0          3m34s
		```
	 3. Confirm that the `nvidia-smi` output includes 3 MIG devices:
		 ```
		 oc logs nvidia-smi-pod
		 ```
		 Sample Output
	```
	+-----------------------------------------------------------------------------------------+
	| NVIDIA-SMI 550.90.07              Driver Version: 550.90.07      CUDA Version: 12.4     |
	|-----------------------------------------+------------------------+----------------------+
	| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
	| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
	|                                         |                        |               MIG M. |
	|=========================================+========================+======================|
	|   0  NVIDIA A30                     On  |   00000000:CA:00.0 Off |                   On |
	| N/A   30C    P0             37W /  165W |      51MiB /  24576MiB |     N/A      Default |
	|                                         |                        |              Enabled |
	+-----------------------------------------+------------------------+----------------------+

	+-----------------------------------------------------------------------------------------+
	| MIG devices:                                                                            |
	+------------------+----------------------------------+-----------+-----------------------+
	| GPU  GI  CI  MIG |                     Memory-Usage |        Vol|      Shared           |
	|      ID  ID  Dev |                       BAR1-Usage | SM     Unc| CE ENC DEC OFA JPG    |
	|                  |                                  |        ECC|                       |
	|==================+==================================+===========+=======================|
	|  0    1   0   0  |              26MiB / 11968MiB    | 28      0 |  2   0    2    0    0 |
	|                  |                 0MiB / 16383MiB  |           |                       |
	+------------------+----------------------------------+-----------+-----------------------+
	|  0    5   0   1  |              13MiB /  5952MiB    | 14      0 |  1   0    1    0    0 |
	|                  |                 0MiB /  8191MiB  |           |                       |
	+------------------+----------------------------------+-----------+-----------------------+
	|  0    6   0   2  |              13MiB /  5952MiB    | 14      0 |  1   0    1    0    0 |
	|                  |                 0MiB /  8191MiB  |           |                       |
	+------------------+----------------------------------+-----------+-----------------------+
	                                                                                       
	+-----------------------------------------------------------------------------------------+
	| Processes:                                                                              |
	|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
	|        ID   ID                                                               Usage      |
	|=========================================================================================|
	|  No running processes found                                                             |
	+-----------------------------------------------------------------------------------------+
	```
	 4. Delete the sample pod created above:
		 ```
		 $ oc delete pod command-nvidia-smi
		pod "command-nvidia-smi" deleted
		 ```	
---

## Configure Custom MIG Profiles
By default, the NVIDIA GPU Operator in OpenShift creates a `default-mig-parted-config` ConfigMap. The MIG Manager uses this ConfigMap to apply a consistent MIG configuration to all MIG-capable GPUs in the cluster. Unless overridden, all such GPUs will follow the profiles defined in this default config—ensuring a standardized partitioning strategy across nodes.

This behavior enables consistent and declarative management of GPU partitioning across nodes and simplifies the automated deployment of standard MIG configurations in OpenShift environments.

Assume each worker node in an OpenShift cluster has 4 A100 GPUs. Depending on your use case, you want to enable MIG on only 2 of the GPUs, leaving the remaining GPUs unpartitioned. To achieve this, you can define a custom MIG profile that specifies which GPUs should be configured with MIG and which should remain in full GPU mode.

Follow the steps below to achieve this:

 1. Start by preparing a custom `configmap` resource file for example `custom_configmap.yaml`. Use the [configmap](https://gitlab.com/nvidia/kubernetes/gpu-operator/-/blob/v1.8.0/assets/state-mig-manager/0400_configmap.yaml) as guidance to help you build that custom configuration. For more documentation about the file format see [mig-parted](https://github.com/NVIDIA/mig-parted).

	For a list of all supported combinations and placements of profiles on A100 and A30, refer to the section on [supported profiles](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html#supported-profiles).

 2. Create the custom configuration within the  `nvidia-gpu-operator`  namespace:
	  ```
	  $ CONFIG_FILE=/path/to/custom_configmap.yaml && \   
	  oc create configmap custom-mig-parted-config \   
	  --from-file=config.yaml=$CONFIG_FILE \   
	  -n nvidia-gpu-operator
    ```
3.	If the custom configuration specifies more than one instance profile, set the strategy to `mixed`:
	```
	oc patch clusterpolicies.nvidia.com/cluster-policy \
	--type='json' \
	-p='[{"op":"replace", "path":"/spec/mig/strategy", "value":"mixed"}]'
	```
4.	Patch the cluster policy so MIG Manager uses the custom config map:
	```
	oc patch clusterpolicies.nvidia.com/cluster-policy \
	--type='json' \
	-p='[{"op":"replace", "path":"/spec/migManager/config/name", "value":"custom-mig-config"}]'
	```
5.	Label the nodes with the profile to configure:
	```
	kubectl label nodes <node-name> nvidia.com/mig.config=custom-config --overwrite
	```
6.	Optional: Monitor the MIG Manager logs to confirm the new MIG geometry is applied:
	```
	oc logs -n nvidia-gpu-operator -l app=nvidia-mig-manager -c nvidia-mig-manager
	```


	```
	Applying the selected MIG config to the node
	time="2024-05-15T13:40:08Z" level=debug msg="Parsing config file..."
	time="2024-05-15T13:40:08Z" level=debug msg="Selecting specific MIG config..."
	time="2024-05-15T13:40:08Z" level=debug msg="Running apply-start hook"
	time="2024-05-15T13:40:08Z" level=debug msg="Checking current MIG mode..."
	.
	.
	.
	.
	time="2024-05-15T13:40:09Z" level=debug msg="Running apply-exit hook"
	MIG configuration applied successfully
	```

Here’s an example snippet from a `custom-config.yaml` file. In this setup, the node has 8 A100 GPUs. As you can see under the `custom-config` section, GPUs 0–3 are left unpartitioned, while GPUs 4–7 are configured with MIG and assigned various instance profiles. This allows you to run mixed workloads efficiently on the same node—maximizing GPU utilization without overcommitting resources.
```
version: v1
mig-configs:
  all-disabled:
    - devices: all
      mig-enabled: false

  all-enabled:
    - devices: all
      mig-enabled: true
      mig-devices: {}

  all-1g.5gb:
    - devices: all
      mig-enabled: true
      mig-devices:
        "1g.5gb": 7

  all-2g.10gb:
    - devices: all
      mig-enabled: true
      mig-devices:
        "2g.10gb": 3

  all-3g.20gb:
    - devices: all
      mig-enabled: true
      mig-devices:
        "3g.20gb": 2

  all-balanced:
    - devices: all
      mig-enabled: true
      mig-devices:
        "1g.5gb": 2
        "2g.10gb": 1
        "3g.20gb": 1

  custom-config:
    - devices: [0,1,2,3]
      mig-enabled: false
    - devices: [4]
      mig-enabled: true
      mig-devices:
        "1g.5gb": 7
    - devices: [5]
      mig-enabled: true
      mig-devices:
        "2g.10gb": 3
    - devices: [6]
      mig-enabled: true
      mig-devices:
        "3g.20gb": 2
    - devices: [7]
      mig-enabled: true
      mig-devices:
        "1g.5gb": 2
        "2g.10gb": 1
        "3g.20gb": 1
```
## Disable the MIG on Red Hat OpenShift

 To utilize the full capacity of the GPU you can always disable/turn off MIG by executing the following command
 ```
MIG_CONFIGURATION=all-disabled && \
  oc label node/$NODE_NAME nvidia.com/mig.config=$MIG_CONFIGURATION --overwrite
 ```
 The MIG manager assigns a `mig.config.state` label to the GPU, then terminates all GPU pods to enable MIG mode and configure the GPU according to the specified settings.

## Conclusion
NVIDIA Multi-Instance GPU (MIG) technology enables efficient GPU resource utilization by partitioning a single GPU into multiple independent instances. This ensures optimal performance for diverse workloads, from AI/ML training to inference and data analytics. By leveraging MIG on platforms like Red Hat OpenShift AI, organizations can maximize GPU efficiency, enhance multi-tenancy, and reduce infrastructure costs. Implementing MIG provides a scalable and flexible approach to GPU acceleration, making it a crucial tool for modern AI and high-performance computing environments.

## References
 [MIG on Red Hat OpenShift Container Platform](https://docs.nvidia.com/datacenter/cloud-native/openshift/latest/mig-ocp.html?target=_blank)
 
 [MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/?target=_blank)
 
 [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html?target=_blank)
