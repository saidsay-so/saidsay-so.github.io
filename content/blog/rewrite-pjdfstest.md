+++
title = "Oxidizing FreeBSD's file system test suite"
date = 2022-09-12
draft = false

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
His high availability helped me a lot to do this project!

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
Even if it's supposed to test POSIX syscalls,
some non-standardized like `chflags(2)` are tested as well.
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

Example:

##### tests/chflags/11.t

```bash
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

### Execution

The test cases can be launched directly,
but they need a TAP harness for the assertions to work.
They are usually executed through the `prove` harness,
which is commonly included with `perl`.
##### Architecture's chart

{% mermaid() %}
flowchart LR
PR{"TAP harness\n prove"} === TC{"Test case\n chflags/11.t"} --- E["Assertion\n expect none stat ${n1} flags"] 
<==> PJD{pjdfstest binary} <==> S[/"Syscall\n stat(n1).st_flags"/]
{% end %}


## Limitations

This architecture has its advantages, 
like easier reading for the tests or quick modifications,
but several limitations also arise from it.

 <!-- * No ATF integration.  This isn't critical, but it sure would be nice if pjdfstest could display detailed results at https://ci.freebsd.org/job/FreeBSD-main-amd64-test/lastCompletedBuild/testReport/ . -->

### Configurability

Some features (like `chflags(2)`) aren't systematically implemented on file systems,
often because they aren't part of the POSIX specification.
To account this, the test suite has a concept of features.
However, the supported configurations were hard coded 
in the test suite and consequently couldn't be easily configured,
like with a configuration file or the command-line.
It also cannot be run as an unprivileged user because it assumes that the current user
is the super-user and doesn't make distinction with the parts requiring privileges.
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
an assertion is lacks context.

### Duplication

Because it's written in shell, DRY (Don't Repeat Yourself) is difficult to achieve.
This leads to a lot of duplication between the tests,
with only a few lines changed between the files.
This is particularly visible on the error tests,
which share an enormous amount of code and yet are not 
factored out.

### TAP plan

Each time a file is modified, its TAP plan needs to be recomputed.
I don't think that needs further elaboration...

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
- can filter them according to configuration and conditions,

and not in the GSoC scope but good to have:

- run tests in parallel,
- get better reports on FreeBSD's CI infrastructure with
  [ATF](https://www.freebsd.org/cgi/man.cgi?query=atf&apropos=0&sektion=0&manpath=FreeBSD+13.1-RELEASE+and+Ports&arch=default&format=html)
  metadata support.

### Test collection

The first thing we had to think on is how to collect the tests.
Since there isn't a `pytest`-like package in Rust, we had to investigate
on existing approaches.
Since the result should be a binary, techniques related to `cargo test` were excluded.
We wanted to go first with the standard Rust `#[test]` attribute to collect the functions,
but the API is not public.

