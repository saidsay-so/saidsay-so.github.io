+++
title = "GSoC 2022: Week 8 report"
date = 2022-08-08
draft = false

[taxonomies]
categories = ["GSoC 2022"]
tags = ["gsoc", "gsoc2022", "pjdfstest", "rust"]

[extra]
lang = "en"
toc = false
show_comment = true
series_only = true
math = false
mermaid = false
+++

The last week, I mostly spent time on improving the test API,
and rewriting non-error tests from rename, unlink, link, truncate and ftruncate.
The bug which prevented using an incomplete config has also been solved.