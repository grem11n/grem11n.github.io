---
title: 'Terraform tips: EBS attachements'
featured: images/blog/terraform.png
date: 2017-03-11
comments: true
layout: post
---

A few things you should know about managing instances with attached EBS volumes with Terraform.

Terraform is a great tool for managing your cloud infrastructure. It supports AWS, Azure, OpenStack and other major vendors. Also, it has providers for many more other things like DNS, databases, VCS, etc. However, sometimes it could be tricky to operate linked resources, especially, when you want to change only one of them.

Let's say, you have an AWS instance, with some provisioning. Terraform configuration would be pretty simple:

*I omit some code to keep listings short. Attributes and their values rely on what you expect from your infrastructure. You can find a lot of examples in [official docs](https://www.terraform.io/docs/)*

```
resource "aws_instance" "my_instacne" {

  /* Your instance attributes */

  /* Here comes provisioning part */
  /* Let's say it is a simple scrit */

  provisioner "remote-exec" {
    inline = [
      "bootstrap.sh ${join(" ", aws_instance.cluster.*.private_ip)}"
    ]
  }
}
```

It's pretty easy and would work like a charm with a lot of use cases. But let's attach an EBS volume to our instance. The configuration would be like this:

```
resource "aws_instance" "my_instacne" {

  /* Your instance attributes */

  /* Here comes provisioning part */
  /* Let's say it is a simple scrit */

  provisioner "remote-exec" {
    inline = [
      "bootstrap.sh ${join(" ", aws_instance.cluster.*.private_ip)}"
    ]
  }
}

resource "aws_ebs_volume" "my_disk" {

  /* EBS volume attributes here */

}

resource "aws_volume_attachment" "bitbucket_ebs_att" {

  /* Here we actually attaching volume to the instance */

}
```

Notice, that an instance itself, EBS drive and "attachment process" are different resources from Terraform's perspective. It means that you may face a situation when your provisioning starts before the volume is actually attached. And what if one of provisioning steps is LVM creation? It's a problem.

A solution advised by Hashicorp is so called "null resource". It gives you an ability to move provisioning out of "instance resource" and Terraform is smart enough to wait apply "null resource" at the end of installation.

Null resource has "triggers" option, which serves as a conditional case determining whether apply this resource or not. From a first glance, it's obvious to trigger null resource with AWS instance. But, in this case, it would be triggered regardless you spinning up a new host or taking the whole cluster down. In second scenario execution of a "null resource" will fail, causing Terraform failure. Moreover, this resource will remain tainted causing more issues before it would be successfully executed.

To avoid such situation, you can trigger provisioning with "aws_volume_attachment". By default, Terraform will trigger this resource on both start and destroy procedures. So, you need to add "skip_destroy" attribute.

[Official documentation](https://www.terraform.io/docs/providers/aws/r/volume_attachment.html#skip_destroy) says:

*skip_destroy - (Optional, Boolean) Set this to true if you do not wish to detach the volume from the instance to which it is attached at destroy time, and instead just remove the attachment from Terraform state. This is useful when destroying an instance which has volumes created by some other means attached.*

Which is not really clear in my opinion and it doesn't cover all use cases.

Final configuration will be like below:

```
resource "aws_instance" "my_instacne" {

  /* Your instance attributes */

}

resource "null_resource" "cluster" {

  triggers {
    volume_attachment = "${join(",", aws_volume_attachment.*.id)}"
  }

  connection {
    host = "${element(aws_instance.my_instance.*.public_ip, 0)}"
  }

  provisioner "remote-exec" {
    inline = [
      "bootstrap-cluster.sh ${join(" ", aws_instance.cluster.*.private_ip)}"
    ]
  }
}

resource "aws_ebs_volume" "my_disk" {

  /* EBS volume attributes here */

}

resource "aws_volume_attachment" "bitbucket_ebs_att" {

  /* Here we actually attaching volume to the instance */

  skip_destroy = true

}
```

Now you will be able to start and stop instances without triggering provisioner in vain. An attachment will be destroyed automatically by AWS, hence re-created on the next instance start with Terraform.

Good luck!

