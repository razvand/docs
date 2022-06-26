---
title: Testing Unikraft
date: 2022-06-26T09:27:37+10:00
weight: 7
summary: "We present the uktest framework for testing with Unikraft. Expected time: 75min."
---

## Testing Unikraft

There are three types of testing: unit testing, integration testing and end-to-end testing.
To better understand the difference between them, we will look over an example of a webshop.
If we're testing the whole workflow (creating an account, logging in, adding products to a cart, placing an order) we will call this **end-to-end testing**.
Our shop also has an analytics feature that allows us to see a couple of data points such as: how many times an article was clicked on, how much time did a user look at it and so on.
To make sure the inventory module and the analytics module are working correctly (a counter in the analytics module increases when we click on a product), we will be writing **integration tests**.
Our shop also has at least an image for every product which should maximize when we're clicking on it. To test this, we would write a **unit test**.

Running the test suite after each change is called **regression testing**. **Automatic testing** means that the tests are run and verified automatically. **Automated regression testing** is the best practice in software engineering.

## Unikraft's Testing Framework

Unikraft's testing framework, [`uktest`](https://github.com/unikraft/unikraft/tree/staging/lib/uktest), has been inspired by [KUnit](https://docs.kernel.org/dev-tools/kunit/index.html) and provides a flexible testing API.
[For licensing reasons](https://opensource.org/licenses/BSD-3-Clause) the Unikraft project cannot use this source code.

In `uktest`, tests are organised hierarchically powered by the the lowest common denominator: the assertion.
However, by inspiration, we can organise the `uktest` library following a similar pattern:

1. The assertion: represents the lowest common denominator of a test: some boolean operation which, when true, a single test passes.
   Assertions are often used in-line and their usage should be no different to the traditional use of the `ASSERT` macro.
   In `uktest`, we introduce a new definition: `UK_TEST_EXPECT` which has one parameters: the same boolean opeation which is true-to-form of the traditional `ASSERT` macro.
   With `uktest`, however, the macro is intelligently placed in context within a case (see 2.).
   Additional text or descriptive explanation of the text can also be provided with the auxiliary and similar macro `UK_TEST_ASSERTF`.

1. The test case: often, assertions are not alone in their means to check the legitimacy of operation in some function.
   We find that a "case" is best way to organise a group of assertions in which one particular function of some system is under-going testing.
   A case is independent of other cases, but related in the same sub-system.
   For this reason we register them together in the same test suite.

1. The test suite: represents a group of test cases.
   This is the final and upper-most heirarchical repesentation of tests and their groupings.
   With assertions grouped into test cases and test cases grouped into a test suite, we end this organisation in a fashion which allows us to follow a common design pattern within Unikraft: the registration model.
   The syntax follows similar to other registation models within Unikraft, e.g. `ukbus`.
   However, `uktest`'s registation model is more powerful (see Test Execution).

### Creating tests

To register a test suite with `uktest`, we simply invoke `uk_test_suite_register` with a unique symbol name.
This symbol is used along with test cases in order to create the references to one-another.
Each test case has only two input parameters: a reference to the suite is part, as well as a canonical name for the case itself.
Generally, the following pattern is used for test suites:

`$LIBNAME_$TESTSUITENAME_testsuite`

An the following for test cases:

`$LIBNAME_test_$TESTCASENAME`

To create a case, simply invoke the `UK_TESTCASE` macro with the two parameters described previously, and use in the context of a function, for example:

```c
UK_TESTCASE(uktest_mycase_testsuite, uktest_test_case)
{
        int x = 1;
        UK_TEST_EXPECT(x > 0);
}
```

Finally, to register the case with a suite (see next section), call one of the possible registration functions:

```c
uk_testsuite_register(uktest_mycase_testsuite, NULL);
```

The above snippet can be organised within a library in a number of ways such as in-line or as an individual file representing the suite.
There are a number of test suite registration handles which are elaborated on in next section.
It should be noted that multiple test suites can exist within a library in order to test multiple features or components of said library.

{{< alert theme="info" >}}

### Recommended Conventions

In order to achieve consistency in the use of `uktest` across the Unikraft code base, the following recommendation is made regarding the registration of test suites:

 1. A single test suite should be organised into its own file, prefixed with `test_`, e.g. `test_feature.c`.
 This makes it easy to spot if tests for a library are being compiled in.
 2. If there is more than one test suite for a library, all tests suites of the library should be stored within a new folder located at the root of the library named `tests/`.
 3. All tests suites should have a corresponding KConfig option, prefixed with the library name and then the word "TEST", e.g. `LIBNAME_TEST_`.
 4. Every library implementing one or more suite of tests must have a new menuconfig housing all test suite options under the name `LIBNAME_TEST`.
 This menuconfig option must invoke all the suites if `LIBUKTEST_ALL` is set to true.
{{< /alert >}}

### Registering tests

`uktest`'s registation model allows for the execution of tests at different levels of the boot process.
All tests occur before the invocation of the application's `main` method.
This is done such that the validity of the kernel-space functions can be legitimised before actual application code is invoked.
A fail-fast option is provided in order to crash the kernel in case of failures for earlier error diagnosis.

When registering a test suite, one can hook into either the constructor "`ctor`" table or initialisation table "`inittab`".
This allows for running tests before or after certain libraries or sub-systems are invoked during the boot process.

The following registation methods are available:

 * `UK_TESTSUITE_AT_CTORCALL_PRIO`,
 * `uk_testsuite_early_prio`,
 * `uk_testsuite_plat_prio`,
 * `uk_testsuite_lib_prio`,
 * `uk_testsuite_rootfs_prio`,
 * `uk_testsuite_sys_prio`,
 * `uk_testsuite_late_prio`,
 * `uk_testsuite_prio` and,
 * `uk_testsuite_register`.

## Work Items

In this work session we will go over writing and running tests for Unikraft.
We will use `uktest`.
`uktest` should be enabled from the Kconfig.

### Support Files

Session support files are available in the session folder.

### 01. Tutorial: Testing a Simple Application

We will begin this session with a very simple example.
We can use the `app-helloworld` as a starting point.
In `main.c` remove all the existing code.
The next step is to include `uk/test.h` and define the factorial function:

```C++
#include <uk/test.h>

int factorial(int n) {
  int result = 1;
  for (int i = 1; i <= n; i++) {
    result *= i;
  }

  return result;
}
```

We are now ready to add a test suite with a test case:

```C++
UK_TESTCASE(factorial_testsuite, factorial_test_positive)
{
       UK_TEST_EXPECT_SNUM_EQ(factorial(2), 2);
}

uk_testsuite_register(factorial_testsuite, NULL);
```

When we run this application, we should see the following output.

```
test: factorial_testsuite->factorial_test_positive
    :	expected `factorial(2)` to be 2 but was 2 ....................................... [ PASSED ]
```

Throughout this session we will extend this simple app that we have just written.

### 02. Adding a New Test Suite

For this task, you will have to modify the existing factorial application by adding a new function that computes if a number is prime.
Add a new testsuite for this function.

### 03. Tutorial: Testing vfscore

We begin by adding a new file for the tests called `test_stat.c` in a newly created folder `tests` in the `vfscore` internal library:

```Makefile
LIBVFSCORE_SRCS-$(CONFIG_LIBVFSCORE_TEST_STAT) += \
    $(LIBVFSCORE_BASE)/tests/test_stat.c
```

We then add the menuconfig option in the `if LIBVFSCORE` block:

```KConfig
menuconfig LIBVFSCORE_TEST
    bool "Test vfscore"
    select LIBVFSCORE_TEST_STAT if LIBUKTEST_ALL
    default n

if LIBVFSCORE_TEST

config LIBVFSCORE_TEST_STAT
    bool "test: stat()"
    select LIBRAMFS
    default n

endif
```

And finally add a new testsuite with a test case.

```C++
#include <uk/test.h>

#include <fcntl.h>
#include <errno.h>
#include <unistd.h>
#include <sys/stat.h>
#include <sys/mount.h>

typedef struct vfscore_stat {
    int rc;
    int errcode;
    char *filename;
} vfscore_stat_t;

static vfscore_stat_t test_stats [] = {
    { .rc = 0,    .errcode = 0,        .filename = "/foo/file.txt" },
    { .rc = -1,    .errcode = EINVAL,    .filename = NULL },
};

static int fd;

UK_TESTCASE(vfscore_stat_testsuite, vfscore_test_newfile)
{
    /* First check if mount works all right */
    int ret = mount("", "/", "ramfs", 0, NULL);
    UK_TEST_EXPECT_SNUM_EQ(ret, 0);

    ret = mkdir("/foo", S_IRWXU);
    UK_TEST_EXPECT_SNUM_EQ(ret, 0);

    fd = open("/foo/file.txt", O_WRONLY | O_CREAT, S_IRWXU);
    UK_TEST_EXPECT_SNUM_GT(fd, 2);

    UK_TEST_EXPECT_SNUM_EQ(
        write(fd, "hello\n", sizeof("hello\n")),
        sizeof("hello\n")
    );
    fsync(fd);
}

/* Register the test suite */
uk_testsuite_register(vfscore_stat_testsuite, NULL);
```

We will be using a simple app without any main function to run the testsuite, the output should be similar with:

```
test: vfscore_stat_testsuite->vfscore_test_newfile
    :	expected `ret` to be 0 but was 0 ................................................ [ PASSED ]
    :	expected `ret` to be 0 but was 0 ................................................ [ PASSED ]
    :	expected `fd` to be greater than 2 but was 3 .................................... [ PASSED ]
    :	expected `write(fd, "hello\n", sizeof("hello\n"))` to be 7 but was 7 ............ [ PASSED ]
```

### 04. Add a Test Suite for nolibc

Add a new test suite for nolibc with four test cases in it.
You can use any POSIX function from nolibc for this task.
Feel free to look over the [documentation](https://github.com/lancs-net/unikraft/blob/nderjung/uktest/lib/uktest/include/uk/test.h) to write more complex tests.

## Further Reading

* [6.005 Reading 3: Test](https://ocw.mit.edu/ans7870/6/6.005/s16/classes/03-testing/index.html#automated_testing_and_regression_testing)
* [A gentle introduction to Linux Kernel fuzzing](https://blog.cloudflare.com/a-gentle-introduction-to-linux-kernel-fuzzing/)
* [Symbolic execution with KLEE](https://adalogics.com/blog/symbolic-execution-with-klee)
* [Using KUnit](https://www.kernel.org/doc/html/latest/dev-tools/kunit/usage.html)
