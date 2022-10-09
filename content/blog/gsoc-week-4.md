+++
title = "GSoC 2022: Week 4 report"
date = 2022-07-11
draft = false

[taxonomies]
categories = ["GSoC 2022"]
tags = ["gsoc", "gsoc2022", "pjdfstest", "rust"]

[extra]
lang = "en"
toc = true
show_comment = false
series_only = true
math = false
mermaid = false
+++

A lot have been done this last week.
A simpler interface to declare test cases and also provides automatic test collection has been implemented, 
originally with [`linkme`](https://github.com/dtolnay/linkme) 
but a linkage problem on FreeBSD 14 forced us to switch for [`inventory`](https://github.com/dtolnay/inventory) instead.
It provides parameterization over file types, file system specific features and file flags.
The configuration file has also been improved, 
especially with more granularity for tests requiring file system exclusive features or file flags.
The test runner has got improvements too. 
It skips tests which don't satisfy requirements and print the required features to launch these tests.
