# terraform-aws-ecs-cluster

[![CircleCI](https://circleci.com/gh/azavea/terraform-aws-ecs-cluster.svg?style=svg)](https://circleci.com/gh/azavea/terraform-aws-ecs-cluster)

A Terraform module to create an Amazon Web Services (AWS) EC2 Container Service (ECS) cluster.

## Table of Contents

* [Usage](#usage)
    * [Auto Scaling](#auto-scaling)
* [Variables](#variables)
* [Outputs](#outputs)

## Usage

This module creates a security group that gets associated with the launch template for the ECS cluster Auto Scaling group. By default, the security group contains no rules. In order for network traffic to flow egress or ingress (including communication with ECS itself), you must associate all of the appropriate `aws_security_group_rule` resources with the `container_instance_security_group_id` module output.

See below for an example.

```hcl
data "template_file" "container_instance_cloud_config" {
  template = "${file("cloud-config/container-instance.yml.tpl")}"

  vars {
    environment = "${var.environment}"
  }
}

module "container_service_cluster" {
  source = "github.com/azavea/terraform-aws-ecs-cluster?ref=3.0.0"

  vpc_id        = "vpc-20f74844"
  ami_id        = "ami-b2df2ca4"
  instance_type = "t2.micro"
  key_name      = "hector"
  cloud_config_content  = "${data.template_file.container_instance_cloud_config.rendered}"

  root_block_device_type = "gp2"
  root_block_device_size = "10"

  health_check_grace_period = "600"
  desired_capacity          = "1"
  min_size                  = "0"
  max_size                  = "1"

  enabled_metrics = [
    "GroupMinSize",
    "GroupMaxSize",
    "GroupDesiredCapacity",
    "GroupInServiceInstances",
    "GroupPendingInstances",
    "GroupStandbyInstances",
    "GroupTerminatingInstances",
    "GroupTotalInstances",
  ]

  subnet_ids = [...]

  project     = "Something"
  environment = "Staging"
}

resource "aws_security_group_rule" "container_instance_http_egress" {
  type        = "egress"
  from_port   = 80
  to_port     = 80
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]

  security_group_id = "${module.container_service_cluster.container_instance_security_group_id}"
}

resource "aws_security_group_rule" "container_instance_https_egress" {
  type        = "egress"
  from_port   = 443
  to_port     = 443
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]

  security_group_id = "${module.container_service_cluster.container_instance_security_group_id}"
}
```

### Auto Scaling

This module creates an Auto Scaling group for the ECS cluster. By default, there are no Auto Scaling policies associated with this group. In order for Auto Scaling to function, you must define `aws_autoscaling_policy` resources and associate them with the `container_instance_autoscaling_group_name` module output.

See this [article](https://segment.com/blog/when-aws-autoscale-doesn-t/) for more information on Auto Scaling, and below for example policies.

```hcl
resource "aws_autoscaling_policy" "container_instance_cpu_reservation" {
  name                   = "asgScalingPolicyCPUReservation"
  autoscaling_group_name = "${module.container_service_cluster.container_instance_autoscaling_group_name}"
  adjustment_type        = "ChangeInCapacity"
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    customized_metric_specification {
      metric_dimension {
        name  = "ClusterName"
        value = "${module.container_service_cluster.name}"
      }

      metric_name = "CPUReservation"
      namespace   = "AWS/ECS"
      statistic   = "Average"
    }

    target_value = "50.0"
  }
}

resource "aws_autoscaling_policy" "container_instance_memory_reservation" {
  name                   = "asgScalingPolicyMemoryReservation"
  autoscaling_group_name = "${module.container_service_cluster.container_instance_autoscaling_group_name}"
  adjustment_type        = "ChangeInCapacity"
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    customized_metric_specification {
      metric_dimension {
        name  = "ClusterName"
        value = "${module.container_service_cluster.name}"
      }

      metric_name = "MemoryReservation"
      namespace   = "AWS/ECS"
      statistic   = "Average"
    }

    target_value = "90.0"
  }
}
```

It's worth noting that the [`aws_autoscaling_policy`](https://www.terraform.io/docs/providers/aws/r/autoscaling_policy.html) documentation suggests we remove `desired_capacity` from the `aws_autoscaling_group` resource when using Auto Scaling. That makes sense, because when it is present, any Terraform plan/apply cycle will reset it.

Unfortunately, removing it from the `aws_autoscaling_group` resource means removing it from the module too.

We will reevaluate things when [Terraform 0.12](https://www.hashicorp.com/blog/terraform-0-12-conditional-operator-improvements) comes out because it promises handling of a `null` `desired_capacity`.


## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|:----:|:-----:|:-----:|
| cloud\_config\_content |  | string | n/a | yes |
| key\_name |  | string | n/a | yes |
| subnet\_ids |  | list | n/a | yes |
| vpc\_id |  | string | n/a | yes |
| ami\_id |  | string | `"ami-6944c513"` | no |
| ami\_owners |  | list | `<list>` | no |
| autoscaling\_group\_name |  | string | `""` | no |
| cloud\_config\_content\_type |  | string | `"text/cloud-config"` | no |
| cluster\_name |  | string | `""` | no |
| cpu\_credit\_specification |  | string | `"standard"` | no |
| desired\_capacity |  | string | `"1"` | no |
| detailed\_monitoring |  | string | `"false"` | no |
| ecs\_for\_ec2\_service\_role\_name |  | string | `""` | no |
| ecs\_service\_role\_name |  | string | `""` | no |
| enabled\_metrics |  | list | `<list>` | no |
| environment |  | string | `"Unknown"` | no |
| health\_check\_grace\_period |  | string | `"600"` | no |
| instance\_type |  | string | `"t2.micro"` | no |
| lookup\_latest\_ami |  | string | `"false"` | no |
| max\_size |  | string | `"1"` | no |
| min\_size |  | string | `"1"` | no |
| project |  | string | `"Unknown"` | no |
| root\_block\_device\_size |  | string | `"8"` | no |
| root\_block\_device\_type |  | string | `"gp2"` | no |
| security\_group\_name |  | string | `""` | no |

## Outputs

| Name | Description |
|------|-------------|
| container\_instance\_autoscaling\_group\_name |  |
| container\_instance\_ecs\_for\_ec2\_service\_role\_arn |  |
| container\_instance\_ecs\_for\_ec2\_service\_role\_name |  |
| container\_instance\_security\_group\_id |  |
| ecs\_service\_role\_arn |  |
| ecs\_service\_role\_name |  |
| id |  |
| name |  |

