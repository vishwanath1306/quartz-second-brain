---
title: Working Set Estimation
layout: post
date: 2023-03-28
tags:
    - working set
    - workload tracking
enableToc: true
---

## Why even do working set estimation

For the [zero Copy Cache](running_zero_copy_cache.md) to work effectively, there needs to be a way to understand the parts of the workloads which are accessed more often than others. 

## Basic algo