{% tip() %}
[Cargo](https://github.com/rust-lang/cargo) is the Rust package manager.
This isn't its sole task, it can also launch unit tests, which are functions
annotated with the `#[test]` attribute.
{% end %}

An experimental public [implementation](https://github.com/rust-lang/rust/issues/50297)
is available on the Nightly channel,
but relying on Nightly for a test suite was quite ironic
and making harder to build it.
Instead, we initially implemented an approach 
similar to [Criterion](https://github.com/bheisler/criterion.rs)'s one,
where a test group (`criterion_group!`) is declared with the test
cases. 

##### Criterion example

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn fibonacci(n: u64) -> u64 {
    match n {
        0 => 1,
        1 => 1,
        n => fibonacci(n-1) + fibonacci(n-2),
    }
}

fn criterion_benchmark(c: &mut Criterion) {
    c.bench_function("fib 20", |b| b.iter(|| fibonacci(black_box(20))));
}

criterion_group!(benches, criterion_benchmark);
criterion_main!(benches);
```

##### Initial implementation

###### main.rs

```rust
fn main() -> anyhow::Result<()> {    
    for group in [chmod::tests] {
      ...
    }

    Ok(())
}

```

###### chmod/mod.rs

```rust
mod errno;
mod lchmod;
mod permission;

crate::pjdfs_group!(chmod; permission::test_case, errno::test_case);
```

###### chmod/permission.rs

```rust
pjdfs_test_case!(
    permission,
    { test: test_ctime, require_root: true },
    { test: test_change_perm, require_root: true },
    { test: test_failed_chmod_unchanged_ctime, require_root: true },
    { test: test_clear_isgid_bit }
);

...
```

However, as we can see, it introduced a lot of boilerplate, 
so we finally decided to use the [`inventory`](https://github.com/dtolnay/inventory)
crate.
With it, we can collect the tests without having to declare a test group.
A test can now be declared with the `crate::test_case!` macro,
which collects the test function name with 
a description (which is displayed to the user),
while allowing parameterization of the test
(to require preconditions or features, iterate over file types, 
specify that the test shouldn't be run in parallel, etc.).

##### main.rs

```rust
fn main() {
    let tests = inventory::iter<TestCase>;
}
```

##### tests/chmod.rs

```rust
crate::test_case! {
    /// chmod does not update ctime when it fails
    failed_chmod_unchanged_ctime, serialized, root => [Regular, Dir, Fifo, Block, Char, Socket]
}
fn failed_chmod_unchanged_ctime(ctx: &mut SerializedTestContext, f_type: FileType) {
    ...
}
```

{% info() %}
Here, the test will not be executed in a parallel context (`serialized`),
require privileges (`root`) and will be executed over the `Regular`, `Dir`, `Fifo`, `Block`, `Char`
and `Socket` file types.
{% end %}

### Writing tests

We now know how to collect the tests, but we still have to write them!
Again, we investigated on how to do it best.
We wanted to write them like how Rust unit tests usually are.
This means using the standard assertion macros 
(`assert!`/`assert_eq!`/`assert_ne!`) and `unwrap`ing `Result`s.

##### Unit test example (from [Rust by example](https://doc.rust-lang.org/rust-by-example/testing/unit_testing.html))

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

// This is a really bad adding function, its purpose is to fail in this
// example.
#[allow(dead_code)]
fn bad_add(a: i32, b: i32) -> i32 {
    a - b
}

#[cfg(test)]
mod tests {
    // Note this useful idiom: importing names from outer (for mod tests) scope.
    use super::*;

    #[test]
    fn test_add() {
        assert_eq!(add(1, 2), 3);
    }

    #[test]
    fn test_bad_add() {
        // This assert would fire and test will fail.
        // Please note, that private functions can be tested too!
        assert_eq!(bad_add(1, 2), 3);
    }
}
```

This implies that the test functions will panic if something goes wrong.
That's usually not a problem because the unit tests are compiled as individual binaries,
so when one test panics, it doesn't stop others from running.
But, in our case, the tests are handled by a unique runner.
We don't want to stop running all the tests when one fails!
This can be solved, by catching the panic
with the help of [`std::panic::catch_unwind`](https://doc.rust-lang.org/std/panic/fn.catch_unwind.html).

Now that we know how to write tests, we can start, right?
Well, we can if we don't want isolation between tests
(and future support for parallelization).
For example, almost all tests need to create files.
If we don't care about isolation, we could just create them
in the current directory and call it a day.
But that wouldn't work well in a parallel context,
therefore we need to isolate the tests' context from each other.
We accomplish this by creating a separate directory for each test function,
and with the `TestContext` struct,
which wraps the methods to create files inside the said directory
and automatically cleanup, among other things.

##### tests/truncate.rs

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

That would be sufficient, if we don't consider that some tests 
require functions which can affect the entire process,
like [changing effective user](https://www.man7.org/linux/man-pages/man7/credentials.7.html) 
or switching [umask](https://man7.org/linux/man-pages/man2/umask.2.html).
We need to accommodate these cases if we want the runner to be able to run the tests
in parallel in the future, hence the previously mentioned `serialized` keyword.
It allows to annotate that a test case should be the only one running
and provide functions which should be used only in this context.

##### tests/chmod.rs

{% info() %}
The function `SerializedTestContext::as_user` allows to change
the effective user in a controlled manner.
{% end %}

```rust
crate::test_case! {
    /// S_ISGID bit shall be cleared upon successful return from chmod of a regular file
    /// if the calling process does not have appropriate privileges, and if
    /// the group ID of the file does not match the effective group ID or one of the
    /// supplementary group IDs
    clear_isgid_bit, serialized, root
}
fn clear_isgid_bit(ctx: &mut SerializedTestContext) {
    let path = ctx.create(FileType::Regular).unwrap();
    chmod(&path, Mode::from_bits_truncate(0o0755)).unwrap();

    // Get a dummy user from context
    let user = ctx.get_new_user();

    chown(&path, Some(user.uid), Some(user.gid)).unwrap();

    let expected_mode = Mode::from_bits_truncate(0o2755);
    // Change user
    ctx.as_user(&user, None, || {
        chmod(&path, expected_mode).unwrap();
    });

    ...
}
```

With all this and a few other methods, we rewrote the test suite
while solving the previous limitations.

### What's new?

#### Isolation

Now, the assertions are grouped inside a test function,
which allows filtering tests with improved granularity.
It also improves reporting,
now that assertions are in test functions with meaningful names
and descriptions.

Run example:

```sh
> pjdfstest -c pjdfstest.toml chmod

pjdfstest::tests::chmod::failed_chmod_unchanged_ctime::socket	
	 chmod does not update ctime when it fails		success
pjdfstest::tests::chmod::failed_chmod_unchanged_ctime::char	
	 chmod does not update ctime when it fails		success
pjdfstest::tests::chmod::failed_chmod_unchanged_ctime::block	

...

Tests: 0 failed, 0 skipped, 26 passed, 26 total
```

Which is clearer than:

```sh
> prove -v tests/rename/00.t
../tests/rename/00.t .. 
1..122
ok 1
ok 2
ok 3
ok 4
ok 5
ok 6
ok 7
...
ok
All tests successful.
Files=1, Tests=122,  8 wallclock secs ( 0.04 usr  0.01 sys +  0.65 cusr  0.41 csys =  1.11 CPU)
Result: PASS
```

#### Performance
#### TL;DR 100x faster and more to expect with parallelization!

This is probably the most exciting part and Rust doesn't disappoint on this one!
With the improved configurability, it's now possible to manually specify a time
for the sleeps, which depends on the timestamp granularity of the tested file system. 
This greatly improves the speed, but this isn't the only slowness factor.

| Test suite | Time |
|------------|------|
| Original test suite (prove) | 350s |
| Original test suite (prove - 8 jobs) | 139s |
| Rewrite with 1s sleep time | 88s |
| Rewrite with 1ms sleep time | 1s |

> Tested on a FreeBSD laptop with 8 cores, on the ZFS file system.
> The original test suite is executed with the `prove` TAP harness.

From these non-rigorous benchmarks, we can see that there is an important speed gap
between the original test suite and its rewrite.
With reduced sleep time, the rewrite can execute the entire test suite in only one second!
Even with 1-second sleeps, the rewrite is still faster than the original!

Benchmark commands:
```sh
> sudo hyperfine -i './target/release/pjdfstest -c pjdfstest.toml >/dev/null' 'prove -rvj8 ../tests >/dev/null'

Benchmark 1: ./target/release/pjdfstest -c pjdfstest.toml >/dev/null
  Time (mean ± σ):      1.338 s ±  0.202 s    [User: 0.002 s, System: 0.027 s]
  Range (min … max):    1.136 s …  1.768 s    10 runs

  Warning: Ignoring non-zero exit code.

Benchmark 2: prove -rvj8 ../tests >/dev/null
  Time (mean ± σ):     138.946 s ±  2.419 s    [User: 2.947 s, System: 9.995 s]
  Range (min … max):   136.582 s … 143.314 s    10 runs

  Warning: Ignoring non-zero exit code.

Summary
  './target/release/pjdfstest -c pjdfstest.toml >/dev/null' ran
  103.82 ± 15.81 times faster than 'prove -rvj8 ../tests >/dev/null'

With 1 second sleeps:

> sudo hyperfine -ir5 './target/release/pjdfstest -c pjdfstest.toml >/dev/null' 'prove -rvj8 ../tests >/dev/null'

Benchmark 1: ./target/release/pjdfstest -c pjdfstest.toml >/dev/null
  Time (mean ± σ):     88.563 s ±  0.265 s    [User: 0.003 s, System: 0.025 s]
  Range (min … max):   88.322 s … 88.946 s    5 runs

  Warning: Ignoring non-zero exit code.

Benchmark 2: prove -rvj8 ../tests >/dev/null
  Time (mean ± σ):     138.636 s ±  2.822 s    [User: 2.779 s, System: 9.532 s]
  Range (min … max):   136.659 s … 143.455 s    5 runs

  Warning: Ignoring non-zero exit code.

Summary
  './target/release/pjdfstest -c pjdfstest.toml >/dev/null' ran
    1.57 ± 0.03 times faster than 'prove -rvj8 ../tests >/dev/null'

```

#### Configurability

The test suite can optionally be configured with a configuration file,
to specify what are the supported features or
the minimum sleep time for file system to takes changes into account for example.

#### Debugging

Because the rewrite is in Rust, standard debuggers (in particular `lldb`) can be used.
It is also easier to trace syscalls with `strace`,
now that a single test function can be executed.

### Progress status

At the time of writing, the test suite has (almost) been completely rewritten!
The original test suite had more than 7400 lines split between 225 files.
This rewrite includes more than 92% of the original test suite, the exception being
the granular tests.
We judged that it would take too much time to rewrite all of them accurately,
especially because NFSv4 ACLs are really complicated (there is no written
standard to check the expected behavior, we have to rely on the implementations)
and the tests are actually incomplete.
Otherwise, some tests still need to be merged,
and we're refactoring the error tests,
but the rewrite is already usable.

<!-- ## Looking forward


### ATF support

### Parallelization

### Features clarification

### Command-line improvements

file types filter, syscall filter -->