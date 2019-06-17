# EKS Workshop 动手实验说明

本文档目的是指导用户熟悉建立Amazon EKS集群的过程，并利用AWS相关的服务和插件部署一套常见应用程序。

部署流程如下：

## 1. 准备工作
### a. 请确保您有海外AWS账号和足够的资源配额
- 请确保您有海外AWS账号的IAM user，该用户有创建相关资源的权限。
### b. 准备跳板机所需要的VPC
- 可以使用默认的VPC或者创建新的VPC, 可以参考[这个文档](https://docs.aws.amazon.com/zh_cn/vpc/latest/userguide/VPC_Scenario1.html#VPC_Scenario1_Implementation)创建新的VPC。

## 2. 创建跳板机
请点击如下链接在各AWS区域创建跳板机：

[Virginia](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?stackName=eks-jumpbox&templateURL=https://s3-ap-southeast-1.amazonaws.com/whe-pub/eks-workshop/jumpbox-eks.yaml)

[Ohio](https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/quickcreate?stackName=eks-jumpbox&templateURL=https://s3-ap-southeast-1.amazonaws.com/whe-pub/eks-workshop/jumpbox-eks.yaml)

[N California](https://console.aws.amazon.com/cloudformation/home?region=us-west-1#/stacks/quickcreate?stackName=eks-jumpbox&templateURL=https://s3-ap-southeast-1.amazonaws.com/whe-pub/eks-workshop/jumpbox-eks.yaml)

[Oregon](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/quickcreate?stackName=eks-jumpbox&templateURL=https://s3-ap-southeast-1.amazonaws.com/whe-pub/eks-workshop/jumpbox-eks.yaml)

[Mumbai](https://console.aws.amazon.com/cloudformation/home?region=ap-south-1#/stacks/quickcreate?stackName=eks-jumpbox&templateURL=https://s3-ap-southeast-1.amazonaws.com/whe-pub/eks-workshop/jumpbox-eks.yaml)

[Seoul](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-2#/stacks/quickcreate?stackName=eks-jumpbox&templateURL=https://s3-ap-southeast-1.amazonaws.com/whe-pub/eks-workshop/jumpbox-eks.yaml)

[Singapore](https://console.aws.amazon.com/cloudformation/home?region=ap-southeast-1#/stacks/quickcreate?stackName=eks-jumpbox&templateURL=https://s3-ap-southeast-1.amazonaws.com/whe-pub/eks-workshop/jumpbox-eks.yaml)

[Sydney](https://console.aws.amazon.com/cloudformation/home?region=ap-southeast-2#/stacks/quickcreate?stackName=eks-jumpbox&templateURL=https://s3-ap-southeast-1.amazonaws.com/whe-pub/eks-workshop/jumpbox-eks.yaml)

[Tokyo](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/quickcreate?stackName=eks-jumpbox&templateURL=https://s3-ap-southeast-1.amazonaws.com/whe-pub/eks-workshop/jumpbox-eks.yaml)

[Montreal](https://console.aws.amazon.com/cloudformation/home?region=ca-central-1#/stacks/quickcreate?stackName=eks-jumpbox&templateURL=https://s3-ap-southeast-1.amazonaws.com/whe-pub/eks-workshop/jumpbox-eks.yaml)

[Frankfurt](https://console.aws.amazon.com/cloudformation/home?region=eu-central-1#/stacks/quickcreate?stackName=eks-jumpbox&templateURL=https://s3-ap-southeast-1.amazonaws.com/whe-pub/eks-workshop/jumpbox-eks.yaml)

[Ireland](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/quickcreate?stackName=eks-jumpbox&templateURL=https://s3-ap-southeast-1.amazonaws.com/whe-pub/eks-workshop/jumpbox-eks.yaml)

[London](https://console.aws.amazon.com/cloudformation/home?region=eu-west-2#/stacks/quickcreate?stackName=eks-jumpbox&templateURL=https://s3-ap-southeast-1.amazonaws.com/whe-pub/eks-workshop/jumpbox-eks.yaml)

[Paris](https://console.aws.amazon.com/cloudformation/home?region=eu-west-3#/stacks/quickcreate?stackName=eks-jumpbox&templateURL=https://s3-ap-southeast-1.amazonaws.com/whe-pub/eks-workshop/jumpbox-eks.yaml)

[Sao Paulo](https://console.aws.amazon.com/cloudformation/home?region=sa-east-1#/stacks/quickcreate?stackName=eks-jumpbox&templateURL=https://s3-ap-southeast-1.amazonaws.com/whe-pub/eks-workshop/jumpbox-eks.yaml)

在Quick create stack页面填入相应参数：
- **InstanceType**: 跳板机实例类型，可以保持默认参数。
- **SubnetId**: 跳板机所在的子网，私有子网或公有子网都可以，如果是私有子网，请确保有NAT网关或NAT实例可访问外网。
- **VpcId**: 跳板机所在VPC，请确保SubnetId是在VpcId所在的VPC内。

勾选 "我确认，AWS CloudFormation 可能创建 IAM 资源"
![](./images/cfn-confirm.png)
约5分钟后，CloudFormation堆栈创建完成:
![](./images/cfn-complete.png)


## 3. 登录跳板机创建EKS集群
- 登录跳板机的Web SSH控制台，控制台的URL可以从CloudFormation的输出界面中找到：
![](./images/cfn-ip.png)
- 创建eksctl config文件。在此配置文件中，node group 使用了mixed instances nodegroup，按需实例和竞价实例搭配使用。更多信息请参考此[文档](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/spot-fleet.html)
  ```
  cd ~
  REGION=`curl -sS http://169.254.169.254/latest/dynamic/instance-identity/document | jq -r .region`
  
  cat <<EOF >>eks-config.yaml
  apiVersion: eksctl.io/v1alpha5
  kind: ClusterConfig

  metadata:
    name: eks-workshop
    region: $REGION

  nodeGroups:
    - name: ng-1
      minSize: 3
      maxSize: 5
      privateNetworking: true
      ssh:
        publicKeyName: whe
      iam:
        attachPolicyARNs:
          - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
          - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
          - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
        withAddonPolicies:
          autoScaler: true
          albIngress: true
      instancesDistribution:
        instanceTypes: ["t3.small", "t3.medium"]
        onDemandBaseCapacity: 0
        onDemandPercentageAboveBaseCapacity: 50
        spotInstancePools: 2
  EOF
  ```
- 运行eksctl创建新的EKS集群：
  ```
  eksctl create cluster -f ./eks-config.yaml
  ```

  更多参数可参考[eksctl官方文档](https://eksctl.io/)。
  
  所有资源创建可能需要15分钟左右。
  
## 4. 创建示例应用程序 2048小游戏
在本部分中，您将创建一个2048小游戏来测试新集群。
  ```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.0.1/docs/examples/2048/2048-namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.0.1/docs/examples/2048/2048-deployment.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.0.1/docs/examples/2048/2048-service.yaml
  ```

## 5. 创建AWS ALB Ingress controller
### a. 部署 ALB ingress controller
部署 AWS ALB Ingress controller所需的RBAC Roles 和 RoleBindings:
  ```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.0.1/docs/examples/rbac-role.yaml
  ```
下载 AWS ALB Ingress controller YAML:
```bash
curl -sS "https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.0.1/docs/examples/alb-ingress-controller.yaml" > alb-ingress-controller.yaml
```
修改 –cluster-name 为上述步骤创建的EKS集群名字:
![](./images/alb-ingress-controller.png)

部署ingress controller:
```bash
kubectl apply -f alb-ingress-controller.yaml
```

### b. 部署 ingress
下载ingress YAML:
```bash
curl -sS https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.0.1/docs/examples/2048/2048-ingress.yaml -o 2048-ingress.yaml
```
编辑 `2048-ingress.yaml` 文件, 在annotations中增加 `alb.ingress.kubernetes.io/target-type: ip`. 本示例中我们将用`ip mode` 作为目标类型，这样ALB的流量将会直接转发到Pod上。
![](./images/alb-ip.png)
然后创建ingress
```bash
kubectl apply -f 2048-ingress.yaml
```

获取ingress资源的`Address`属性：
```bash
kubectl -n 2048-game describe ing/2048-ingress
```
![](./images/alb-address.png)

### c. 用浏览器打开上一步的ALB地址，开始您的2048游戏之旅！
![](./images/2048.png)

## 6. 清理环境
### a. 删除ingress
  ```bash
  kubectl delete -f 2048-ingress.yaml
  ```
### b. 删除EKS集群
  ```bash
  eksctl delete cluster eks-workshop --region $REGION
  ```
### c. 删除跳板机CloudFormation stack