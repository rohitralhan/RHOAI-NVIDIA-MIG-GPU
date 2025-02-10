# Optimizing GPU Utilization: Nvidia GPU Sharing with MIG on Red Hat OpenShift AI

Maximizing GPU utilization is a game-changer in modern computing, especially for AI and ML, where GPUs take the spotlight for their unmatched ability to handle parallel tasks and crunch massive datasets at lightning speed. With thousands of cores firing in parallel, modern GPUs excel at complex model training and real-time data analysis—tasks that traditional CPUs simply can’t keep up with.

By unlocking the full power of GPUs, organizations can supercharge MLOps workflows, accelerate time-to-insight, and boost infrastructure efficiency. The result? Reduced hardware costs, lower operational overheads, and a future-proof system that's agile enough to scale with ever-evolving demands. One powerful approach to optimizing GPU resources is sharing them across multiple workloads.

So, how do we squeeze every drop of performance from GPUs? The answer lies in **sharing**—leveraging **Virtual GPUs (vGPU), Multi-Instance GPUs (MIG), and GPU Time-Slicing** to allocate resources dynamically and maximize efficiency. Ready to push your GPUs to the limit? Let’s dive in!

In this article, we will focus on Multi-Instance GPU (MIG)

## Multi-Instance GPU (MIG)

The **Multi-Instance GPU (MIG)** feature in NVIDIA GPUs allows secure partitioning of a single GPU into **multiple isolated instances**, enabling multiple users to share GPU resources efficiently. This is ideal for workloads that don’t fully utilize a GPU, allowing parallel execution for maximum efficiency.

Each MIG instance has dedicated compute resources, memory pathways, and isolated L2 cache and DRAM bandwidth, ensuring **consistent performance with predictable throughput and latency**. By partitioning streaming multiprocessors (SMs) and other GPU engines, MIG provides **fault isolation and guaranteed QoS** for virtual machines (VMs), containers, and processes—allowing multiple workloads to run in parallel on a single physical GPU. With partitioning at the hardware level, it delivers improved performance with lower overhead and enhanced security.

Note - MIG is available on GPUs based on the **NVIDIA Ampere architecture** and newer. Like NVIDIA A30, A40, A100, NVIDIA H100 etc.

