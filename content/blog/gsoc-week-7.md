+++
title = "GSoC 2022: Week 7 report"
date = 2022-08-02
draft = false

[taxonomies]
categories = ["GSoC 2022"]
tags = ["gsoc", "gsoc2022", "pjdfstest", "rust"]

[extra]
lang = "en"
toc = false
show_comment = true
math = false
mermaid = false
+++

The last week, some features have been implemented for the test runner,
the most important one being the support for running tests without
being privileged (root user). It is now possible to run tests which
don't require privileges and the runner will automatically skips those
which need them (to create block/char files, or switch users for
example). The bug which prevented automatic clean up of test files has
been fixed.

As for the tests, tests for ETXTBSY have been written, and new
functions have been added to the other error tests.
Now, the rewrite supports more than 50% of the tests from the original
test suite!
The API for switching users is also being reworked, 
to limit the number of users needed.

Finally, the documentation now includes the supported command-line
arguments and includes more information for the test suite users.