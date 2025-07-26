---
title: "Building a database (part 2): Crash resistance"
date: "2025-07-26"
summary: "Small summary"
toc: true
readTime: true
autonumber: true
math: true
katex: true
tags: []
showTags: false
---

In this part we will implement a Write Ahead Log (WAL). The WAL is a mechanism for creating crash resistance and enabling point in time recoveries.

# Crash resistance

