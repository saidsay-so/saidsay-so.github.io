+++
title = "GSoC 2022: Week 5 report"
date = 2022-07-18
draft = false

[taxonomies]
series = ["GSoC 2022"]
tags = ["gsoc", "gsoc2022", "pjdfstest", "rust"]

[extra]
lang = "en"
toc = true
show_comment = false
math = false
series_only = true
mermaid = false
+++

This last week, support for serialized test cases has been added. It is
mainly intended for tests which needs to change user.
Tests for utimensat and posix_fallocate have been implemented, and
preliminary work for chmod has been started, and should be continued
this week.
Sleep duration configuration is now available, allowing to speed up
execution time.
The test runner got improvements too. Now, at the end of its execution,
a basic summary of failed/skipped/passed tests is printed. Filtering
tests by name is now possible. 
