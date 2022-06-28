+++
title = "Rewrite it in Rust: pjdfstest"
date = 2022-06-20
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

So, here we are! My proposal has been accepted, I'm so relieved!
But what this project is about, why rewrite it in Rust anyway?

<!-- more -->

In this post, I will answer to these questions,
by explaining what is the project,
its current shortcomings and how rewriting it in Rust would solve them.

{% info() %}
The proposal is available [here](https://github.com/musikid/gsoc/).
{% end %}

## What is pjdfstest?

First of all, [pjdfstest](https://github.com/pjd/pjdfstest) is a test suite for POSIX system calls.
It is particularly useful to test file systems, and mainly used to this effect.
In fact, it was originally written to validate the ZFS port to FreeBSD,
but it now supports multiple operating systems and file systems.


### Architecture

{% mermaid() %}
flowchart LR
PR{"prove\n(TAP harness)"} === TC{"Test case\n(TAP producer)"} ==> E["Assertion\n`expect` bash function"] ==> PJD{pjdfstest binary} ==> S[/Syscall/]
PJD ==> E
{% end %}

After this little introduction, let's explore how its current architecture is built.
The entry point for testing is `prove`, a TAP test harness.
It will execute the test cases,
and produce a report from the output of these cases.

{% info() %}
Any TAP harness would work, but it's simpler to just consider `prove`.
{% end %}

{% tip() %}
More info about the TAP protocol [here](https://en.wikipedia.org/wiki/Test_Anything_Protocol).
{% end %}

#### Test case

These cases are grouped in folders named after the syscall being tested.
They are made up of assertions,
using `expect` bash function, which calls the `pjdfstest` binary. 
The binary then executes the syscall and send its result back to the `expect` function,
which compares the expected and actual outputs,
and fail when it has to.

The typical shape of a test case is:

- Description
- Check feature support
- TAP plan for the case
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

### Limitations

This architecture has its advantages, but several limitations also arise from those points.

Since the test suite relies on TAP,
a lot of assertions are written in a single file.
The report is also fairly limited because of the protocol.
And with files that have more than 1000 assertions, debugging become difficult.

Talking of debuggability, the fact that it's written in shell makes it easier to write tests, 
but also increase the difficulty to debug and review them, 
given the lack of shell debugger and isolation between the assertions.

Performance is also one of the shortcomings of using shell.
Because it relies on external programs to execute most of the operations,
it will be a lot slower than a compiled (or even an interpreted) test suite.

## Rewrite it in Rust

With this rewrite, we aim to solve those limitations.

### Structure

Test cases are grouped by syscalls.
Each syscall folder then have a `mod.rs` file, 
which contains declarations of the test cases inside this folder,
and a `pjdfs_group!` statement to export them.
For example:

#### Layout

```
chmod (syscall folder)
├── mod.rs (declarations)
└── permission.rs (a test case)
```

#### mod.rs

```rust
mod permission;

crate::pjdfs_group!(chmod; permission::test_case);
```

Each module inside a group should export a test case (with `pjdfs_test_case`),
which contains a list of test functions.
In our example, `permission.rs` would be:

#### permission.rs

```rust
use crate::{
    pjdfs_test_case,
    test::{TestContext, TestResult},
};

// chmod/00.t:L58
fn test_ctime(ctx: &mut TestContext) -> TestResult {
  for f_type in FileType::iter().filter(|&ft| ft == FileType::Symlink) {
      let path = ctx.create(f_type)?;
      let ctime_before = stat(&path)?.st_ctime;

      sleep(Duration::from_secs(1));

      chmod(&path, Mode::from_bits_truncate(0o111))?;

      let ctime_after = stat(&path)?.st_ctime;
      test_assert!(ctime_after > ctime_before);
  }

  Ok(())
}

pjdfs_test_case!(permission, { test: test_ctime });
```

Now, the assertions are grouped inside a test function,
which allows to filter tests.
It also improve reporting now that assertions are in test functions with meaningful names (instead of a number), 
and may open the possibility to run them in parallel.

### Debug

Because the rewrite is in Rust, standard debuggers (in particular `lldb`) can be used.
It will also be easier to trace syscalls with `strace`, now that a single test function can be executed.

### Performance

#### TL;DR 10% faster and more to expect with parallelization!

From a non-accurate benchmark that I did over the `chmod/00.t` test case,
the results look encouraging.
While the original took 13s, the rewrite took 1 second less, 
and this time is essentially spent on `sleep` calls!
It could greatly be improved by adding parallelization (and by splitting test functions per file type in this particular case).

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

## Thanks

Many thanks to my mentor [Alan Somers](https://github.com/asomers), for accepting my proposal, 
and guiding me through this journey (and is coincidentally one of nix's maintainers)!
