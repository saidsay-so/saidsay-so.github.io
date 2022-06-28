+++
title = "GSoC 2022: Week 1 report"
date = 2022-06-20
draft = false

[taxonomies]
categories = ["GSoC 2022"]
tags = ["gsoc", "gsoc2022", "pjdfstest", "rust"]

[extra]
lang = "en"
toc = true
show_comment = false
math = false
mermaid = false
+++

GSoC official coding period has started the previous week.

So far, we have the building blocks for the project.
The runner is in a very basic state but, it works.
We also have test collection, through the `pjdfs_test_case` and
`pjdfs_group` macros.
The documentation, while being a big work-in-progress,
is available [here](https://musikid.github.io/pjdfstest/).
And we have the first tests written, for the `chmod` syscall.
