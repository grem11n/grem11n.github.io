---
tile: How to cycle Terraform
date: 2017-03-29
comments: true
layout: post
---
As you can know, Terraform checks your code for cycles. It's when anounced resource triggers a resource from which it was called. There could be a few hops, though.

I must admit that this is very useful check. And resouce cycles are definately a bug in your code. However, I want to show one use case, when a "desired change" ends up with a cycle.

AWS doesn't allow instance to communicate with itself via public address (who ever needs it?) But some software uses such calls for internal health checks. For example, Atlassian products like JIRA do. Therefore, to have a "green" check you have to whitelist Jira's own public IP in it's security group.

Let's say you have everything in Terraform. Most likely you will have an elastic IP for your Jira instance. It isn't a part of infrastructure to replace every day. Moreover, there are still a bunch of things that are very hard to automate.

In order to whitelist that EIP in terraform code you may want to do something like below:

```
resource "aws_security_group" "myGroup" {
  name        = "myGroup"
  description = "My group"
  ingress {
    ...
    cidr_blocks = [
      ...
      "${aws_eip.myInstance_eip.public_ip}/32",
      ...
    ]
  }
}
```

And voila! You have a cycle in Terraform: change in the security group will affect an instnce, change in the instance will affect EIP, which then declared in security group.

```
aws_security_group => aws_instance => aws_eip => aws_security_group
```

Therefore, it's still a solution to just hardcode Elastic IPs of your Atlassian things within a security group :\