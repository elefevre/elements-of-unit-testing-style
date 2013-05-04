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

What of you need to check the content of the exception message? JUnit 4 does offer a mechanism for this, but I find it a little cumbersome *REFERENCE*. I prefer the [catch exception library](https://code.google.com/p/catch-exception/):

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


Reduce reliance on setup/teardown methods
---------------------------------------------

JUnit provides optional @Before/@After annotations to mark some methods for running before or after all other test methods (there are also @BeforeClass/@AfterClass that will be run just once, before or after all other test methods). These used to be know as the setUp() and tearDown() methods in previous versions of JUnit.

Any code in there is code that won't be considered when reading a test. And, if read, might also break the flow of reading the story told by the test. As much as possible, do not use those methods.

As an alternative, I tend to put some things as initialization of instance variables, usually simply the instanciation of the class under test, plus sometimes initialization of a few mock objects. Initialization of instance variables is a lot more limiting than code in methods, and that's partly the point. Code there (code common to all test methods) should be glaringly simple. If I'm forced to move things to the @Before method (because initialization requires complex manipulation), then it is a code smell.

    Geocoder geocoder = mock(Geocoder.class);
    PointOfInterests pois = mock(PointOfInterests.class);
    ResourceLoader loader = new ResourceLoader(geocoder, pois);

    @Test
    public void should_produce_an_error_when_city_and_postal_code_are_unspecified() {
        ...
    }


Reduce reliance on annotations
------------------------------

Some annotations are necessary for your test to run, particularly @Test. However, many of them are superfluous and will hide things from your test methods. Frameworks that provide annotations usually have the right intentions (hide irrelevant details from the tests), but they almost always come with a high price. Here is an example from using annotations provided the well-considered Mockito library. Can you find the bug?

    @RunWith(MockitoJUnitRunner.class)
    public class PointOfInterestFinderTest {
        @Mock
        PointOfInterests pois;
        @Mock
        Geocoder geocoder;

        PointOfInterestFinder finder = new PointOfInterestFinder(pois, geocoder);

        @Test
        public void can_search_for_points_of_interest_near_an_address() {
            assertThat(finder.search("a", "a", "a", "a", null)).isNotEmpty();
        }
    }

Found it? This is rather subtle. You see, the @Mock annotation, as expected, assigns a mock to the annotated instance variable. However, it does so _after_ the test class has been instanciated. In other words, _after_ PointOfInterestFinder, the class under test here, has been instantiated. So our finder has been passed only null values in its constructor. Even if the mocks are actually instantiated by the time the test method is called, it is too late for the instance variable created earlier.

A fix is to introduce an @Before method (4 lines of code). Another is to use yet another annotation, @InjectMocks, on the instance variable representing our class under test. However, this also comes with its own set of quirks and limitations[1]. This is sufficently unexpected to be the origin of a good proportion of the requests for help on the Mockito mailing list. My answer is almost always to avoid all those annotations altogether[2].

[1]: for example, it will fail silently if you do not remove the call to the constructor. Also, it attemps to figure out what to pass as parameters to the constructor, based on the type, but using the same type for separate parameters is ambiguous. There is also the question of having multiple constructors... Trust me, you'll want to avoid this annotation altogether.

[2]: see *XXXXXXX* and *XXXXX*

These problems are not due to Mockito itself. They are technical limitation from the Java environment. However, other annotations, at the very best, tends to make their intentions unclear. The @Rule annotation from JUnit 4.7 *check*, for example, is presented (by Kent Beck, no less) as a good way to create resources that must be created safely before a test method and, more importantly, must be shut down cleanly (in effect partly replacing the need for @Before/@After methods). However, few people find this easy to understand. @Before/@After methods are probably the more intuitive way (especially for newcomers on the code base) to go in general.

There are other examples, such as @Inject from Spring or XXXXXXXXXXXXX. They all suffer from some limitation. Either they require other annotations to work properly (often on the test class itself), or their behavior is difficult to understand, or they place limitations on how you can write your tests, or, simply, hide things that might be useful to see directly from within your tests.

In light of this, I recommend avoiding annotations as much as possible. It is possible that someone will eventually show me one that is worth the trouble. I won't hold my breath, though.


Keep as much context as possible within your test method
--------------------------------------------------------

If you apply all the above tips, you will naturally find much code in your actual test methods. This is a good thing. As much as possible, everything your test needs should be right there in the test method. No hidden behavior. No magic scurrying around finding other pieces of context. No looking for external resource files directly loaded from the file system.

If your test method ends up being too long for your taste (how much is too long for a test method? 10 lines? 20?), then consider this a smell, not in your tests, but rather in your production code. Should split responsabilities among more classes? Tranform helper classes into mocked services?

This goal is very much achievable in unit tests. However, I've found them a lot harder in my integration tests, and many of them end up inheriting from rather complex abstract classes. It is still an ideal that you should keep in mind, though.


Inline test values
------------------

In production code, it is recommend to extract local variables to make their usage clearer.

    isAllowedToDrink(currentYear - yearOfBirth);

    // this version is easier to understand
    int age = currentYear - yearOfBirth;
    isAllowedToDrink(age);

By the same token, test values are frequently extracted to local variables.

    String name = "very long name that might ";
    
    store.createUserWithName(name);
    
    assertThat(store.findByName(name).getName()).isEqualTo("Eric");

I do not believe this makes code easier to read.

_XXXXXXXXXXXXXXXXXXX_

Stuff to work on
================

* use local static varargs methods as builder methods
* inline test values
* do not use logs
* mock types that you do not control (sometimes)
* use factory classes to mock types instantiated in your production code
* hide unnecessary values (with nulls if necessary)
* do not use BDD/integration tests frameworks for unit testing
* use assertion libraries
* private? final? instance variables