---
layout:     post
title:      基础设施即代码，使用Terraform 和 cdktf 管理基础设施
subtitle:   基础设施即代码，使用Terraform 和 cdktf 管理基础设施
date:       2023-02-10
author:     ica10888
catalog: true
tags:
    - terraform
    - docker
---


# 基础设施即代码，使用Terraform 和 cdktf 管理基础设施

基础设施即代码(Infrastructure-as-Code，IaC)，意思是使用代码来管理和审查基础设施的合理性，在通过完善的CICD流程和代码审查工作流的帮助下，可以支持Devops在基础设施方向上的延伸，不仅仅可以通过代码基础设施，完完整整地实现多租户上的copy，保留完整且正确的服务器配置，实现业务无论是在腾讯云，阿里云，还是海外的AWS，Azure，谷歌云上的复制和迁移。更重要的是，避免了配置偏移的可能性和人为操作的错误，节省劳动力，方便review和同步。

### terraform 

terraform 是声明式语言（Declarative Language），首先需要配置文件

 main.tf

``` shell
terraform {
  required_providers {
    tencentcloud = {
      source = "tencentcloudstack/tencentcloud"
    }
  }
}

provider "tencentcloud" {
  secret_id  = "xxxxx"
  secret_key = "xxxxx"
  region     = "ap-guangzhou"
}
```

 tke.tf

```shell
# 创建 TKE 集群
resource "tencentcloud_kubernetes_cluster" "tke_test" {
  vpc_id                                     = "vpc-4s39are5"
  cluster_version                            = "1.18.4"
  cluster_cidr                               = "172.16.0.0/16"
  cluster_max_pod_num                        = 64
  cluster_name                               = "test-1"
  cluster_desc                               = "created by terraform"
  cluster_max_service_num                    = 2048
  cluster_internet                           = true
  managed_cluster_internet_security_policies = ["0.0.0.0/0"]
  cluster_deploy_type                        = "MANAGED_CLUSTER"
  cluster_os                                 = "tlinux2.4x86_64"
  container_runtime                          = "containerd"
  deletion_protection                        = false

  worker_config {
    instance_name              = "some-node"
    availability_zone          = "ap-guangzhou-4"
    instance_type              = "S5.MEDIUM2"
    system_disk_type           = "CLOUD_SSD"
    system_disk_size           = 50
    internet_charge_type       = "TRAFFIC_POSTPAID_BY_HOUR"
    internet_max_bandwidth_out = 1
    public_ip_assigned         = true
    subnet_id                  = "subnet-541d6xtq"
    security_group_ids         = ["sg-8dvl87xh"]
    enhanced_security_service = false
    enhanced_monitor_service  = false
    password                  = "Pass@123"
  }

}
```

使用命令

``` shell
terraform init
terraform apply -auto-approve
```

### cdktf

cdktf同样是CLI commands ，cdktf的使用场景是 tf 文件无法满足业务逻辑的时候，需要使用代码实现部分逻辑时可以通过cdktf实现

使用命令

```shell
brew install cdktf #需要注意如果无法安装成功，可以通过brew edit修改makefile
mkdir cdktf-demo
# 然后在该目录下初始化代码
cdktf init #我选择的是4 python-pip (居然没有golang实现...)
```

然后修改cdktf.json,这里使用了docker和tencentcloud的Providers

```json
{
  "language": "python",
  "app": "python3 ./main.py",
  "terraformProviders": ["aws@~> 2.0","kreuzwerker/docker@~> 3.0","tencentcloudstack/tencentcloud@~> 1.61.10"],
  "codeMakerOutput": "imports",
  "context": {
    "excludeStackIdFromLogicalIds": "true",
"allowSepCharsInLogicalIds": "true"
  }
}
```

使用命令，生成imports里面的 Generate Code

``` shell
cdktf get
```

##### docker

``` py
#!/usr/bin/env python
from constructs import Construct
from cdktf import App, TerraformStack
from imports.docker import Image
from imports.docker import Container
from imports.docker import DockerProvider

class MyStack(TerraformStack):
    def __init__(self, scope: Construct, ns: str):
        super().__init__(scope, ns)

        DockerProvider(self, 'docker')

        docker_image = Image(self, 'nginxImage',
            name='nginx:latest',
            keep_locally=False)

        Container(self, 'nginxContainer',
            name='tutorial',
            image=docker_image.name,
            ports=[{
                'internal': 80,
                'external': 8000
            }])

app = App()
MyStack(app, "learn-cdktf-docker")

app.synth()

```

  docker实现

##### tke

```py
#!/usr/bin/env python
from constructs import Construct
from cdktf import App, TerraformStack
from imports.tencentcloud import KubernetesClusterConfig
from imports.tencentcloud import KubernetesClusterWorkerConfig
from imports.tencentcloud import TencentcloudProvider

class MyStack(TerraformStack):
    def __init__(self, scope: Construct, ns: str):
        super().__init__(scope, ns)

        TencentcloudProvider(self,'tencentcloud',
            region="ap-guangzhou",
            secret_id="xxxxx",
            secret_key="xxxxx")

        kubernetes_cluster_worker_config= KubernetesClusterWorkerConfig(
            instance_name="some-node",
            availability_zone="ap-guangzhou-4",
            instance_type="S5.MEDIUM2",
            system_disk_type="CLOUD_SSD",
            system_disk_size=50,
            internet_charge_type="TRAFFIC_POSTPAID_BY_HOUR",
            internet_max_bandwidth_out=1,
            public_ip_assigned=True,
            subnet_id="subnet-541d6xtq",
            security_group_ids=["sg-8dvl87xh"],
            enhanced_security_service=False,
            enhanced_monitor_service=False,
            password="Pass@123")

        KubernetesClusterConfig(
            vpc_id="vpc-4s39are5",
            cluster_version="1.18.4",
            cluster_cidr="172.16.0.0/16",
            cluster_max_pod_num=64,
            cluster_name="test-1",
            cluster_desc="created by cdktf",
            cluster_max_service_num=2048,
            cluster_internet=True,
            managed_cluster_internet_security_policies=["0.0.0.0/0"],
            cluster_deploy_type="MANAGED_CLUSTER",
            cluster_os="tlinux2.4x86_64",
            container_runtime="containerd",
            deletion_protection=False,
            worker_config=[kubernetes_cluster_worker_config])

app = App()
MyStack(app, "learn-cdktf-docker")

app.synth()

```

 tke实现

使用下面命令部署

``` shell
cdktf deploy
```

### 参考文档

[introduction-terraform](https://lonegunmanb.github.io/introduction-terraform/)

[CDK for Terraform](https://developer.hashicorp.com/terraform/cdktf)
