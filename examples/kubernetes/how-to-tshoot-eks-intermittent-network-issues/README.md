# EKS Intermittent Connectivity Issues

---

## Prerequisites

- EKS cluster
- kubectl 
- IMDS access
- S3 bucket
- eksctl
- IAM Policy on worker node to allow push to S3

---
## Warning 

The pods deployed in this sample will have hostNetwork set to true.
A pod can access the network namespace and network resources of the node if it is running in the host network of the node where it is deployed.
In this case, the pod can listen for requests directed to particular addresses which allows users to collect packet captures on a node level from the pod.
Please remove these pods from the cluster as soon as desired data collection completes. 

---

## Usage

When troubleshooting issues related to intermittent connectivity in a cluster that are difficult to reproduce, you can setup a pod that runs tcpdump and streams the packet captures to an S3 bucket for storage and retrieval. This allows users to look into packet captures around the time frame of an issue and analyze the flow of communication to investigate the cause of connectivity drops. 

1. Edit the manifest YAMLs to reflect the S3 bucket name in your account. 
2. You can make changes to the tcpdump command as well to make it more inline with the communication that you're trying to retrieve a packet capture for. Refer : https://linux.die.net/man/8/tcpdump

You would be required to add permissions shared in the manifests section to use the IAM policy required for node to push captures to the S3 bucket. 

--- 

## Getting Started

First, setup a namespace to run the pods in. Use the manifest called namespace.yaml in the manifests folder to do that.  

Next, you would need to setup an S3 bucket or you can choose to use an existing S3 bucket. Should you need to create a new S3 bucket, please refer the best pracatices document below to help setup S3 bucket with server access logging that does NOT have public access enabled : 

- Security best practices for Amazon S3 - https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html

Server access logging provides detailed records of the requests that are made to a bucket. Server access logs can assist you in security and access audits, help you learn about your customer base, and understand your Amazon S3 bill. Further, Amazon EKS control plane logging provides audit and diagnostic logs directly from the Amazon EKS control plane to CloudWatch Logs in your account. These logs make it easy for you to secure and run your clusters. You can select the exact log types you need, and logs are sent as log streams to a group for each Amazon EKS cluster in CloudWatch. Refer to documentation below to get started with EKS Control Plane logging setup : 

- Amazon EKS control plane logging - https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html

To keep analysis targetted, first try to figure out if the connectivity issue happens for a particular worker node, set of worker nodes or randomly anywhere across the cluster. Follow the sections below to work with either of the mentioned scenarios. 

#### Providing correct IAM permissions to stream captures to S3

There's two approach to accomplish this : 

1. Assign IAM policy to worker node role and remove the added permissions once data collection completes
   
   To use this approach, navigate to your AWS IAM console, and create an IAM policy using the s3-tcpdump-policy.json specified in manifests folder. Once you create the IAM policy, assign it to the worker node IAM role. 

2. *Preferred Approach* : Leverage IAM Role for Service Account to assign IAM permissions to the tcpdump pods only
   
   You can run the following command to create the S3 policy file that allows write-only access to an Amazon S3 bucket. If you want to create this S3 policy, copy the following contents to your device. Replace <bucket-to-push-logs-to> with your bucket name and run the command. 

```
cat >s3-tcpdump-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetBucketLocation"
            ],
            "Resource": "arn:aws:s3:::<bucket-to-push-logs-to>"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::<bucket-to-push-logs-to>/*"
        }
    ]
}
EOF
```

Create the IAM policy.

```
aws iam create-policy --policy-name s3-tcpdump-policy --policy-document file://s3-tcpdump-policy.json
```

Create an IAM role and associate it with a Kubernetes service account. You can leverage eksctl to do that for you : 

```
eksctl create iamserviceaccount --name s3-tcpdump-service-account --namespace s3-tcpdump --cluster <my-cluster> --role-name eks-s3-tcpdump --attach-policy-arn arn:aws:iam::111122223333:policy/s3-tcpdump-policy --approve
```


Confirm that the Kubernetes service account is annotated with the role.

```
kubectl describe serviceaccount s3-tcpdump-service-account -n s3-tcpdump
```

#### Create Docker Image

Using the Dockerfile in manifests, setup the container image to capture packet information on the worker node. On your local machine, in the directory where you chose to store the Dockerfile, run the following commands to first build your image : 

```
docker build -t <name-tag> .
```

Once the image build completes, check image on your host using `docker images`. To push your docker image to your ECR repository, follow the steps in this documentation below : 

- Pushing a Docker image - https://docs.aws.amazon.com/AmazonECR/latest/userguide/docker-push-ecr-image.html

---

#### Particular Worker Node

If you've identified that the issue appears often on a particular worker node, then use nodeSelector in deplyment manifest to ensure that you capture packets from that particular node.  
To setup captures, the deployment manifest would need to be edited to add the value for kubernetes.io/hostname to reflect the node name that you want the pod to be scheduled on. 

#### Set of Worker Nodes

If you've identified the issue to be apperaing for a partciluar set of worker nodes, then use affinity along with labelSelector in deplyment manifest for the set of nodes to schedule pods on those particular nodes. 
Reference : https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#types-of-inter-pod-affinity-and-anti-affinity

#### Randomly any Worker Node

In cases where a particular node or set of nodes can not be identified that lead to connectivity issues, the pod could be run as a daemonset to ensure that pcaps are captured from all the nodes and sent to an existing S3 bucket. 

