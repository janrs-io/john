---
title: "What Is a CPU Really? Why Can It Perform Math?"
author: "John Yang"
lang: en
pubDatetime: 2021-09-22T02:37:00Z
tags:
  - Architecture
  - CS-Fundamentals
description: "An intuitive walkthrough connecting electrical switches, Boolean truth tables, and Claude Shannon's digital logic to explain how CPUs compute."
---

What exactly is a CPU, and how does it execute mathematical operations?

## Simple Physical Circuits

Start with a familiar physics circuit diagram:

![CPU circuit diagram](../../../assets/images/posts/img-4a72b3a9836d.png)

The diagram shows three elementary circuits: series, parallel, and an inverter (drawn informally rather than with strict standard symbols).

Switches `a` and `b` control whether electricity reaches the light bulb. Using `1` for **on/lit** and `0` for **off/dark**, each circuit produces a predictable input-output table:

![Series circuit outcome table](../../../assets/images/posts/img-8b3a8c1d66dd.png)

![Parallel circuit outcome table](../../../assets/images/posts/img-4af9b492a4a4.png)

![Inverter circuit outcome table](../../../assets/images/posts/img-7a3d41447861.png)

Simple as they look, these three tables represent the physical bedrock of modern computers. A modern CPU is built by composing billions of tiny switches acting according to rules just like these.

***

## George Boole

British mathematician **George Boole** introduced **set operations** and **Boolean logic**. The three core set operations—union, intersection, and complement—map directly to logical operators used in code every day:

![Set theory operations](../../../assets/images/posts/img-d559164d268b.png)

- Union -> `OR`
- Intersection -> `AND`
- Complement -> `NOT`

In Boolean logic, values are restricted to `0` and `1`. Their truth tables look like this:

![OR truth table](../../../assets/images/posts/img-858fa78eb09a.png)

![AND truth table](../../../assets/images/posts/img-92cff7238ecd.png)

![NOT truth table](../../../assets/images/posts/img-ec6d9b6c1e03.png)

Compare these Boolean truth tables with the three electrical circuit tables above—**they match row for row**. Mathematics and physics meet right here, giving electrical circuits the ability to evaluate logical expressions.

## Claude Shannon

In 1932, **Claude Shannon** studied George Boole's theory at the University of Michigan and later worked on MIT's mechanical **Differential Analyzer**. Shannon realized that electronic switches (`1`/`0`), binary numbers (`1`/`0`), and Boolean truth tables are all manifestations of the same algebra, founding the field of **Digital Circuit Design** and showing how electronic switches can execute logic and arithmetic.

***

## Scaling from Binary Addition to Complex Computation

Once hardware circuits can add binary digits, complex mathematical operations—from square roots and trigonometry to calculus and Fourier transforms—can be broken down numerically into repeated additions and logical operations. Hardware circuit design turns those mathematical rules into high-speed execution.

## Summary

A **CPU** is fundamentally a vast collection of carefully engineered switching circuits that perform computation by executing Boolean logic and binary arithmetic in silicon.
