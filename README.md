Elements of unit testing style
==============================

Many papers have been written on the process of writing unit tests, including using the test-first approach. Other papers describe what makes a good test, from the technical point of view. In this chapter, I want to focus on the style of the resulting test. A form of coding standards, if you will, focused on the tests.

Coding standards already exist. However, it is my conviction that they are best applied to production code. Rules are a bit different for tests. In the following, I describe what differing rules I use in my tests.

Name test methods with underscores
----------------------------------

In production code, common coding standards for Java code recommend writing method names in CamelCase.

    retrieveUserDetails()
    createABankAccount()

This makes sense when those methods are called. It feels as if requests are being made by the user of the objects ("bankTeller.createABankAccount()", please). However, methods in unit tests are not called by our code. They do not need to represent requests. They do need to tell a story. This story is best told using underscore.

    should_not_fail_when_dividing_by_zero()
    can_add_2_and_2_and_return_4()



Stuff to work on:
names should start with 'can' or 'should'
tests should have a cyclomatic complexity of 1 avoid loops
duplicate code (sparingly)
use local static varargs methods as builder methods
avoid calling test methods from other test methods, even for setup
avoid using the setup/teardown methods
inline test values
instantiate your class under test as an instance variable
instantiate your mocks as instance variables
do not use logs
avoid annotations in your tests, except for @Test
mock types that you do not control (sometimes)
use factory classes to mock types instantiated in your production code
