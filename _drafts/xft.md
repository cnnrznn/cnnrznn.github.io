---
layout: single
title:  "XFT: Fault Tolerance for the Real World
tags:
  - distributed sytems
  - paper reviews
---

Typically, protocols for distributed systems can be classified as either _crash fault tolerant_ (CFT) or *byzantine fault tolerant* (BFT).
In the real world, CFT is unattainable due to software or hardware faults.
However, BFT is too expensive, requiring $n \gt 3t$ nodes where $t$ is the upper-bound number of faulty nodes.
The authors of this paper observe that the assumptions made for BFT protocols are too strong for a real-world attacker.
Namely, an attacker may not have the ability to create arbitrary partitions or network schedules.
This paper presents a technique that achieves _higher resilience_ than CFT while achieiving _higher availability_ than BFT by exploiting these more realistic assumptions.

## Assumptions
* $n$ replicas
* Replicas connected by reliable point-to-point communication channel
* Clients and replicas form an all-to-all graph
* *Network fault* - inability of correct processes to communicate within some $\delta$
* *Anarchy* $t_{nc} + t_c + t_p \gt t$ and $t_{nc} \gt 0$
* $t \leq \lfloor \frac{n-1}{2} \rfloor$

## Algorithm

### Overview
XPaxos operates in a sequence of _views_.
In each view, the protocol proceeds in _normal_ mode, accepting and commiting transactions from clients.
In each view, $t+1$ replicas form a *synchronous group* where transactions are *actively* replicated within the group, and *lazily* replicated on replicas outside the group.
If a fault is detected within the group, a *view change* is initiated.

During a view change, a new synchronous group is chosen, and replicas from the old group transfer data about their *commit log* to members of the new group.

## Conclusion
