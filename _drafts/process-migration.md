---
layout: single
title:  "A Summary of Process Migration"
tags:
  - distributed systems
---

Process migrations is the act of moving a running program from one host operating system to another.
Process migration has many modern permutations, such as VM migration, container migration, and live migration.
In this article, I will give an overview of the different kinds of process migration and the challenges one must face in order to implement a process migration system.

## Terminology and Subtlties
Migration occurs between two *hosts*, operating systems.
These hosts can be physical or virtual; both cases are handled in the same way.

All forms of migration can be reduced to *process migration*, because a system for solving one can be applied to all three.
These forms include "raw" process migration, *VM migration*, and *container migration*.
The distinction between the three is necessary because each comes with a semi-unique set of challenges.

During migration, the application must experience *downtime*.
This is the period in which the process is not being executed on either host.
It is still an open research questions whether non-downtime process migration is possible.

## General Challenges
To migrate a process, one must (1) pause the process, (2) send its state to the destination host, then (3) resume the process.
In general, optimizations to the process migration problem occur in (1) and (3), and correctness is ensured by (2).
Here, I discuss stage (2).

Migrating a process is complicated; there are many pieces of state that a linux process relies on.

* Memory mappings
* CPU registers
* File descriptors
  - Regular files
  - Inet sockets
  - Unix sockets

## The 3 Musketeers

## Optimization - "Live" Migration

## True Live Migration

## Conclusion
