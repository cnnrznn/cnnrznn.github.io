---
layout: single
title:  "GoMR revisited"
---

A few months ago, I quit my job.
Since then one of my goals was to finish this long-standing project.
Originally, GoMR was hacked together to collect data for a tech-talk style presentation.
Because of this, many corners were cut that I believe prevented the system from being considered production ready.

I am proud to say I have blown up the repository and rebuilt GoMR from its foundation.
The project is in a place I feel comfortable leaving it.
I believe the software has a clean architecture, and could be made production ready if I had more time and resources.

In this article, I want to discuss what was wrong with the old system, and what is right with this one.
I want to talk about software architecture, seperation of concerns, and problem isolation.
I want to talk about resiliency and fault tolerance.
I want to talk about simplicity.

# The old

To give my original talk on the GoMR system, I wanted to collect data comparing Spark to GoMR.
And, to really flex, I wanted to collect data from the system running on a cluster, not just a single node.
In the end, this led to the distributed version of my system being coupled - not highly, completely - to Kubernetes.
This coupling meant that the GoMR repository was doing two jobs simultaneously: resource management and data processing.
For the new version, I only wanted to focus on the latter.
I don't want to control how people deploy and run each executable instance beyond reasonable basic assumptions.

For those interested, the code lives under a tag at [v0.2](https://github.com/cnnrznn/gomr/releases/tag/v0.2).

# What it does, and what it doesn't

# Developer experience

# Architecture

## Tiers
