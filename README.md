Elements of unit testing style
==============================

Many papers have been written on the process of writing unit tests, including using a test-first approach. Other papers (such as [this article](http://codebetter.com/jeremymiller/2005/07/20/qualities-of-a-good-unit-test/) describe what makes a good test, from the technical point of view: tests should be atomic, independant, isolated, clear, easy and fast. In this chapter, I want to focus on the style of the resulting test. A form of coding standards, if you will, focused on the tests.

Coding standards already exist, of course. However, it is my conviction that the available ones apply best to production code. Rules are a bit different for tests. In the following, I describe what rules I use in my tests.


Name test methods with underscores
----------------------------------

In production code, common coding standards for Java code recommend writing method names in CamelCase.

    retrieveUserDetails()
    createABankAccount()

This makes sense when those methods are called. It feels as if requests are being made by the user of the objects (dear "bankTeller.createABankAccount()", please). However, methods in unit tests are not called by our own code. They do not need to represent requests. They do need to tell a story, and this story is best told using underscore.

    should_search_by_name_address()
    cannot_fail_when_address_is_not_specified()


Tell a story with the method name
---------------------------------

There is actually some (healthy) debate on how to name your test methods in a unit test class. My preference is to let my method names tell a story. For this, I like my method names to start with a verb that lets me describe what will be tested, in functional terms if possible. I write grammatically-correct sentences, complete with articles and plurals.

For example, here is a test method:

    public class LocalBusinessResourceTest {
        @Test
        public void should_limit_the_number_search_matches_to_20_by_default() {
            ...
        }
    }

What I read, half-unconsciously, is "The class LocalBusinessResource should limit the number of search matches to 20, by default." The test itself will expose the technical details, and I'll make sure that the figure "20" will appear somewhere, in order to make the whole thing as obvious as possible. However, the method name tells me the general functional contract, and will avoid detailing the implementation.

Starting with "should" or "can" helps in this regard. Some authors advocate starting with ["fact"](https://github.com/marick/Midje) or following a [UnitOfWork\_StateUnderTest\_ExpectedBehavior](http://osherove.com/blog/2005/4/3/naming-standards-for-unit-tests.html) pattern. I feel it makes the flow of the story less fluid, though.

One drawback of this style is that method names are longer than is usual. I have never found this to be an issue in practice. Only on rare occasions do the names reach more than 100 characters, which is very acceptable.

For more about using "should", check out some of Liz Keogh's posts, such as [this one](http://lizkeogh.com/2005/03/14/the-should-vs-will-debate-or-why-doubt-is-good/).


Work with a test framework idiomatic to your programming language
-----------------------------------------------------------------

Code in Java, test in Java. In JavaScript, test in JavaScript. Ideally with a test framework that lets you write your unit tests in a way that does not feel contrived or constrained compared to writing production code.

For example, until its version 4.0, JUnit required that test classes to extend a special class, TestCase, and that test methods started with a special prefix. These constraints felt rather arbitrary (in truth, they were present for technical reasons) and were replaced with annotations in later versions of JUnit.

In contrast, unit testing your production language in a different language (as advocated [here](http://www.ibm.com/developerworks/java/library/j-pg11094/), for example) will require you to adjust your mental state every time you switch from production code to unit test code. Instead, we want as seamless a transition as possible, as we'll frequently switch from one to the other, especially when practicing TDD.

The same advice applies when considering BDD frameworks that come with their own language or formalism such as Cucumber and FitNesse. Although those may have value as acceptance test environments, using them for unit testing will slow down the feedback cycle created by quickly going from reading/writing tests to reading/writing production code.

Finally, notice how proficient you are in your production code? That comes from years of practice. It also comes from a productive development environment. Use those to your advantage.


Avoid branches in the test code
-------------------------------

As opposed to production code, tests should describe a linear story. What things we start with, what we do with them, and what we get. Some call this the Given/When/Then pattern, others the Arrange/Act/Assert pattern. This means that the code should be written as a single thread that can be read with as little mental effort as possible.

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

        Store store = new Store(list);
        
        assertThat(store.findByName("string")).hasSize(5);
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
This has the added benefit of a more explicit error message from JUnit.

What of you need to check the content of the exception message? JUnit 4 does offer a mechanism for this (see the [ExpectedException Rule](https://github.com/junit-team/junit/wiki/Exception-testing#expectedexception-rule), but I find it rather cumbersome. I prefer the [catch exception library](https://code.google.com/p/catch-exception/):

    public void can_fail_when_searching_on_an_invalid_name() {
        catchException(search).findByName("an invalid name");

        assertThat(caughtException()).isInstanceOf(RuntimeException.class);
        assertThat(caughtException()).hasMessage("Invalid name");
    }


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

I am very much opposed to this technique, as it makes a lot less clear what is being tested, and what might actually break. My preference goes to duplicating code. If it seems too painful to do so, then it is probably a sign that production code should be refactored. Only at the last resort would I start to factorize initialization code into a separate methods. Never would I call another test.


Reduce reliance on setup/teardown methods
-----------------------------------------

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

A fix is to introduce an @Before method (4 lines of code). Another is to use yet another annotation, @InjectMocks, on the instance variable representing our class under test. However, this also comes with its own set of quirks and limitations (among other things, it will fail silently if you do not remove the call to the constructor, guess, and sometimes fail, what to pass as parameters to the constructor, guess and saometimes fail which constructor to use, pass null when the dependency is not provided, etc.). This is sufficently unexpected to be the origin of a good proportion of the requests for help on the Mockito mailing list. My answer is almost always to avoid all those annotations altogether (see [this thread](https://groups.google.com/d/topic/mockito/Kik9Pt3kW6k/discussion) for example).

These problems are not due to Mockito itself. They are technical limitation from the Java environment. However, other annotations, at the very best, tends to make their intentions unclear. The @Rule annotation from JUnit 4.7 *XXXXXXXXXXXXXXXXXcheckXXXXXXXXXXXXXXX*, for example, is presented (by Kent Beck, no less) as a good way to create resources that must be created safely before a test method and, more importantly, must be shut down cleanly (in effect partly replacing the need for @Before/@After methods). However, few people find this easy to understand. @Before/@After methods are probably the more intuitive way (especially for newcomers on the code base) to go in general.

There are other examples, such as @RunWith(SpringJUnit4ClassRunner.class) from Spring Framework or RestFuse for testing HTTP APIs. They all suffer from some limitation in my view. Either they require the usage of additional annotations (often on the test class itself), or their behavior is difficult to understand, or they place limitations on how you can write your tests, or, simply, hide things that might be useful to see directly from within your tests.

In light of this, I recommend avoiding annotations as much as possible. It is possible that someone will eventually show me one that is worth the trouble. In the meantime, I'll remain skeptical.


Keep as much context as possible within your test method
--------------------------------------------------------

If you apply all the above tips, you will naturally find much code in your actual test methods. This is a good thing. As much as possible, everything your test needs should be right there in the test method. No hidden behavior. No more scurrying around finding other pieces of context. No looking for external resource files directly loaded from the file system. 

Some smells to look for:

* data declared at the beginning of your test class (inline them in your test method)
* calculated expected values (use literal values)
* test data stored in files (inline them in your tests)

If your test method ends up being too long for your taste (how much is too long for a test method? 10 lines?), then consider this a smell, not in your tests, but rather in your production code. Guided by the feedback from your test, you should be able to write better production code. Should responsabilities be split among more classes? Should helper classes be made into services?

This goal is very much achievable in unit tests. However, I've found them a lot harder in my integration tests, and many of them end up inheriting from rather complex abstract classes. It is still an ideal that you should keep in mind, though.


Avoid local variables in test methods
-------------------------------------

In production code, it is recommend to extract local variables to make their usage clearer.

    isAllowedToDrink(currentYear - yearOfBirth);

    // this version is easier to understand
    int age = currentYear - yearOfBirth;
    isAllowedToDrink(age);

By the same token, I often see test values extracted to local variables.

    String name = "Eric";

    store.createUserWithName(name);

    assertThat(store.findByName(name).getName()).isEqualTo("Eric");

I do not believe this makes code much easier to read. I'd rather see the following:

    store.createUserWithName("Eric");
    
    assertThat(store.findByName("Eric").getName()).isEqualTo("Eric");

It makes clearer that things are happening in distinct steps. First, we update the store with a new name. Later, when we search with a String that has the same value, then we get our previously stored element. Reusing the same variable for storage can create confusion ("is the underlying code comparing pointers and not values?"), and takes valuable space on screen.
Note that I do not mind the risk of mistyping the hard-coded values. _The point of this test is that it will catch those problems._

Note that this works for any data type you have, not just primitive types, as long as your data types have a properly implemented equals() method:

    store.createUser(user());
    
    assertThat(store.findAllUsers()).contains(user());

In this example, user() is a static builder method, local to this test class, that creates a new instance of User, passing dummy data to the constructor if necessary (usually nulls, empty collections and zeroes).
The important point is that we are making clear that users are being manipulated and that the current is not dealing with a particular attribute of the user (if it did, then I'd also have builder methods such as user(String firstName), user(int age), etc.).

Aren't we losing the information provided by the name of the variable? Often, the name of the variable is not particularly explicit, such as "name" in the earlier example. If the name of the variable _was_ conveying information, then I tend to use it as the value itself:

    // not recommended
    String longName = "Eric Eric Eric Eric Eric Eric Eric Eric Eric Eric Eric Eric";

    store.createUserWithName(longName);

    assertThat(store.findByName(longName).getName()).isEqualTo("Eric Eric Eric Eric Eric Eric Eric Eric Eric Eric Eric Eric");

    // better
    store.createUserWithName("long name long name long name long name long name long name long name");

    assertThat(store.findByName("long name long name long name long name long name long name long name").getName()).isEqualTo("long name long name long name long name long name long name long name");


Create dedicated builder methods within test classes
----------------------------------------------------

Some of the data classes used in my tests are sometimes a bit difficult to instantiate. In extreme situations, they can take several lines of code just for creating a single object.

    DateTime started = new DateTime(1L);
    DateTime ended = new DateTime(2L);
    new BackTest("session id", "backtest id", "solution id", Status.ENDED, "details", 0, started, ended);

Not all of this is relevant for the test you are currently considering. What you want is to see only the parameters that have an impact in the current context.

Over the years, I have used several strategies to work around this. One option is to pass nulls, except for the parameter you are interested in, but the remaining might still be too distracting. Another is to provide an [Object Mother](http://martinfowler.com/bliki/ObjectMother.html), a class that have pre-configured objects; this is the solution I like the least, as it creates indirect coupling between tests. Yet another idea is to provide several constructors in the classes being instantiated; this have limitations, though, as not all combinaisons of parameters are possible.

In the end, I've settled on builder methods _inside my test classes_. They would come in as many flavors as needed for my test class:

    private static BackTest backTest(String backtestId) {
        DateTime started = new DateTime(1L);
        DateTime ended = new DateTime(2L);
        new BackTest("session id", backtestId, "solution id", Status.ENDED, "details", 0, started, ended);
    }

    private static BackTest backTest(String backtestId, String solutionId) {
        DateTime started = new DateTime(1L);
        DateTime ended = new DateTime(2L);
        new BackTest("session id", backtestId, solutionId, Status.ENDED, "details", 0, started, ended);
    }

If by chance there is ambiguity in the parameters necessary, it is easy to rename builders as appropriate:

    private static BackTest backTestWithBackTestId(String backtestId) {
        DateTime started = new DateTime(1L);
        DateTime ended = new DateTime(2L);
        new BackTest("session id", backtestId, "solution id", Status.ENDED, "details", 0, started, ended);
    }

    private static BackTest backTestWithSolutionId(String solutionId) {
        DateTime started = new DateTime(1L);
        DateTime ended = new DateTime(2L);
        new BackTest("session id", "backtest id", solutionId, Status.ENDED, "details", 0, started, ended);
    }

Note how those builder methods are private; I much prefer keep them specific to my test classes. In my projects, I have not found much value in factorizing them into some BackTestBuilder class (you might have realized by now that I put a lot of effort into avoiding coupling between test classes). These methods are also static. This is partly for aesthetic reasons (italics are nice), and also because it makes clearer that they should not be considered as part of the code currently under test.

In the past, I also tended to create builder methods that take varargs (see [this post from 2011](http://ericlefevre.net/wordpress/2011/11/21/javas-varargs-are-for-unit-tests/) for more):

    assertThat(findLongestName(users("a long name", "short name"))).isEqualTo(user("a long name"));

Nowadays, I tend to write builder methods for single objects, and call them multiple times:

    assertThat(findLongestName(newArrayList(user("a long name"), user("short name"))).isEqualTo(user("a long name"));


Test whole objects
------------------

I occasionally see tests that check whether an object has been modified:

    Company company = oldCompany.addEmployee(new Employee("John"));

    assertThat(company.getEmployees()).contains(new Employee("John"));

The reasoning is that the test is more focused on the exact thing being tested. It might also be easier to write, as everything else in the object can be ignored.

My problem with this approach is that it does not make obvious that the new object is a modified copy of the previous one. Also, it uses production code (the getEmployees() method) which might be computed out of some other data and might lead to unexpected behavior.

I'd much prefer see the whole object being tested. I view data structures as self-contained, especially when they are immutable. The fact that it is sometimes possible to observe a portion of this object does not convey clearly that the entire new object is in a new state. Besides, getters are not always present, and I'd refrain from creating them just for the purpose of a test.

My preference goes to code like this:

    Company company = new Company().addEmployee(new Employee("John"));

    List<Employee> employees = new ArrayList();
    employees.add(new Employee("John"));
    assertThat(company).isEqualTo(new Company(employees);

This is rather ugly, though. Custom builders methods will help. With a bit of inlining, I usually end up pushing the following code:

    assertThat(new Company().addEmployee(employee("John"))) //
        .isEqualTo(new Company(newArrayList(employee("John")));


Use assertion libraries
-----------------------

Assertions are ways to express the verifications to be done after exercizing your code under test. Early versions of JUnit came with a limited set of assertions (with a few variants):

* assert()
* assertEquals()
* notEqualsMessage()

This proved rather insufficient, and messages were not always very explicit, so later versions of JUnit expanded on those. By JUnit 3, there were 8 assertions methods, and finally, in JUnit 4.4, there was the addition of assertThat(), a generic assertion method based on Hamcrest ([quoting Kent Beck](https://twitter.com/KentBeck/status/331422460371156992): "@elefevre i like hamcrest, but i don't think the dependency (our first ever) was worth it on balance. haven't talked about it publicly tho."). This made developers aware of the existence of fluent APIs for verifying the results from their production code. These APIs are great, and I strongly recommend them for writing your tests.

I am aware of 3 fluent assertion libraries:

* Hamcrest, a version of which is included in JUnit 4.4 and later 
* FEST Assert
* AssertJ, actually a fork from FEST Assert

My preference goes to AssertJ. Like FEST Assert, I find its style particularly readable. Unlike AssertJ, one of its basic principles is to come with many well-named methods to check for values.

Here are examples of usages:

    // with JUnit, prior to v4.4
    assertEquals(store.searchByName("name").size(), 10);

    // with JUnit 4.4+, or with Hamcrest
    assertThat(store.searchByName("name"), hasSize(10));

    // with FEST Assert or AssertJ
    assertThat(store.searchByName("name")).hasSize(10);

I feel that AssertJ's style, although little different from Hamcrest's, is a bit more readable. Also, it benefits from easier auto-completion.


Do not use logs
---------------

It is well known that printing debug traces in the console from production code is bad practice (see [this thread](http://stackoverflow.com/questions/8601831/do-not-use-system-out-println-in-server-side-code), for example). The general recommendation is to use logs instead (although there is also some debate on that, as in [this post](http://www.mockobjects.com/2007/04/test-smell-logging-is-also-feature.html) by Steve Freeman). So it would seem natural to do the same on the test side.

The question is, what do you want logs in the tests for? Logs are generally used for diagnostic of unexpected events in production. There is no such need in tests. If your failing tests do not provide enough context, then you must refactor them to do so. Generally, this will mean making your assertions more clear. In the worst case, you might have to run your test via a debugger.

Now, if the problem is that the unit tests are using resources that can not be easily inspected (random numbers, for example), then they are most likely _not_ unit tests, but rather integration tests. You should refactor them to take well-know numbers instead.


Avoid descriptions in assertions
--------------------------------

A feature of assertions provided by JUnit is that each of them come with an optional Description parameter.

    assertEquals("result", compute());
    assertEquals("Something went wrong", "result", compute());

I've always felt that those descriptions, like Javadoc comments in production code, added very little value. I've also noticed that, with the recent spread of assertions library, descriptions have become much less prevalent, even though those libraries do offer the option of adding descriptions.

    assertThat(compute()).describedAs("Something went wrong").isEqualTo("result");
    assertThat("Something went wrong", compute(), equalTo("result"));

It seems that descriptions have been used in older versions of JUnit as a way to cope against the limitations of the available assertion. Now that assertions can be a lot more specific ("<['hello', 'world', 'world']> contains duplicate(s):<['world']>"), it seems that descriptions provide little value.

Today, there is no excuse for clutches like descriptions in your tests. Use specific assertions liberaly instead. And remember, unit tests are made to be easily re-run, so if the error message is not explicit enough, you always have the option to run your tests and see for yourself.


Mock types that you do not control, as a first step
---------------------------------------------------

Steve Freeman and Nat Pryce make a compeling case that you should only [mock types that you own](http://www.mockobjects.com/2008/11/only-mock-types-you-own-revisited.html). Although the reasoning is right, I do not feel this is the simplest possible thing.

My approach is to mock even classes that I do not own. That is, I have no problem with injecting my services with objects external to the source code directly under my control.

There are limitations to that. For example, some external classes are marked as final, or some of their methods are final and/or static, making it painfully hard to mock (it is possible to do so with some mocking libraries, but I never found it was worth the trouble). In those cases, yes, I would wrap them with a class of my own design that would simply redirect to these external classes. However, these technical limitations aside, I tend to mock external classes without shame.

    public void should_find_only_text_files_in_the_specified_directory() {
        File file = mock(File.class);
        when(file.list()).thenReturn(new String[] { "readme.txt", "foobar" });

        assertThat(store.list(file)).contains("readme.txt");
    }

I also mock external services. In this example, mocking the central class from Morphia lets me easily check whether it is configured as expected.

    public void should_register_all_business_classes() {
        Mapper mapper = mock(Mapper.class);
        Morphia morphia = mock(Morphia.class);
        MorphiaWrapper wrapper = new MorphiaWrapper(morphia);
        when(morphia.getMapper()).thenReturn(mapper);

        wrapper.registerBusinessClasses();

        verify(mapper).addMappedClass(Customer.class);
        verify(mapper).addMappedClass(Purchase.class);
    }

However, in practice, it turns out that this sort of situation is a lot less common than it seems. Also, it often turns out that I end up introducing intermediate classes between my services and external classes. In the first example listed here, the store ended up taking only "LocalFiles" in parameter, an immutable class that contained only the information useful for the rest of the application.

    public void should_find_only_text_files_in_the_specified_directory() {
        assertThat(store.list(new LocalFile().withFiles("readme.txt", "foobar"))).contains("readme.txt");
    }


Use wrapper classes to mock helper classes
------------------------------------------

Occasionally, it is useful to refer to static methods from helper classes. A good example is obtaining the current time from the System class:

    public void saveCurrentTimeToFile() {
        long now = System.currentTimeMillis();
        ...
    }


The issue, of course, is that this is really hard to instrument in your test classes. An option is to pass the time in parameter:

    public void saveCurrentTimeToFile(long now) {
        ...
    }

This works only to a point, especially if one wants to hide this from client classes. My solution is to introduce a wrapper class:

    public class SystemTime {
        public long currentTimeMillis() {
            return System.currentTimeMillis();
        }
    }

This class can usually be injected automatically by Spring or Guice. Or, as I often do, it is possible to add constructors especially for instantiated this wrapper class:

    public Store() {
        this(new SystemTime());
    }

    // useful for testing
    Store(SystemTime systemTime) {
        this.systemTime = systemTime;
    }

I often end with almost one wrapper class per helper class, especially those from Apache Commons: IOUtils, SystemUtils, StringUtils... My wrapper classes generally have the same name as their external counterparts, making it easier to find them.

Should you test your wrapper classes? My advise is to make them so simple that testing should not be necessary.


Use factoris to help mock classes instantiated in production code
-----------------------------------------------------------------

Instantiating things in your production code is usually not a problem. You instantiate a new object, manipulate it, and use its result (or send it to another service). Reasonably easy to test.

Sometimes, the object is more complex. Maybe you need information from the filesystem and you are instantiating Files. Maybe you are using a scheduling library that runs tasks at specific intervals (and instantiating the scheduler puts all sorts of things in movement). It might be cleaner to wrap them all with your own classes, but you want to know more before doing so.

In those situations, I usually start by extract the instantiation code in a local method, and override this method in the unit tests:

    public void should_save_data_to_the_store_file() {
        final File file = mock(File.class);
        Store store = new Store() {
            @Override
            File newFile(String filename) {
                return file;
            }
        };

        store.save("data");
        ...
    }

This is rather ugly, so if this happens a couple of times for the same type, I tend to quickly introduce a factory.

    public class FileBuilder {
        public File newFile(String name) {
            return new File(name);
        }
    }

    public class Store {
        private FileBuilder fileBuilder;

        public Store(FileBuilder fileBuilder) {
            this.fileBuilder = fileBuilder;
        }

        public void save(String data) {
            File file = fileBuilder.newFile("store");
            ...
        }
    }

    public class StoreTest {
        File file = mock(File.class);
        FileBuilder fileBuilder = mock(FileBuilder.class);

        Store store = new Store(fileBuilder);

        @Test
        public void should_save_data_to_the_store_file() {
            when(fileBuilder.newFile("store")).thenReturn(file);
            
            store.save("data");
            ...
        }
    }

Like wrappers, your builder classes should so simple that testing is not necessary.


Do not assume that the system is in a useable state
---------------------------------------------------

Many papers on unit tests advise ensuring that tests leave the system in a clean state. I do not necessarily agree with this view. Instead, I'd rather tests assume that the system is in a dirty state (within reason) and clean it themselves.

        @Test
        public void should_save_a_user_successfully() throws IOException {
            store.clear(); // assuming that store is a shared resource

            store.save(new User());

            assertThat(store.countElements()).isEqualTo(1);
        }

A benefit is that, if one of your tests does fail, it won't affect the behavior of other tests (yes, I know that things like @After are there to help tests clean after themselves; this is not always enough or easy to write in a completely sage way. Plus, this makes for more elements to maintain around your tests).
Another is that it puts in a single place everything that is necessary for your tests to run. No clicking around trying to figure out what might have happened before and fix a suspicious test that does not seem to close its resources properly.
A final benefit, and this is the most important one, is that it shows what tests are candidates for refactoring. Having to call "store.clear()" is rather unfortunate and it makes the tests slightly uglier; would a better option be to make it into a mock object? This would also make your tests run faster. This sort of feedback is harder to obtain when cleanup code is hidden in special separate methods.


Do not assume or impose that tests be run in a specific order
-------------------------------------------------------------

In the early days in JUnit, many teams I knew were trying to use every feature it provided (it had so few! they must all be have been useful, right?). That meant stuffing tests into "test suites". This was a pain because it was all too easy to forget to update the suite after a new test had been created. It was also too tempting to comment out a failing test, only to forget to uncomment it after fixing it.

Worst is that it tempted developers to write tests that were expecting to be run in a certain order. For example, one test would populate the database, the following would check that a search was possible, and the last would check that deletion would work. In other words, those tests would be largely interdependent, difficult to parallelize, difficult to maintain (are you sure the 'delete' test has actually removed something? or was it that the other test just never properly populate the database?).

My advice is to stop relying on such mecanism. Do not assume that tests will be run in a specific order. This will help you make them as independent of each other as possible. Which will also put pressure towards writing production code in small, independent modules.


What about functional tests?
----------------------------

All those rules apply to functional tests. That said, performance costs are greater for functional tests, as servers are started, test data are populated, etc.

However, keeping those rules in mind helps me design tests that are easier to read and maintain. Often, I start by applying them strictly, particularly at the beginning of a project, and only slowly relax them as I have no choice but to speed up the build process to keep it under an acceptable duration.

