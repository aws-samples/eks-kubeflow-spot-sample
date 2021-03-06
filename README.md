# Running Kubeflow on EKS with mixed of Spot GPU instances
Amazon EC2 P3 instances deliver high performance compute in the cloud with NVIDIA® V100 Tensor Core GPUs. These instances offer up to one petaflop of mixed-precision performance per instance to significantly accelerate machine learning(ML) and high-performance computing (HPC) applications. However, the cost of a P3 is ten times the price per core of a general-purpose instance (M5) or compute-optimized (C5) instance. Amazon EC2 Spot instances allow you to request spare Amazon EC2 computing capacity for up to 90% off the On-Demand price. However, EC2 Spot instances can get terminated anytime within a 2-minute notification. Such termination requires special handling compared to On-Demand. e.g., handling the interruption or failover to On-Demand in the case there is no enough Spot capacity for the application to complete.  

This example demonstrates how to use Spot instances for running ML and HPC application like Spark powered by Kubeflow hosted on EKS. The example masks from the data scientist the compute allocation semantics. i.e., it favors Spot for the ML or HPC application and seamlessly failover to On-Demand with minimal disruption to the customer application. 

## EKS cluster setup
* [Install aws-CLI, eksctl, and kubctl tools](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html). 

* Deploy the cluster with a spot-based node group (ASG) to host the application control-plane, Kubeflow. 

`eksctl create cluster -f=specs/cluster.yaml` 

* Deploy P3 Spot-based mixed instances node-group.

`eksctl create cluster --config-file=specs/p3spot.yml`

* Deploy P3 On-Demand-based mixed instances node-group.

`eksctl create cluster --config-file=specs/p3od.yml`

* Deploy NVIDIA device plugin for Kubernetes.

After your GPU worker nodes join the cluster, you must apply the NVIDIA device plugin for Kubernetes as a DaemonSet on your cluster with the following command.

`kubectl apply -f specs/nvidia-device-plugin.yml`

You will need a single daemon set for both (or all) GPU-based node groups. Make sure the instances you used are set under `nodeAffinity` e.g.,

```yaml
    affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                - key: beta.kubernetes.io/instance-type
                  operator: In
                  values:
                    - p3.2xlarge
                    - p3.4xlarge
                    - p3.8xlarge
```

