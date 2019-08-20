---
layout:     post
title:      Test-Driven Bug Fixing
date:       2019-08-20
summary:    A pragmatic way to test iOS Applications
categories: ios, testing
---

Hopefully we can all agree that testing is important. Having a suite of tests that runs (hopefully) everytime you make a change lets you make changes with confidence that you aren't going to introduce a bug to your users.

Sometimes though, it's tough to know where to started with testing. A search for ios testing best practices turns up a variety of results talking about test-driven development (TDD), dependency injection, mocking/stubbing, and many other concepts. The problem is, if you're working in an existing codebase these concepts may not be present. You likely don't have the time (or a desire) to refactor everything in one fell swoop, but you would like to start iteratively improving the quality of your codebase, and adding test coverage along the way.

A great way to get started with testing in a codebase is by following a test-driven approach to fixing bugs. If you want to skip to an example of this approach, [here you go](#example).

## What is Test-Driven Bug Fixing

Test-driven bug fixing is a narrow application of test-driven development (TDD) principles specifically for fixing bugs. While it may _sound_ complicated, I'm hoping to convince you that it's not anything to be afraid of. Test-driven bug fixing involves adding a test (or tests!) that will fail when the bug is present, and will pass when the bug is fixed.

## Why you should use it

Test-driven bug fixing is a great approach because the scope of what you're testing is typically rather small. This means that if you need to do any refactoring in order to make your code more testable, you're not making massive changes to your codebase. More likely, you're tweaking a few methods in a single file. It also means that this testing and related refactoring can be done independently of the rest of the codebase. While each bug fix may only improve the quality of a single method, over time concepts such as dependency injection, or writing pure fuctions and isolating side effects will start to make their way through your codebase.


## An Example Please?
{: #example}

Now for some code. I'll take you through an example of applying test-driven priciples to fixing a bug.

### Identifing the Bug

We'll start by looking at the following method:

```swift
func isInTheFuture(_ dateUnderTest: Date) -> Bool {
    return Date().timeIntervalSince(dateUnderTest) > 0
}
```

We've unintentionally flipped the order of our date comparison, so this method will return `false` for all future values.

### Writing the Test

Now that we've identified where the bug is, it's time to start writing the test. Since we need to test the behavior of the `isInTheFuture(_:)` method, we might start to write something that tests the following 3 cases:

1. Verify true is returned if the date is in the future
2. Verify false is returned if the date is now
3. Verify false is returned if the date is not in the future

For #1 and #3, we can create values because we know that `Date()` is used in the method. However for #2, how do we create a test value to excercise that case? Maybe with some refactoring we can make it just as easy to create a test for case #2.


### Pre-Test Refactoring

One trick when testing dates, is to inject a function `() -> Date` and use that rather than directly calling the date initializer `Date()`. We can even provide `Date.init` as a default value so all of the `isInTheFuture` call sites don't have to change a thing!

After this refactor, we end up with a new form of our `isInTheFuture` method:

```swift
func isInTheFuture(
    _ dateUnderTest: Date,
    now: () -> Date = Date.init
) -> Bool {
    return now().timeIntervalSince(dateUnderTest) > 0
}
```

_IMPORTANT: This is not the time to actually fix the bug. We're trying to recreate the existing behavior, but in a more testable form._

### Writing the Test (again)

Now that we've refactored our method to enable testing, we can write a test (or tests) to verify the three cases we defined above.

```swift
func test_isInTheFuture() {
    let referenceDate = Date(timeIntervalSinceReferenceDate: 0)
    let dateAfterReferenceDate = Date(timeIntervalSinceReferenceDate: 60)

    // Verify true is returned if the date is in the future.
    let result = isInTheFuture(dateAfterReferenceDate, now: { referenceDate })
    XCTAssertTrue(result)

    // Verify false is returned if the date is now.
    let result2 = isInTheFuture(dateAfterReferenceDate, now: { dateAfterReferenceDate })
    XCTAssertFalse(result2)

    // Verify false is returned if the date is not in the future.
    let result3 = isInTheFuture(referenceDate, now: { dateAfterReferenceDate })
    XCTAssertFalse(result3)
}
```

### Running the Test

Time to run our test! Unsurprising it fails.

![test fail]({{ site.baseurl }}/images/20190813/test_fail.png)


### Refactoring the Code

Now it's time to actually implement the fix. We should swap the order of our comparison so that we're getting the time interval since the current date.

```swift
func isInTheFuture(_ dateUnderTest: Date, now: () -> Date = Date.init) -> Bool {
    return dateUnderTest.timeIntervalSince(now()) > 0
}
```

### Running the Test Again

Now when we run our test again, it passes 🎉!

![test success]({{ site.baseurl }}/images/20190813/test_success.png)

We've successfully accomplished a test-driven bug fix.

## In Conclusion...

Bugs in software are a fact of life. As that example hopefully demonstrated, as you fix them it's easy to practice test-driven bug fixing. While fixing our bug, we improved our code in a number of ways. First, we improved the quality of our code by refactoring to enable testing of our code. We made our method's dependency explicit, and exposed a way to control the dependency while writing a test. Next, we increased the test coverage, and have added a layer of protection that will hopefully prevent this bug from ever occurring in production again. Lastly and most importantly, we accomplished the previous two points in an iterative way. We didn't have to re-write our entire class, or restructure our whole application to add dependency injection and testability.
