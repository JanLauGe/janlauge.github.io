---
layout: post
comments: true
title: Building Our Own Open Source Supercomputer with R and AWS
date: 2018-12-27 18:00:00
categories: [DataScience, MachineLearning, CloudComputing, R]
tags: [DataScience, MachineLearning, CloudComputing, R]
excerpt_separator: <!--more-->
---
**This post will demonstrate how to build a scaleable computing cluster
on AWS. This will enable you to run hundreds or thousands of models in a
short amount of time. We will rely completely on R and open source
R packages.**

<!--more-->

## Introduction

An ever-increasing number of businesses is moving to the cloud and using
platforms such as [Amazon Web Services](https://aws.amazon.com/)(AWS)
for their data infrastructure. This is convenient for
Data Scientists like myself because it means that my knowledge from previous
jobs becomes much more applicable to a new role and I can hit the ground
running.

Lately I have become very excited about the
[`future`](https://cran.r-project.org/web/packages/future/vignettes/future-1-overview.html)
package and how it makes the scalig of computational tasks easy and intuitive.
# give a summary of future here
I wanted to see what we could do with `future` and other open source R packages
such as [`ssh`](https://ropensci.org/technotes/2018/06/12/ssh-02/)
by [rOpenSci](https://ropensci.org/), and
[`remoter`](https://cran.r-project.org/web/packages/remoter/vignettes/remoter.pdf),
another brand new package by Drew Schmidt.

The basic idea:
* use R and AWS to spin up our own cloud compute cluster
* log in to the head node and define a computationally expensive task
* farm out this task to a number of worker nodes in our cluster
* do all of this WITHOUT having to switch between RStudio, RStudioServer, the command line, the AWS console, etc.

Why do I care about the last point? Well, Data Science is a science and should
rely on the [Scientific Method](https://en.wikipedia.org/wiki/Scientific_method).
One core component of the Scientific Method is reproducibility, and one of the
best ways to keep your Data Science workflow reproducible is to write code that
can run start to finish without any user intervention. This also allows for
greater applicability in the future because you can re-use your previous data
product or service in the next project without having to re-do any manual steps.
Don't just take my word for it, here is another great Hadley
Wickham video in which he stresses the same point:

<iframe width="560" height="315" src="https://www.youtube.com/embed/cpbtcsGE0OA" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

So without further ado, let's get started implementing the bullet point list!


// A diagram showing the Setup


## Preparation

There are a few basic requirements that need to be in place:
1. an active AWS account.
1. an AMI with `R`, `remoter`, `tidyverse`, and `future` installed.
1. a working `ssh` key pair on your local machine that allows you to ssh into your ec2 instances.

Detailed instructions on how to fulfil these basic requirements are beyond the
scope of this post. You can find more information in the articles linked below.
* [What Is Amazon EC2?](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html)
* [Running R on AWS](https://aws.amazon.com/blogs/big-data/running-r-on-aws/)
* [The CloudyR project](http://cloudyr.github.io/)


# Setup

Load the required packages. Also make sure your AWS access credentials are set.
I do this using `Sys.setenv`. There is other ways but I found that this works
best for me. We also specify the AMI ID (that's the system configuration that
we are going to use for our instance), and the instance type (this is a good
[overview](https://aws.amazon.com/ec2/instance-types/); I am using
`t2.micro` because it is free). If you have any problems with this step, double-
check that the region set in `Sys.setenv` matches the region of your AMI.

```r
library(aws.ec2)
library(ssh)
library(remoter)
library(tidyverse)

# set access credentials
aws_access <- aws.signature::locate_credentials()
Sys.setenv(
  "AWS_ACCESS_KEY_ID" = aws_access$key,
  "AWS_SECRET_ACCESS_KEY" = aws_access$secret,
  "AWS_DEFAULT_REGION" = aws_access$region
)

# set parameters
aws_ami <- "ami-06485bfe40a86470d"
aws_describe <- describe_images(aws_ami)
aws_type <- "t2.micro"
```

Ready for launch!

```r
ec2inst <- run_instances(
  image = aws_ami,
  type = aws_type)

# wait for boot, then refresh description
Sys.sleep(10)
ec2inst <- describe_instances(ec2inst)

# get IP address of the instance
ec2inst_ip <- get_instance_public_ip(ec2inst)
```

The instance should be running and we can connect to it via ssh.
That works, but personally I'd prefer to stay in RStudio instead of switching
to the command line


```r
# CMD string to start remoter::server on instance
r_cmd_start_remoter <-
  "sudo Rscript -e 'remoter::server(port = 55555, showmsg = TRUE)'"

# connect and execute
plan(multicore)
x <- future(
  ssh_exec_wait(
    session = con,
    command = r_cmd_start_remoter))
```




And the amazing bit: all of this took me about a day to set up from scratch!
Show me how to do THAT in Python ;)


discussion of this functionality in aws.ec2 issues
https://github.com/cloudyr/aws.ec2/issues/30

example in same repo on how it could be implemented
https://github.com/cloudyr/aws.ec2/issues/39

nice walk through of R and AWS
https://www.daeconomist.com/post/2018-06-27-aws/
