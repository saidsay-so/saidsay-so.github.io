+++
title = "Oxidizing FreeBSD's file system test suite"
date = 2022-09-12
draft = true

[taxonomies]
categories = ["GSoC 2022"]
tags = ["gsoc", "gsoc2022", "pjdfstest", "rust"]

[extra]
lang = "en"
toc = true
show_comment = true
math = false
mermaid = true
+++

> Your proposal Rewrite PJDFSTest suite has been accepted by Org The FreeBSD Project for GSoC 2022.

So, here we are! My proposal for the Google Summer of Code has been accepted,
and so start a long journey to rewrite this test suite!
But what this project is about, why rewrite it in Rust anyway?

<!-- more -->

In this post, I will answer to these questions,
by explaining what is the project,
its current shortcomings and how rewriting it in Rust has solved them.

## Thanks

These introductory words are to thank my mentor [Alan Somers](https://github.com/asomers),
for accepting the proposal, helping and guiding me through this journey!
His high availability despite his other obligations really helped me to do this project!

## Code

The code is available on [GitHub](https://github.com/musikid/pjdfstest).

## What is pjdfstest?

First, [pjdfstest](https://github.com/pjd/pjdfstest) is a test suite for POSIX system calls.
It checks that they behave correctly according to the specification, e.g. return the right errors,
change time metadata when appropriated, succeed when they should, etc.
It is particularly useful to test file systems, and mainly used to this effect.
In fact, it was originally written to validate the ZFS port to FreeBSD,
but it now supports multiple operating systems and file systems, while still 
primarily used by the FreeBSD project to test against UFS and ZFS file systems.
Its tests are written in Shell script and relies on the TAP protocol to be executed, 
with a C component to use syscalls from shell script, 
which we are going to see more in detail in the next section.

{% tip() %}
[TAP](https://testanything.org/) is a simple protocol for test results.
{% end %}

{% info() %}
Even if it's targeted towards POSIX syscalls,
some non-standardized syscalls like `chflags(2)` are tested as well.
{% end %}

### Architecture

After this little introduction, let's explore how its current architecture is built.
Like explained earlier, the main language for the test suite is Shell script.
This is a high-level language, available on multiple platforms
and with a simple syntax if we consider the POSIX-compatible subset.
However, it is impossible to rely solely on it to test syscalls.
Even if most syscalls have binaries counterparts,
what should be tested is their implementations.

#### `pjdfstest.c`

This is where the `pjdfstest` program comes useful.
It acts as a thin layer between syscalls and shell, by providing a 
command-line interface similar to how these syscalls should be called in C.
It returns the result on the standard output, 
whether that be a formatted one like for the `stat(2)` output,
an error string when a syscall fails, or simply 0 when all went well.
For example, to `unlink` a file: `./pjdfstest unlink path` which should return 0 if the file
has been successfully deleted, or `ENOENT` for example if it doesn't exist.

{% mermaid() %}
flowchart LR
PR{"TAP harness\n prove"} === TC{"Test case\n truncate/00.t"} --- E["Assertion\n expect 0 truncate path 1234567"] 
<==> PJD{pjdfstest binary} <==> S[/"Syscall\n truncate(&quot;path&quot;, 1234567)"/]
{% end %}


### Test case

The test cases are grouped in folders named after the syscall being tested and contain assertions.
An assertion is simply a call to the `expect` bash function, which takes as parameters the expected output
and the arguments to be forwarded to the `pjdfstest` binary. 
The binary then executes the syscall and send its result back to the `expect` function,
which compares the output with what's expected,
and fails when it should.

The typical shape of a test case is:

- Description
- Feature support
- TAP plan
- Assertions

Example (`tests/chflags/11.t`):

```bash
#!/bin/sh
# vim: filetype=sh noexpandtab ts=8 sw=8
# $FreeBSD: head/tools/regression/pjdfstest/tests/chflags/11.t 211352 2010-08-15 21:24:17Z pjd $

# Description

desc="chflags returns EPERM if a user tries to set or remove the SF_SNAPSHOT flag"

dir=`dirname $0`
. ${dir}/../misc.sh

# Check feature support

require chflags_SF_SNAPSHOT

# TAP plan

echo "1..145"

# Assertions

n0=`namegen`
n1=`namegen`
n2=`namegen`

expect 0 mkdir ${n0} 0755
cdir=`pwd`
cd ${n0}

for type in regular dir fifo block char socket symlink; do
	if [ "${type}" != "symlink" ]; then
		create_file ${type} ${n1}
		expect EPERM -u 65534 -g 65534 chflags ${n1} SF_SNAPSHOT
		expect none stat ${n1} flags
		expect EPERM chflags ${n1} SF_SNAPSHOT
		expect none stat ${n1} flags
		expect 0 chown ${n1} 65534 65534
		expect EPERM -u 65534 -g 65534 chflags ${n1} SF_SNAPSHOT
		expect none stat ${n1} flags
		expect EPERM chflags ${n1} SF_SNAPSHOT
		expect none stat ${n1} flags
		if [ "${type}" = "dir" ]; then
			expect 0 rmdir ${n1}
		else
			expect 0 unlink ${n1}
		fi
	fi
```

## Limitations

This architecture has its advantages, 
like easier reading for the tests or quick modifications,
but several limitations also arise from it.

 <!-- * Cannot be run as an unprivileged user. -->
 <!-- * No ATF integration.  This isn't critical, but it sure would be nice if pjdfstest could display detailed results at https://ci.freebsd.org/job/FreeBSD-main-amd64-test/lastCompletedBuild/testReport/ . -->

### Configurability

Some features (like `chflags(2)`) aren't systematically implemented on file systems,
often because they aren't part of the POSIX specification.
To account this, the test suite has a concept of features.
However, those were hard coded in the test suite
and consequently could not be easily configured,
like with a configuration file or the command-line.

### Isolation

A lot of assertions are written in a single file.
This is due to TAP which encourages large tests
instead of small isolated ones, but also because it's harder
to split and refactor a shell script. 

Given the lack of isolation between these assertions,
it is more difficult to debug errors.

### Debugging

As a consequence of large test files,
it is difficult to understand what went wrong in case of failure.
Also, writing tests in shell script made them easy to read
and quick to modify,
but paradoxically harder to debug, because there's no sh debugger.
The TAP-based approach doesn't help because the number used to validate
an assertion is disconnected from any context.

### Duplication

Because it's written in shell, DRY (Don't Repeat Yourself) is difficult to achieve.
This leads to a lot of duplication between the tests,
with only a few lines changed between the files.
This is particularly visible on the error tests,
which share an enormous amount of code and yet are not 
factored out.

### TAP plan

Each time a file is modified, its TAP plan needs to be recomputed.
I don't think that I really need further elaboration...

### Performance

Performance is also one of the shortcomings.
Because it's written in shell script, 
which relies on external programs to execute most of its operations,
it takes almost 5 minutes to complete the suite, when it could be way less.
One of the reasons is that it runs all test cases sequentially, 
when it's actually possible to run them independently 
given that test files do not rely on each other. 
It also contains many 1-second sleeps, which could be smaller 
if it had better configurability.

## Rewrite it in Rust

With this rewrite, we aim to solve those limitations.
Other languages have been considered, namely Python and Go.
However, since we had a strong preference for Rust,
the test suite got oxidized!

In addition to solving the previously stated shortcomings,
we want to write a Rust binary, which:

- is self-contained (embed all the tests),
- can execute the tests (with the test runner),
- can filter them according to configuration and conditions.

Another goal is being able to run the test suite on FreeBSD's CI infrastructure
and get better reports with ATF metadata support.

### Test collection

The first thing we had to think on is how to collect the tests.
Since there isn't a `pytest`-like package in Rust, we had to investigate
on existing approaches.
Since the result should be a binary, techniques related to `cargo test` were excluded.
We wanted to go first with the standard Rust `#[test]` attribute to collect the functions,
but the API is not public.
An experimental [implementation](https://github.com/rust-lang/rust/issues/50297)
is available on the Nightly channel,
but relying on Nightly for a test suite was quite ironic,
in addition of limiting the.
So, instead, we implemented an approach similar to Criterion's one,
where a test group (`criterion_group!`) is declared with the test
cases. However, it introduced a lot of boilerplate, 
and we finally decided to use the [`inventory`](https://github.com/dtolnay/inventory)
crate.
With it, we can collect the tests without having to declare a test group.
A test can now be declared with the `crate::test_case!` macro,
which collects a description while allowing parameterization of the test,
to require preconditions or features, or over file types.

### Writing tests

We now know how to collect the tests, but we still have to write them!
Again, we investigated on the interface which could fit better here.
We already knew that we wanted to write them like how
Rust unit tests usually are.
This means using the standard assertion macros (`assert!`/`assert_eq!`/`assert_ne!`)
and `unwrap`ing `Result`s.
That also means that the test functions would panic when something goes wrong.


```rust
crate::test_case! {
    /// truncate should extend a file, and shrink a sparse file
    extend_file_shrink_sparse
}
fn extend_file_shrink_sparse(ctx: &mut TestContext) {
    let file = ctx.create(FileType::Regular).unwrap();
    let size = 1234567;
    assert!(truncate(&file, size).is_ok());

    let actual_size = lstat(&file).unwrap().st_size;
    assert_eq!(actual_size, size);

    let size = 567;
    assert!(truncate(&file, size).is_ok());

    let actual_size = lstat(&file).unwrap().st_size;
    assert_eq!(actual_size, size);
}
```

Now, the assertions are grouped inside a test function,
which allows filtering tests with improved granularity.
It also improves reporting, now that assertions are in test functions with meaningful names (instead of a number), 
and open the possibility to run them in parallel.

### Debug

Because the rewrite is in Rust, standard debuggers (in particular `lldb`) can be used.
It is also easier to trace syscalls with `strace`,
now that a single test function can be executed.

### Performance

#### TL;DR 10% faster and more to expect with parallelization!

From a non-accurate benchmark that I did over the `chmod/00.t` test case,
the results look encouraging.
While the original took 13s, the rewrite took 1 second less, 
and this time is essentially spent on `sleep` calls!
It could be greatly improved by adding parallelization (and by splitting test functions per file type in this particular case).

Benchmark command:
```sh
[root@meow rust]# hyperfine "./target/debug/pjdfs_runner" "prove -q ../tests/chmod/00.t"
Benchmark 1: ./target/debug/pjdfs_runner
  Time (mean ± σ):     12.009 s ±  0.008 s    [User: 0.003 s, System: 0.002 s]
  Range (min … max):   12.005 s … 12.032 s    10 runs
 
  Warning: Statistical outliers were detected. Consider re-running this benchmark on a quiet PC without any interferences from other programs. It might help to use the '--warmup' or '--prepare' options.
 
Benchmark 2: prove -q ../tests/chmod/00.t
  Time (mean ± σ):     13.167 s ±  0.035 s    [User: 0.792 s, System: 0.507 s]
  Range (min … max):   13.121 s … 13.239 s    10 runs
 
Summary
  './target/debug/pjdfs_runner' ran
    1.10 ± 0.00 times faster than 'prove -q ../tests/chmod/00.t'
```

## Looking forward

### Progress status

### ATF support

### Parallelization

### Features clarification

### Command-line improvements

file types filter, syscall filter