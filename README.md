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

    should_search_by_name_address()
    cannot_fail_when_address_is_not_specified()


Work with a framework idiomatic to your programming language
------------------------------------------------------------

If you code in Java, test in Java. In JavaScript, test in JavaScript. Ideally with a test framework that lets you write your unit tests in a way that does not feel contrived (or constrained) compared to writing production code.
For example, until its version 4.0, JUnit required test classed to inherit from a special class, TestCase, and that test methods started with a special prefix 'test'. Although a good framework, these constraints felt rather arbitrary[^
1] and were replaced with annotations in later versions of JUnit.


In contrast, unit testing your production language in a different language[^2] will require you to adjust your mental state every time you switch from production code to unit test code. Instead, we want as seamless transition as possible, as we'll frequently go from one to the other, especially when practicing TDD.


The same advice applies when considering BDD frameworks with their own language or formalism such as Cucumber and FitNesse. Although those may have value as acceptance test environments, using them for unit testing will slow down the feedback cycle created by quickly going from reading/writing tests to reading/writing production code.

Finally, notice how proficient you are in your production code? That comes from years of practice. It also comes from a production development environment. Use those to your advantage.

[1]: in truth, they were present for technical reasons; it did feel like putting too many constraints on the developer, though
[^2]: as advocated [here](http://www.ibm.com/developerworks/java/library/j-pg11094/), for example


Method names should tell a story
--------------------------------

There is actually dome (healthy) debate on how to name your test methods in a unit test class. My preference is to let my method names tell me a story. For this, I like my method names to start with a verb that lets me describe what will be tested, in functional terms if possible.

For example, here is a method:
    public class LocalBusinessResourceTest {
        @Test
        public void should_limit_the_number_search_matches_to_20_by_default() {
            ...
        }
    }

What I read, half-unconsciously, is "The class LocalBusinessResource should limit the number of search matches to 20, by default." The test itself will lay the technical details, and I'll make sure that the figure "20" will appear somewhere, in order to make the whole thing as obvious as possible. However, the method name tells me the general functional contract, and will avoid detailing the implementation (although I'll admit we are getting close).

Starting with "should" or "can" helps in this regard. Some authors advocate starting with ["fact"](https://github.com/marick/Midje) or following a [UnitOfWork\_StateUnderTest\_ExpectedBehavior](http://osherove.com/blog/2005/4/3/naming-standards-for-unit-tests.html) pattern. I feel it makes the flow of the story less natural, though.

One drawback of this style is that method names are longer than is usual. I have never found this to be an issue in practice. Only on rare occasions do the names reach more than 100, which is very acceptable.

For more about using "should", check out some of Liz Keogh's posts, such as [this one](http://lizkeogh.com/2005/03/14/the-should-vs-will-debate-or-why-doubt-is-good/).


Tests should have no branches
-----------------------------
# avoid loops, try/catch clauses
As opposed to production code, tests describe a straight story. What things we start with, what we do with them, and what we get (some call this the Given/When/Then pattern, or the *XXXXXX* Act Assert). This means that the code would be written as a single branch that can be read with as little mental effort as possible.

Although it might seem obvious in simple cases, it also means that, for example, you should avoid loops when initializing your test data:

    public void can_limit_search_results_to_5()
    {
    	// I recommend this over loops
        List<String> list = new ArrayList<String>();
        list.add("string");
        list.add("string");
        list.add("string");
        list.add("string");
        list.add("string"); // 5
        list.add("string"); // 6
        
        assertThat(search.findByName("")).hasSize(6);
    }

Similarly, avoid try/catch clauses. In older versions of JUnit, making sure that an exception was thrown meant writing code like this:

    public void can_fail_when_searching_on_an_invalid_name() {
        try {
            search.findByName("an invalid name");
            fail();
        }
        catch (RuntimeException e) {
            // success
        }
    }

Since JUnit 4.0, this can be refactored as:
    @Test(expected = RuntimeException.class)
    public void can_limit_search_results_to_ten() {
        search.findByName("an invalid name");
    }
This has the added benefit of a more explicit error message.

What of you need to check the content of the exception message? JUnit 4 does offer a mecanism for this, but I find it a little cumbersome *REFERENCE*. I prefer the *GOOGLE CODE PROJECT EXCEPTION*:

    *CODE SAMPLE*


Duplicate code (sparingly)
--------------------------
It is my belief that as much care should be given to the code in the test as to the code in production. However, as already shown in this paper, I do not think that exactly the same rules apply.

In particular, I think it is a lot more acceptable to duplicate code in your tests:


    @Test
    public void should_limit_matches_to_100_meters_when_street_is_specified()
    {
        when(geocoder.geocode("Country", "City", "Street")).thenReturn(new Coordinate(1., 1.));
        when(allPois.withinCircle(new Coordinate(1., 1.), 100.0)).thenReturn(newArrayList(new Business("Restaurant")));

        assertThat(localBusinessResource.searchBusinesses("Country", "City", "Street"))//
                .contains(new Business("Restaurant"));
    }

    @Test
    public void should_limit_matches_to_100_meters_when_street_is_specified()
    {
        when(geocoder.geocode("Country", "City", "Street")).thenReturn(new Coordinate(1., 1.));
        when(allPois.withinCircle(new Coordinate(1., 1.), 5000.0)).thenReturn(newArrayList(new Business("Restaurant")));

        assertThat(localBusinessResource.searchBusinesses("Country", "City", "Street"))//
                .contains(new Business("Restaurant"));
    }

In this situation, I much prefer duplicate lines of code, rather than hide my test setup.

Some people like to have an initial unit test that sets things up, and checks for basic behavior. And then call this test from other, more intricate tests.

    private Messages messages = new Messages();

    @Test
    public void should_save_a_new_message()
    {
    	messageCreator.add("Hello", messages);

        assertThat(messages).hasSize(1);
    }

    @Test
    public void should_delete_a_message()
    {
    	should_save_a_new_message();

    	messageCreator.remove("Hello");

        assertThat(messages).isEmpty();
    }

I am very much against this technique, as it makes a lot less clear what is being tested, and what might actually break. My preference goes to duplicating code. If it seems too painful to do so, then it is probably a sign that production code should be refactored. Only at the last resort would I start to factorize initialization code into a separate methods. Never would I call another test.

Stuff to work on
================
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
avoid sub-blocks (check exception details with https://code.google.com/p/catch-exception/)
hide unnecessary details (with nulls if necessary)
do not use BDD/integration tests frameworks for unit testing
use assertion libraries