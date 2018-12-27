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







For ssh problems:
https://help.ubuntu.com/community/SSH/OpenSSH/Keys#Troubleshooting
[root@SERVER~#] mkdir /etc/ssh/user1
[root@SERVER~#] cp /home/user1/.ssh/authorized_keys /etc/ssh/user1/
[root@SERVER~#] chown -R user1:user1 /etc/ssh/user1
[root@SERVER~#] chmod 755 /etc/ssh/user1
[root@SERVER~#] chmod 644 /etc/ssh/user1/authorized_keys
[root@SERVER~#] vi /etc/ssh/sshd_config

#RSAAuthentication yes
#PubkeyAuthentication yes
# changed .ssh/authorized_keys to /etc/ssh/user1/authorized_keys <<<<<<<
AuthorizedKeysFile      /etc/ssh/user1/authorized_keys
#AuthorizedKeysCommand none
#AuthorizedKeysCommandRunAs nobody



And the amazing bit: all of this took me about a day to set up from scratch!
Show me how to do THAT in Python ;)


discussion of this functionality in aws.ec2 issues
https://github.com/cloudyr/aws.ec2/issues/30

example in same repo on how it could be implemented
https://github.com/cloudyr/aws.ec2/issues/39

nice walk through of R and AWS
https://www.daeconomist.com/post/2018-06-27-aws/