more on [gpu-ami](https://docs.aws.amazon.com/eks/latest/userguide/gpu-ami.html)

* Deploy cluster-autoscaler

Discover the GPU node-groups
```bash
aws autoscaling describe-auto-scaling-groups|jq '.AutoScalingGroups[].AutoScalingGroupName'
"eksctl-ai-us-west-2-nodegroup-m5spot-NodeGroup-E16WIJDMWB83"
"eksctl-ai-us-west-2-nodegroup-p3od-NodeGroup-NKHMN74TFOPF"
"eksctl-ai-us-west-2-nodegroup-p3spot-NodeGroup-1EC86FRMZ7V6N"
```
In our example, the GPU nodes groups are `p3od` and `p3spot`

Edit the [specs/cluster-autoscaler-multi-asg.yaml](specs/cluster-autoscaler-multi-asg.yaml).Search for `cluster-autoscaler-priority-expander` config map and set the On-Demand GPU nodegroup(`p3od`) lower priority than the Spot GPU node-group(`p3spot`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-priority-expander
  namespace: kube-system
data:
  priorities: |-
    10:
      - .*-non-existing-entry.*
    20:
      - eksctl-ai-us-west-2-nodegroup-p3od-NodeGroup-NKHMN74TFOPF
    60:
      - eksctl-ai-us-west-2-nodegroup-p3spot-NodeGroup-1EC86FRMZ7V6N
```

Also in the Pod spec section set the expander to use priority option with the GPU node-groups. It this example allow up to 7 Spot GPU instances and up to 3 On-Demand GPU instances.

```yaml
    command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=priority
        - --nodes=0:7:eksctl-ai-us-west-2-nodegroup-p3spot-NodeGroup-1EC86FRMZ7V6N
        - --nodes=0:3:eksctl-ai-us-west-2-nodegroup-p3od-NodeGroup-NKHMN74TFOPF
```

## Deploy kubeflow

* It is recommended to set the Kubeflow dashboard with authentication enabled to avoid malicious user exploitation. Before deploying Kubeflow, create a domain name, mint a certificate, and store in AWS Certificate Manager (ACM). 
Capture the certificate ARN.

* In AWS Cognito, create user pool and capture the following: cognitoAppClientId, cognitoUserPoolArn, cognitoUserPoolDomain

* Follow the Kubeflow [install guide](https://www.kubeflow.org/docs/aws/deploy/install-kubeflow/). Use the example in [specs/kubeflow/ai-us-west-2](specs/kubeflow/ai-us-west-2) for the deployment.
The Kubeflow workload is deployed on the node-group specified in [cluster.yml](specs/cluster.yaml), `m5spot`.

* Enable [Authentication and Authorization](https://www.kubeflow.org/docs/aws/authentication/)



By now you have a cluster kubeflow config like in [specs/kubeflow/](specs/kubeflow/) with generated files like `aws_config` and `kustomize`

## Build and deploy custom Jupyter notebook on Spot instances
The directory [/images](images) includes two main images. The first, spot-sig-handler-image powers a daemon set that runs on every Spot instance and "listens" to Spot interruptions. 

```bash
POLL_INTERVAL=${POLL_INTERVAL:-5}
NOTICE_URL=${NOTICE_URL:-http://169.254.169.254/latest/meta-data/spot/termination-time}

while http_status=$(curl -o /dev/null -w '%{http_code}' -sL ${NOTICE_URL}); [ ${http_status} -ne 200 ]; do
  echo $(date): ${http_status}
  sleep ${POLL_INTERVAL}
done
```

Upon interruption, a node is being drained.
```bash
kubectl drain ${NODE_NAME} --force --ignore-daemonsets
```
 i.e., no new workload is scheduled.

In case one wishes to capture interruption rates, we suggest using SQS to store interruption cases. For that one need to populate the queue name in [specs/region-config.yaml](specs/region-config.yaml), build the image and deploy the daemon-set.

```bash
cd images/spot-sig-handler-image/
./build.sh

cd ../../
kubectl apply -f specs/spot-sig-handler-ds.yaml
```
Every Spot instance will be monitored by now, so every interruption is logged in SQS, and every impacted pod will receive a SIGTERM signal. 

3/ Build and deploy the customer Jupyter notebook

```bash
cd images/jupyter-pyspark-image/
./build.sh
```

Change default image list in kubeflow dashboard by

```bash
kubectl edit cm jupyter-web-app-config -n kubeflow
```

```yaml
data:
 spawner_ui_config.yaml: |
 # (ellipsis)
 spawnerFormDefaults:
 image:
 # (ellipsis)
 options:
 - gcr.io/kubeflow-images-public/tensorflow-1.15.2-notebook-cpu:1.0.0
 - gcr.io/kubeflow-images-public/tensorflow-1.15.2-notebook-gpu:1.0.0
 - gcr.io/kubeflow-images-public/tensorflow-2.1.0-notebook-cpu:1.0.0
 - gcr.io/kubeflow-images-public/tensorflow-2.1.0-notebook-gpu:1.0.0
 # you can add your image tag HERE like
 - some-registry.io/yahavb/jupyter-spark:v1.0
```
Restart a pod labeled “app.kubernetes.io/name=jupyter-web-app”, which reloads 1. configuration.

```bash
kubectl delete po -l app.kubernetes.io/name=jupyter-web-app -n kubeflow
```

## Data preparation with Jupyter PySpark example notebook.

Using the Kubeflow dashboard, start the PySpark example notebook. We will begin to a massive Spark job and observe EKS auto-scale the spark workload across GPU Spot instances, and failover to GPU On-Demand as Spot capacity is no longer available with no modification.  

The sample notebook includes java and python options. Both are equivalent and start a driver pod in the say namespace `zip`. The driver will allocate `spark.kubernetes.executor.request.cores` cores per executer and launch `spark.executor.instances`. When the executers ends, the dirver pod will remain in `Complete` state. 

```
%%bash

/opt/spark-2.4.6/bin/spark-submit --master "k8s://https://kubernetes.default.svc:443" \
--deploy-mode cluster \
--name spark-python-pi \
--conf spark.executor.instances=50 \
--conf spark.kubernetes.container.image=seedjeffwan/spark-py:v2.4.6 \
--conf spark.kubernetes.driver.pod.name=spark-python-pi-driver \
--conf spark.kubernetes.namespace=zip \
--conf spark.kubernetes.driver.annotation.sidecar.istio.io/inject=false \
--conf spark.kubernetes.executor.annotation.sidecar.istio.io/inject=false \
--conf spark.kubernetes.pyspark.pythonVersion=3 \
--conf spark.kubernetes.executor.request.cores=4 \
--conf spark.kubernetes.authenticate.driver.serviceAccountName=spark /opt/spark/examples/src/main/python/pi.py 128000
```

## Monitoring

* Enable [Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/deploy-container-insights-EKS.html). 

* Enable detailed CloudWatch monitoring for the two GPU-based Auto Scaling Metrics. 

* Create a CloudWatch dashboard that features three Stacked area graphs. 

    - ContainerInsights•pod_cpu_utilization•ClusterName: ai-us-west-2

    - ContainerInsights•cluster_node_count•ClusterName: ai-us-west-2

    - Auto Scaling•GroupDesiredCapacity•AutoScalingGroupName for each node-group

## Results

The upper graph shows the overall CPU used by the Spark executers. The middle graph depicts the number of nodes (EC2 Instances) that started upon the need for CPU. The bottom graph depicts the distribution of nodes between Spot and On-Demand. We can see that the Spot node-group `p3spot` picks the load first and when it reached its capacity [7](https://github.com/yahavb/eks-kubeflow-spot-sample/blob/d2f13be6d86d4932802da98ef9669a58dc533495/specs/cluster-autoscaler-multi-asg.yaml#L170) and from there the On-Demand `p3od` node group.

![Auto Scale](auto-scale.png)
