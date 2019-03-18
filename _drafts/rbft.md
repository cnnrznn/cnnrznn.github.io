---
layout: single
title:  "Robust Byzantine Fault Tolerance
tags:
  - distributed systems
---

This is a summary of the paper *Making Byzantine Fault Tolerant Systems Tolerate Byzantine Faults*.
The paper addresses the problem of modern BFT solutions being designed for the *benign* case.
As a side-effect, most solutions handle byzantine behaviors poorly.
This paper presents a design principle, *robust BFT (RBFT)* as well as a system, **Aardvark**, that implements it.
