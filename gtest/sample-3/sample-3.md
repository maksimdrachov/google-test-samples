## Sample 3

uses a test fixture

Test fixtures: using the same data configuration for multiple tests

If you find yourself writing two or more tests that operate on similar data, you can use a _test fixture_. This allows you to reuse the same configuration of object for several different test.

To create a fixture:

1. Derive a class from `::testing::Test`. Start its body with `protected:`, as we'll want to access fixture members from sub-classes.
2. Inside the class, declare any objects you plan to use.
3. If necessary, write a default constructor or `SetUp()` function to prepare the object for each test. A common mistake is to spell `Setup()` (instead of `SetUp()`)
4. If necessary, write a destructor or `TearDown()` function to release any resources you allocated in `SetUp()`.
5. If needed, define subroutines for your tests to share.

When using a fixture, use `TEST_F()` instead of `TEST()` as it allows you to access objects and subroutines in the test fixture:

```cpp
TEST_F(TestFixtureName, TestName) {
    ... test body ...
}
```

Like `TEST()`, the first argument is the test suite name, but for `TEST_F()` this must be the name of the test fixture class. 

Also, you must first define a test fixture class before using it in a `TEST_F()`, or you'll get the compiler error "virtual outside class declaration".

For each test defined with `TEST_F()`, googletest will create a fresh test fixture at runtime, immediately initialize it via `SetUp()`, run the test, clean up by calling `TearDown()`, and then delete the test fixture. Note that different tests in the same test suite have different test fixture objects, and googletest always deletes a test fixture before it creates the next one. googletest does not reuse the same test fixture for multiple tests. Any changes one test makes to the fixture do not affect other tests.

As an example, let's write tests for a FIFO queue class named `Queue`, which has the following interface:

```cpp
template <typename E>   // E is the element type
class Queue {
  Queue();
  void Enqueue(const E& element);
  E* Dequeue();  // Returns NULL if the queue is empty.
  size_t size() const;
  ...
};
```

First, define a fixture class. By convention, you should give it the name `FooTest` where `Foo` is the class being tested.

```cpp
class QueueTest : public ::testing::Test {
 protected:
  void SetUp() override {
     q1_.Enqueue(1);
     q2_.Enqueue(2);
     q2_.Enqueue(3);
  }

  // void TearDown() override {}

  Queue<int> q0_;
  Queue<int> q1_;
  Queue<int> q2_;
};
```

In this case, `TearDown()` is not needed since we don't have to clean up after each test, other than what's already done by the destructor. 

Now we'll write tests using `TEST_F()` and this fixture.

```cpp
TEST_F(QueueTest, IsEmptyInitially) {
  EXPECT_EQ(q0_.size(), 0);
}

TEST_F(QueueTest, DequeueWorks) {
  int* n = q0_.Dequeue();
  EXPECT_EQ(n, nullptr);

  n = q1_.Dequeue();
  ASSERT_NE(n, nullptr);
  EXPECT_EQ(*n, 1);
  EXPECT_EQ(q1_.size(), 0);
  delete n;

  n = q2_.Dequeue();
  ASSERT_NE(n, nullptr);
  EXPECT_EQ(*n, 2);
  EXPECT_EQ(q2_.size(), 1);
  delete n;
}
```

The above uses both `ASSERT_*` and `EXPECT_*` assertions. The rule of thumb is to use `EXPECT_*` when you want the test to continue to reveal more errors after the assertion failure, and use `ASSERT_*` when continuing after failure doesn't make sense. For example, the second assertion in the `Dequeue` test is `ASSERT_NE(n, nullptr)`, as we need to dereference the pointer `n` later, which would lead to a segfault when `n` is `NULL`.

**Invoking the Tests**

`TEST()` and `TEST_F()` implicitly register their tests with googletest. So, unlike other C++ testing frameworks, you don't have to re-list all your defined tests in order to run them.

After defining your tests, you can run them with `RUN_ALL_TESTS()`, which returns `0` if all the tests are successful, or `1` otherwise. Note that `RUN_ALL_TESTS()` runs all tests in your link-unit—they can be from different test suites, or even different source files. 

You must not ignore the return value of `RUN_ALL_TESTS()`, or you will get a compiler error. The rationale for this design is that the automated testing service determines whether a test has passed based on its exit code, not on its stdout/stderr output; thus your `main()` function must return the value of `RUN_ALL_TESTS()`.

Most users should _not_ need to write their own `main` function and instead link with `gtest_main` (as opposed to with `gtest`), which defines a suitable entry point. The remainder of this section should only apply when you need to do something custom before the tests run that cannot be expressed within the framework of fixtures and test suites.

If you write your own `main` function, it should return the value of `RUN_ALL_TESTS()`.

```cpp
int main(int argc, char ** argv) {
    ::testing::InitGoogleTest(&argc, argv);
    return RUN_ALL_TESTS();
}
```

The `::testing::InitGoogleTest()` function parses the command line for googletest flags, and removes all recognized flags. This allows the user to control a test program's behaviour via various flags, which we'll cover in the [AdvancedGuide](https://google.github.io/googletest/advanced.html). You must call this function before calling `RUN_ALL_TESTS()`, or the flags won't be properly initialized. 

But maybe you think that writing all those `main` functions is too much work? That's why googletest provides a basic implementation of `main()`. If it fits your needs, then just link your test with the `gtest_main` library and you're good to go.

