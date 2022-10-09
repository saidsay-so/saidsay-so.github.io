+++
title = "GSoC 2022: Week 3 report"
date = 2022-07-04
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

I was expecting to work on the new interface, but I had to solve some
issues, namely the return type of the test functions and the
configuration file.

Now, the test functions do not have to return a `Result`, and instead can
unwrap the errors, which cause a panic which is captured by the test
runner. They also can use the standard assertion macros (`assert!` and
`assert_eq!`). All of this provide a more idiomatic way to write test
functions.

As for the configuration file, it will allow to execute opt-in tests,
for example `posix_fallocate`, which is available on FreeBSD, but doesn't
work on all file systems.

Finally, my mentor @asomers has implemented a way to collect the tests
automatically, by using the inventory crate, without having to manually
create a test group and a test case.
I originally suggested to use this crate (or linkme) to collect the
tests, but we were worried of how it could be hard to implement,
besides the fact that there were some [concerns] at the time on the
underlying mecanism of these crates.

[concerns]: https://github.com/rust-lang/rust/issues/47384#issuecomment-1022913071