![https://raw.githubusercontent.com/rohitralhan/RHOAI-NVIDIA-MIG-GPU/refs/heads/main/images/mig-overview.jpg)](https://raw.githubusercontent.com/rohitralhan/RHOAI-NVIDIA-MIG-GPU/refs/heads/main/images/mig-overview.jpg)
<p align=center>MIG Architecture (image credit: <a style="text-decoration:none" href="https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html">Nvidia</a>)
</p>

---

Consider a scenario where a large application with high memory and compute demands runs alongside smaller programs on the same GPU, the small programs experience significant slowdowns. The larger program saturates the GPU, causing the small programs that were expected to finish in 5 hours to take 15 hours instead. This results in poor user experience, as the smaller task’s delay is more noticeable, even though both programs are affected. 

The solution is as shown in the image below where a large model is give the right size/amount of MIG slices to complete their work without effecting the smaller programs which now have their own isolated GPU slice.


The following figure illustrates how MIG technology partitions an A100 GPU into smaller, virtual GPUs. These "smaller GPUs" have reduced memory and fewer computing cores compared to the original GPU.

![https://raw.githubusercontent.com/rohitralhan/RHOAI-NVIDIA-MIG-GPU/refs/heads/main/images/mig-multiWorkLoad.png)](https://raw.githubusercontent.com/rohitralhan/RHOAI-NVIDIA-MIG-GPU/refs/heads/main/images/mig-multiWorkLoad.png)
<p align=center>Workloads on MIG Partitions<font size="2px"> (image credit: <a style="text-decoration:none" href="https://www.nvidia.com/en-gb/technologies/multi-instance-gpu/">NVIDIA</a>)</font>
</p>

---

### Key Features
 - **Hardware-Level Partitioning** - enables a single NVIDIA GPU to be split into **several separate GPU instances**, each with its own dedicated resources like memory, compute cores etc..
 - **Resource Customization** - **MIG slices** can be configured with varying amounts of **memory** and **compute power** based on workload requirements, ensuring optimal resource allocation.
 - **Security and Isolation** - hardware isolation guarantees **predictable performance** and **prevents data leakage** or potential security breaches between instances, offering a secure computing environment.
 - **Efficient Resource Utilization** - By enabling multiple workloads to share a single GPU, MIG maximizes GPU usage, improving **performance efficiency** and optimizing hardware costs.


## Enable MIG on Red Hat OpenShift

### Prerequisite
 - Access to Red Hat OpenShift Cluster with GPUs available
 - NVIDIA GPU Operator is Installed and Configured (GPU Cluster Policy)
 - Cluster administrator access to your OpenShift cluster
 - oc CLI tool for accessing the cluster from the command line

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
There are two MIG strategies: the **`Single`** MIG strategy, used when all GPUs on a node have MIG enabled, and the **`Mixed`** MIG strategy, used when only some GPUs on a node have MIG enabled.

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
	NODE_NAME1=worker1.redhat.com
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

	Name: worker1.redhat.com
	Roles:  worker
	Capacity:
		nvidia.com/gpu:  		1
		nvidia.com/mig-1g.6gb: 		2
		nvidia.com/mig-2g.12gb:  	1
	Allocatable:
		nvidia.com/gpu:  		0
		nvidia.com/mig-1g.6gb: 		2
		nvidia.com/mig-2g.12gb:  	1
		Resource  			Requests  	Limits
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
		NAME             READY   STATUS      RESTARTS   AGE
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
## Using MIG in Red Hat OpenShift AI
MIG is now enabled, we can now go ahead and test it using Red Hat OpenShift AI (RHOAI) as explained below.
For using MIG profiles in Red Hat OpenShift AI we will need to:

 - Configure Accelerator Profiles in RHOAI Dashboard
 - Deploy a model server and the model with the associated MIG Profile
 - Scale the model to provide Load Balancing for the model server

#### Configure Accelerator Profiles in RHOAI Dashboard
An accelerator profile is a Custom Resource Definition (CRD) that specifies the configuration for a particular accelerator. It can be managed directly through the OpenShift AI Dashboard under **Settings → Accelerator Profiles**. Here we will create all the needed profiles corresponding to our MIG configuration **```mig-1g-6gb```** and **```mig-2g-12gb```** since the **```all-balanced```** MIG configuration for A30 GPUs created **```2 x 1g.6gb and 1 x 2g.12gb```**.

 1. Login to RHOAI dashboard
 
![enter image description here](https://raw.githubusercontent.com/rohitralhan/RHOAI-NVIDIA-MIG-GPU/refs/heads/main/images/RHOAILoginOut.gif)
<p align=center>Click <a style="text-decoration:none" target="_blank" rel="noopener" href="https://github.com/rohitralhan/RHOAI-NVIDIA-MIG-GPU/tree/main/images/mig-config-images/login">here</a> to see individual images
</p>


 2. Navigate to **`Settings → Accelerator Profiles`** click **`Create accelerated profiles`** button and follow the on-screen instructions to create two accelerator profiles **```mig-1g-6gb```** and **```mig-2g-12gb```**.

![enter image description here](https://raw.githubusercontent.com/rohitralhan/RHOAI-NVIDIA-MIG-GPU/refs/heads/main/images/CreateAccProfileOut.gif)
<p align=center>Click <a style="text-decoration:none" target="_blank" rel="noopener" href="https://github.com/rohitralhan/RHOAI-NVIDIA-MIG-GPU/tree/main/images/mig-config-images/acc-profile">here</a> to see individual images
</p>
---
### Deploy a model server and the model with the associated MIG Profile
Once the Accelerator Profiles are created, navigate to the data science project, create a model server, and deploy the model. For this demonstration, we are using an iris model available [rf_iris.onnx](https://raw.githubusercontent.com/rohitralhan/RHOAI-NVIDIA-MIG-GPU/refs/heads/main/models/rf_iris.onnx).

 1. Download the [rf_iris.onnx](https://raw.githubusercontent.com/rohitralhan/RHOAI-NVIDIA-MIG-GPU/refs/heads/main/models/rf_iris.onnx) model.
 2. Upload the model to a S3/S3 compatible bucket 
 3. Login to the Red Hat OpenShift AI console 
 4. Navigate to **```Data Science Projects → Create Project```**, follow the onscreen instructions to create the project
 5. Next create a data connection for saving the **```rf_iris.onnx```** models.
	 1. Navigate to the **```Connections```** tab and click **```Add Connection```** button
	 2. Follow the onscreen instructions for the **```S3 compatible object storage```** and click create   
 6. Now we will create a model server 
 7. Navigate to the **```Models```** tab and click **```Add model server```** button. Fill out the form with the following values:  
	 i. Model server name: infer-model-server  
	 ii. Serving runtime: **`OpenVINO Model Server`**  
	 iii. Model server replicas: **`1`**
	 iv. Model server size: **`Small`** (this can be adjusted according to the model needs)
	 v. Accelerator: **`NVIDIA mig-2g-12gb`** (the reference to the MIG partition created)
	 vi. Number of accelerators: **`1`**
	 vii. **`Enable`** the Make deployed models available through an external route option  
	 viii. **`Enable`** the Require token authentication option
 8. Click **```Add```** to add a model server.
 9. Once the model server is added click **```Deploy model```** on the right of the model server. Fill out the form with the following values:
	 i. Model name:  **```iris-model```**
	 ii. Model framework:  **`onnx - 1`**
	 iii. Select  Existing data connection
	 iv. Select the  **`iris-data-connection`**  data connection
	 v. Path:  **`iris`** or path to your model in the S3 buket
 10. Click **```Deploy```** to deploy the model

![enter image description here](https://raw.githubusercontent.com/rohitralhan/RHOAI-NVIDIA-MIG-GPU/refs/heads/main/images/DeployOut.gif)
<p align=center>Click <a style="text-decoration:none" target="_blank" rel="noopener" href="https://github.com/rohitralhan/RHOAI-NVIDIA-MIG-GPU/tree/main/images/mig-config-images/deploy">here</a> to see individual images
</p>
If you navigate to  **```Workloads → Pods → nvidia-driver-daemonset-***** pod → Terminal```** in the **```nvidia-gpu-operator```** project ,  type **```nvidia-smi```** it will show a similar output as shown in the image below. As you can see in our case the model server is using the MIG profile with GIID as 1 which is our 12 GB (```mig-2g-12gb```) MIG instance as selected above while creating the model server.

![enter image description here](https://raw.githubusercontent.com/rohitralhan/RHOAI-NVIDIA-MIG-GPU/refs/heads/main/images/NvidiaMig-2g.png)

---
### Scale model server for Load Balancing
Scaling a model server in **OpenShift** for **load balancing**, using **Multi-Instance GPU (MIG)** is a great way to optimize GPU resource utilization, especially on **NVIDIA  A or H series** or similar GPUs. Next we will look at how we can scale the model server in RHOAI:

Follow the steps below to scale the model server:<BR>
&nbsp;&nbsp;i. Login to Red Hat OpenShift AI<BR>
&nbsp;&nbsp;ii. Navigate to the project **```sample-project → Models tab```**<BR>
&nbsp;&nbsp;iii. Click the three dots next to **```Deploy Model```** button<BR>
&nbsp;&nbsp;iv. Click **```Edit model server```**<BR>
&nbsp;&nbsp;v. Increase the **```Number of model server replicas to deploy```** to 3<BR>
&nbsp;&nbsp;vi. Change the **```Accelerator```** to **```NVIDIA mig-1g-6gb```**<BR>
&nbsp;&nbsp;vii. Click the **```Update```** button to update/redeploy the model<BR>

The animation below shows multiple mig-1g-6gb partitions being utilized across two worker nodes having an NVIDIA A30 GPU each and two ```mig-1g-6gb``` partitions on each GPU.

![enter image description here](https://raw.githubusercontent.com/rohitralhan/RHOAI-NVIDIA-MIG-GPU/refs/heads/main/images/ScaleDeployOut.gif)
<p align=center>Click <a style="text-decoration:none" target="_blank" rel="noopener" href="https://github.com/rohitralhan/RHOAI-NVIDIA-MIG-GPU/tree/main/images/mig-config-images/deploy-scale">here</a> to see individual images
</font></p>


## Disable the MIG on Red Hat OpenShift

 To utilize the full capacity of the GPU you can always disable/turn off MIG by executing the following command
 ```
MIG_CONFIGURATION=all-disabled && \
  oc label node/$NODE_NAME nvidia.com/mig.config=$MIG_CONFIGURATION --overwrite
 ```
 The MIG manager assigns a `mig.config.state` label to the GPU, then terminates GPU pods to enable/disable MIG mode and configure the GPU according to the specified settings.

## Conclusion
NVIDIA Multi-Instance GPU (MIG) technology enables efficient GPU resource utilization by partitioning a single GPU into multiple independent instances. This ensures optimal performance for diverse workloads, from AI/ML training to inference and data analytics. By leveraging MIG on platforms like Red Hat OpenShift AI, organizations can maximize GPU efficiency, enhance multi-tenancy, and reduce infrastructure costs. Implementing MIG provides a scalable and flexible approach to GPU acceleration, making it a crucial tool for modern AI and high-performance computing environments.

## References
 [MIG on Red Hat OpenShift Container Platform](https://docs.nvidia.com/datacenter/cloud-native/openshift/latest/mig-ocp.html?target=_blank)
 
 [MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/?target=_blank)
 
 [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html?target=_blank)
