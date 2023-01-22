# Amazon EKS auto scaling with Karpenter

## Creating an EKS cluster

Create an EKS cluster with no managed node groups (MNG). Karpenter will be used for provisioning the worker nodes rather than using MNG. Fargate will be used for hosting Karpenter (solving the chicken/egg problem)

```
export CLUSTER_NAME="<cluster-name>"
export AWS_DEFAULT_REGION="<region>"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
```

```
eksctl create cluster -f - << EOF
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_DEFAULT_REGION}
  version: "1.23"
  tags:
    karpenter.sh/discovery: ${CLUSTER_NAME}
fargateProfiles:
  - name: karpenter
    selectors:
    - namespace: karpenter
iam:
  withOIDC: true
EOF

export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.endpoint" --output text)"
```



## Installing Karpenter

1. Set an environment variable with the Karpenter version you want to install
```
export KARPENTER_VERSION=v0.22.1
```

2. Provision the relevant infrastrucutre and IAM roles using Cloudformation

```
TEMPOUT=$(mktemp)

curl -fsSL https://karpenter.sh/"${KARPENTER_VERSION}"/getting-started/getting-started-with-eksctl/cloudformation.yaml  > $TEMPOUT \
&& aws cloudformation deploy \
  --stack-name "Karpenter-${CLUSTER_NAME}" \
  --template-file "${TEMPOUT}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${CLUSTER_NAME}"
```

3. Add the Karpenter node role to your aws-auth configmap, allowing nodes to connect

```
eksctl create iamidentitymapping \
  --username system:node:{{EC2PrivateDNSName}} \
  --cluster "${CLUSTER_NAME}" \
  --arn "arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}" \
  --group system:bootstrappers \
  --group system:nodes
```

4. Create an AWS IAM Role, Kubernetes service account, and associate them using IRSA

```
eksctl create iamserviceaccount \
  --cluster "${CLUSTER_NAME}" --name karpenter --namespace karpenter \
  --role-name "${CLUSTER_NAME}-karpenter" \
  --attach-policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME}" \
  --role-only \
  --approve

export KARPENTER_IAM_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${CLUSTER_NAME}-karpenter"
```

5. Create the EC2 Spot Service Linked Role

```
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true
# If the role has already been successfully created, you will see:
# An error occurred (InvalidInput) when calling the CreateServiceLinkedRole operation: Service role name AWSServiceRoleForEC2Spot has been taken in this account, please try a different suffix.
```

6. Use Helm to deploy Karpenter to the cluster
```
docker logout public.ecr.aws
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version ${KARPENTER_VERSION} --namespace karpenter --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=${KARPENTER_IAM_ROLE_ARN} \
  --set settings.aws.clusterName=${CLUSTER_NAME} \
  --set settings.aws.clusterEndpoint=${CLUSTER_ENDPOINT} \
  --set settings.aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-${CLUSTER_NAME} \
  --set settings.aws.interruptionQueueName=${CLUSTER_NAME} \
  --wait
```

## Creating Karpenter Provisioners

Create a default Provisioner, based on on-demand instances. Replace the subnet IDs below with the IDs of the private subnets of the cluster created above. Change the limits for the max CPU/memory spinned up by the provisioner based on your requirements to control costs

```
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["on-demand"]
    - key: kubernetes.io/arch
      operator: In
      values: ["amd64"]
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  consolidation:
    enabled: true
  providerRef:
    name: default
---
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default
spec:
  subnetSelector:
    aws-ids: "subnet-066137093347bb9c9,subnet-09e1b5a9bf2d600e2,subnet-0d5c53524a34a0ff9"
  securityGroupSelector:
    aws-ids: "sg-0598ff5a1cec39829"
```

```
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: spot
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["on-demand","spot"]
    - key: kubernetes.io/arch
      operator: In
      values: ["amd64"]
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  consolidation:
    enabled: true
  labels:
    purchase-option: spot
  taints:
  - effect: NoSchedule
    key: spot
  providerRef:
    name: default
```

## Deploying a sample app into on-demand instances

1. Create `Deployment` for the sample app

```
kubectl apply -f - << EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
  namespace: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - image: us.gcr.io/k8s-artifacts-prod/hpa-example
        name: hpa-example
        resources:
          requests:
            cpu: 200m
EOF
```

2. Create `HorizontalPodAutoscaler` for automatically scaling out/in the sample app, based on CPU utilisation

```
kubectl apply -f - << EOF
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: php-apache
spec:
  maxReplicas: 10
  metrics:
  - resource:
      name: cpu
      target:
        averageUtilization: 50
        type: Utilization
    type: Resource
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
EOF
```

3. Generate load on the application
```
kubectl run -i -n php-apache --tty load-generator --image=busybox /bin/sh
```

```
while true; do wget -q -O - http://php-apache; done
```

4. Keep the for loop above running, and check the instances provisioned by Karpenter via the EC2 console or by executing the following command:
```
watch -n 2 kubectl get node
```

5. Exit the for loop, and clean up by executing the following commands:
```
kubectl delete pod load-generator -n php-apache
```

```
kubectl delete hpa php-apache -n php-apache
```


```
kubectl delete deployment php-apache -n php-apache
```

Check the instances provisioned by Karpenter via the EC2 console -- the on-demand instances should be reduced to 1 (instances are still required for other pods that are still running in the cluster)



## Deploying a sample app into Spot instances

1. Create `Deployment` for the sample app. Add the required `nodeSelector` and `tolerations` for the app to get deployed into Spot instances, rather than on-demand instances

```
kubectl apply -f - << EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
  namespace: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      nodeSelector:
        purchase-option: spot
      tolerations:
      - key: spot
        operator: "Exists"
        effect: NoSchedule
      containers:
      - image: us.gcr.io/k8s-artifacts-prod/hpa-example
        name: hpa-example
        resources:
          requests:
            cpu: 200m
EOF
```

2. Create `HorizontalPodAutoscaler` for automatically scaling out/in the sample app, based on CPU utilisation

```
kubectl apply -f - << EOF
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: php-apache
spec:
  maxReplicas: 10
  metrics:
  - resource:
      name: cpu
      target:
        averageUtilization: 50
        type: Utilization
    type: Resource
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
EOF
```

3. Generate load on the application

```
kubectl run -i -n php-apache --tty load-generator --image=busybox /bin/sh
```

```
while true; do wget -q -O - http://php-apache; done
```

4. Keep the for loop above running, and check the instances provisioned by Karpenter via the EC2 console or by executing the following command:
```
watch -n 2 kubectl get node
```

5. Exit the for loop, and clean up by executing the following commands:

```
kubectl delete pod load-generator -n php-apache
```

```
kubectl delete hpa php-apache -n php-apache
```


```
kubectl delete deployment php-apache -n php-apache
```

Check the instances provisioned by Karpenter via the EC2 console -- all the spot instances provisioned by Karpenter should be terminated after few seconds

