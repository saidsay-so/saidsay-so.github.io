+++
title = "Rewrite PJDFSTest suite in Rust"
date = 2022-05-22
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
But what this proposal is about, what does it have to do with Rust anyway?

<!-- more -->

In this post, I will answer to these questions,
by explaining what is the project,
its current shortcomings and how rewriting it to Rust would solve them.

{% info() %}
The proposal is available [here](https://github.com/musikid/gsoc/).
{% end %}

## What is PJDFSTest?

First of all, [PJDFSTest](https://github.com/pjd/pjdfstest) is a test suite for POSIX system calls.
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
It will execute the test cases (in `tests/`),
and produce a report from the TAP outputs of these cases.

{% tip() %}
More info about the TAP protocol [here](https://en.wikipedia.org/wiki/Test_Anything_Protocol).
{% end %}

These cases are grouped in folder,
generally named after the syscall being tested,
and split between files named by a number (for example `tests/chmod/00.t`). 
They are made up of assertions,
using `expect` bash function, which calls the `pjdfstest` binary. 
The binary then executes the syscall and send its result back to the `expect` function,
which compares the expected and actual outputs,
and fail when it has to.

<!--

#### Test case

The typical shape of a test case is:

- Description
- Include of `misc.sh`
- Check for feature support
- TAP plan for the case
- Randomly generated names
- Assertions

```sh
#!/bin/sh

desc="chflags returns EPERM if a user tries to set or remove the SF_SNAPSHOT flag"

dir=`dirname $0`
. ${dir}/../misc.sh

require chflags_SF_SNAPSHOT

echo "1..145"

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

-->

### Limitations

Several limitations arise from those points.
First, since the test suite relies on TAP, 
a lot of assertions are written in a single file.
Some files have more than 1000 assertions, making debugging them difficult.

Talking of debuggability, the fact that it's written in shell makes it easier to write tests, 
but incredibly hard to debug and review, 
given the lack of a good shell debugger and isolation between the assertions.

## Rewrite it in Rust

### 


## Conclusion

## Thanks

Thanks to my mentor [asomers](https://github.com/asomers), who will guide me through this journey (and is coincidentally one of nix's maintainers!).
