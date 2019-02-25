---
layout: post
comments: true
title: Building Our Own Open Source Supercomputer with R and AWS
description: Part 2 of 2 demonstrating how to use cloudyr/aws.ec2, rOpenSci/ssh,
  and remoter to launch AWS EC2 instances and connect to them directly from RStudio.
image: /assets/aws_ssh_symbol.png
date: 2019-02-03 18:00:00
categories: [DataScience, MachineLearning, CloudComputing, R]
tags: [DataScience, MachineLearning, CloudComputing, R]
excerpt_separator: <!--more-->
---
**How to build a scaleable computing cluster on AWS and run hundreds or
thousands of models in a short amount of time. We will completely rely on R and
open source R packages. This is post 2 out of 2.**

<!--more-->

## Introduction

In a [previous post]({{ site.url }}//2019/building-our-own-open-source-supercomputer-with-R/)
I outlined how to get started with R and EC2 instances using the coudyR aws.ec2
package. It's time to take it to the next level and create our own little
"supercomputer", i.e. a cluster of instances connected and working together to
speed up our data science workflow.

As mentioned before, this works well for "embarrassingly parallel" problems.
However, nothing about them is embarrassing so don't be afraid to put your
hand up and admit that this is highly applicable to your workflow.

To remind you where we left of last time:
* we created an Amazone Machine Image (ami) with R, a few packages, and our ssh keys
* we started a new ec2 instance from that image
* we used `shh` to connect and start `remoter` on the instance

In this second post we will launch a number of additional instances
and then distribute a computational task between the instances.


# Example Dataset
