# terraform-ec2srcdst

## DEPRECATED

**This module is deprecated**: Due to the cumbersome dependencies needed to setup this module properly, we decided it was not worth it to have all this code in a module, and we set things up together with the autoscaling group instead. More info can be found [in this pull request](https://github.com/skyscrapers/terraform-ec2srcdst/pull/2).

## Description

Terraform module to disable the EC2 source destination check in an autoscaling group using lifecycle hooks and Lambda.

To keep things simple, this module uses autogenerated names for most of the resources it creates. To be able to better identify such resources, you can provide extra tags via the `var.tags` variable, which will be appended to all resources that support tagging.

This module will create the following:

- A Lambda function to disable the source destination check for all launched instances
  - The Lambda function is limited to only the provided autoscaling groups via IAM
  - The Lambda function has broader permissions with `ec2:ModifyInstanceAttribute`, as that action can't be locked down to specific resources
- A CloudWatch log group for the Lambda function
- An IAM role for the lambda function
- All the glue to bind the Autoscaling group lifecycle hooks with the Lambda function

**Note** that you still need to create the autoscaling lifecycle hook with the correct name (`disable-srcdstcheck`). Ideally you should create it via the `initial_lifecycle_hook` attribute in the [`aws_autoscaling_group`](https://www.terraform.io/docs/providers/aws/r/autoscaling_group.html) resource(s), otherwise the hook won't trigger on the initial Terraform run. You can use the output `autoscaling_group_initial_lifecycle_hook` of this module to feed the `initial_lifecycle_hook` or `aws_autoscaling_lifecycle_hook` attributes. For example:

```tf
resource "aws_autoscaling_group" "foobar" {
  availability_zones   = ["us-west-2a"]
  name                 = "terraform-test-foobar5"
  health_check_type    = "EC2"
  termination_policies = ["OldestInstance"]

  initial_lifecycle_hook = ["${module.disable_srcdstcheck.autoscaling_group_initial_lifecycle_hook}"]

  tag {
    key                 = "Foo"
    value               = "foo-bar"
    propagate_at_launch = true
  }
}
```

Also note that the `initial_lifecycle_hook` argument won't create the lifecycle hook on existing autoscaling groups, you'll have to do that manually or via the [`aws_autoscaling_lifecycle_hook`](https://www.terraform.io/docs/providers/aws/r/autoscaling_lifecycle_hooks.html) resource.

## Variables

| Name | Description | Type | Default | Required |
|------|-------------|:----:|:-----:|:-----:|
| autoscaling\_group\_names | List of autoscaling group names to attach the lambda function to | list | n/a | yes |
| lambda\_log\_retention\_in\_days | Specifies the number of days you want to retain log events in the lambda function log group | string | `"30"` | no |
| tags | Map with additional tags to add to created resources | map | `<map>` | no |

## Outputs

| Name | Description |
|------|-------------|
| autoscaling\_group\_initial\_lifecycle\_hook | Configuration block to add to the `initial_lifecycle_hook` argument on the autoscaling groups |
| cloudwatch\_event\_target\_id | The unique target assignment ID for the CloudWatch event target |
| lambda\_cloudwatch\_log\_group\_name | Name of the created Cloudwatch log group for the Lambda function |
| lambda\_function\_name | Name of the created Lambda function |

## Example

```tf
module "disable_srcdstcheck" {
  source                  = "github.com/skyscrapers/terraform-ec2srcdst"
  autoscaling_group_names = ["some-autoscaling-group-name", "some-other-autoscaling-group-name"]

  tags = {
    foo = "bar"
  }
}
```