---

#### Daemonset : 

**Note:** Ensure that you update the tcpdump command in the pod spec below in order to target the correct destination. Refer this doc for [examples](https://www.tcpdump.org/manpages/tcpdump.1.html#lbAF). 

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: aws-tcpdump
  name: aws-tcpdump
  namespace: s3-tcpdump
spec:
  selector:
    matchLabels:
      app: aws-tcpdump
  template:
    metadata:
      labels:
        app: aws-tcpdump
    spec:
      automountServiceAccountToken: false
      enableServiceLinks: false
      hostNetwork: true
      securityContext:
        seccompProfile:
          type: RuntimeDefault
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
      containers:
      - image: <docker-image-repo-URI>
        name: aws-tcpdump-aws-cli
        securityContext:
          seccompProfile:
            type: RuntimeDefault
          runAsNonRoot: true
          runAsUser: 1000
          runAsGroup: 1000
          allowPrivilegeEscalation: true
          capabilities: 
            add:
            - NET_RAW
            - NET_ADMIN
            drop:
            - ALL
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        command:
          - sh
          - -c
          - |
            #!/bin/bash
            INSTANCE=$(wget -q -O - http://169.254.169.254/latest/meta-data/instance-id)
            while true; 
            do 
            YEAR=$(date +%Y); 
            MONTH=$(date +%m); 
            DAY=$(date +%d); 
            HOUR=$(date +%H); 
            MINUTE=$(date +%M); 
            tcpdump -i any -W1 -G60 -Z 1000 -w - | aws s3 cp - s3://<your-S3-bucket-name>/tcp-dumps/${INSTANCE}/${YEAR}-${MONTH}-${DAY}-${HOUR}:${MINUTE}-dump.pcap;
            done
```

---

#### Deployment :

**Note:** Ensure that you update the tcpdump command in the pod spec below in order to target the correct destination. Refer this doc for [examples](https://www.tcpdump.org/manpages/tcpdump.1.html#lbAF). 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aws-tcpdump
  namespace: s3-tcpdump
  labels:
    app: aws-tcpdump
spec:
  replicas: 1
  selector:
    matchLabels:
      app: aws-tcpdump
  template:
    metadata:
      labels:
        app: aws-tcpdump
    spec:
      automountServiceAccountToken: false
      enableServiceLinks: false
      serviceAccountName: s3-tcpdump-service-account
      hostNetwork: true
      securityContext:
        seccompProfile:
          type: RuntimeDefault
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - aws-tcpdump
              topologyKey: kubernetes.io/hostname
            weight: 100
      containers:
      - command:
          - sh
          - -c
          - |
            #!/bin/bash
            INSTANCE=$(wget -q -O - http://169.254.169.254/latest/meta-data/instance-id)
            while true; 
            do 
            YEAR=$(date +%Y); 
            MONTH=$(date +%m); 
            DAY=$(date +%d); 
            HOUR=$(date +%H); 
            MINUTE=$(date +%M); 
            tcpdump -i any -W1 -G60 -Z 1000 -w - | aws s3 cp - s3://<your-S3-bucket-name>/tcp-dumps/${INSTANCE}/${YEAR}-${MONTH}-${DAY}-${HOUR}:${MINUTE}-dump.pcap;
            done
        image: <docker-image-repo-URI>
        securityContext:
          seccompProfile:
            type: RuntimeDefault
          runAsNonRoot: true
          runAsUser: 1000
          runAsGroup: 1000
          allowPrivilegeEscalation: true
          capabilities: 
            add:
            - NET_RAW
            - NET_ADMIN
            drop:
            - ALL
        imagePullPolicy: Always
        name: aws-tcpdump-aws-cli
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
      restartPolicy: Always
```

---
## Cleanup

Once the required data is collected analyzed, please clean up the resources created on the EKS cluster and delete the S3 bucket to avoid uncurring additional storage costs over time. 

To clean up deployment, please run : 

```
kubectl delete deployment aws-tcpdump -n s3-tcpdump
```


To clean up daemonset, please run : 

```
kubectl delete daemonset aws-tcpdump -n s3-tcpdump
```

Once the pods are removed from the cluster, delete the service account using : 

```
kubectl delete serviceaccount s3-tcpdump-service-account -n s3-tcpdump
```

And finally, the namespace can be deleted using : 

```
kubectl delete namespace s3-tcpdump
```

To delete S3 bucket, navigate to AWS S3 console. If you provisioned an S3 bucket only to collect tcpdumps, feel free to delete the entire bucket. If you're using an existing S3 bucket, delete the folder `tcp-dumps` to remove all pcap files. 


---

## CI Scan with Checkov

The Kubernetes resource are continuously scanned using [Checkov](https://www.checkov.io/5.Policy%20Index/kubernetes.html) with some checks being skipped. See below:

| Checks | Details | Reasons |
| ---- | ---- | ---- |
| CKV_K8S_19 | Containers should not share the host network namespace | In order to capture packets from a host level for any communication issues to any endpoint, container needs access to host network. |
| CKV_K8S_20 | Containers should not run with allowPrivilegeEscalation | `allowPrivilegeEscalation` in Kubernetes controls whether a process can gain more privileges than its parent process. `tcpdump` requires additional privileges to capture packets on the host.  |

