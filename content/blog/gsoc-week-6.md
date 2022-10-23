+++
title = "GSoC 2022: Week 6 report"
date = 2022-07-29
draft = false

[taxonomies]
series = ["GSoC 2022"]
tags = ["gsoc", "gsoc2022", "pjdfstest", "rust"]

[extra]
lang = "en"
toc = true
show_comment = false
series_only = true
math = false
mermaid = false
+++

This last week, not a lot of features have been implemented. Support
for creating file in CWD by default or an optional provided path has
been added to the test runner.
However, a lot of tests concerning errors have been added. Tests now
include ELOOP, ENOTDIR, EINVAL, EISDIR, ENAMETOOLONG, EEXIST, EACCES
and ENOENT.