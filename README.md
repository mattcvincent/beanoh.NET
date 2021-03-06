Beanoh.NET, pronounced \'beanˌō dot net\ , is a simple open source way to verify you Spring.NET context. Teams that leverage Beanoh spend less time focusing on configuring Spring.NET and more time adding business value. 

For licensing information refer to the license section. 

## Features
1. Verify that all of your objects are wired correctly
2. Prevent duplicate object definition overwriting

## Dependencies
Beanoh.NET is for verifying Spring.NET contexts, so naturally it depends on Spring.NET. More specifically, Spring.Core, Spring.Aop and Spring.Data. Moreover, it also depends on Castle.Core for proxying, Common.Logging for logging, Moq for mocking and more importantly NUnit as the implementation framework of the Beanoh.NET test fixtures. 

For Licensing information of those dependencies : 

1. Castle.Core : Apache License V2.0 (http://www.apache.org/licenses/LICENSE-2.0.html)
2. Common.Logging : Apache License V2.0 (http://www.apache.org/licenses/LICENSE-2.0.html)
3. Moq : BSD License (http://opensource.org/licenses/bsd-license.php)
4. NUnit : http://nunit.org/index.php?p=license&r=2.5
5. Spring.NET : Apache License V2.0 (http://www.apache.org/licenses/LICENSE-2.0.html)

## Writing Beanoh.NET Test Fixtures
Beanoh.NET tries to follow a simple convention so that you will spend less time learning Beanoh.NET and more on gaining the feedback it provides. The convention in question is that your NUnit test fixture needs to extend the base `SourceAllies.Beanoh.BeanohTestCase` test fixture, and writing one Spring.NET bootstrap context that loads all the contexts you need to verify. That bootstrap context need to be in a certain location and named in a certain way for it to be picked up by Beanoh.NET. We will talk about those requirements shortly. 


### a. Assert Simple Context Loading
For example, let's say we want to verify a Spring.NET context found in `assembly://MyAssembly/My.Cute.Namespace/someSpringContext.xml` that defines objects and wire them together and it can possibly be loading other contexts in its turn. First we create a NUnit Test Fixture, let's call it `BasicWiringTest`, and it needs to extend `SourceAllies.Beanoh.BeanohTestCase` test fixture. 

    //Our Code will look like this 
    namespace My.Tests.Verbose.Namespace
    {
        [TestFixture]
       public class BasicWiringTest : BeanohTestCase 
        {
            [Test]
            public void TestWiring()
            {
                AssertContextLoading();
            }
        }
    }


To make a simple verification that every object definition in our Spring.NET context is being loaded, all we need is to define a test method `TestWiring()` that just calls `BeanohTestCase.AssertContextLoading()` method. 

Next, we need to define the bootstrap context that we were talking about earlier. For now all you need to know about is that it needs to satisfy the following criteria : 
1. It's to be named `X-BeanohContext.xml`. Where `X` is the name of your Test Fixture class, hence for our example the context name is `BasicWiringTest-BeanohContext.xml`.
2. It needs to live in the same assembly as our test fixture. 
3. It needs to be in the same folder as our test fixture so that it will have the same namespace as the test fixture.
4. It needs to be defined as an embedded resource so it will be copied along with our test fixture during compilation and execution. 


    <!-- BasicWiringTest-BeanohContext.xml bootstrap context will be as follows --> 
    <objects xmlns="http://www.springframework.net">
      <import resource="assembly://MyAssembly/My.Cute.Namespace/someSpringContext.xml"/>
    </objects>
    

Now, you can just run the test and let Beanoh.NET do its magic. 

Something to point out is when the bootstrap conforms to these criteria, the effect will be that when you build your test assembly, you will find the test fixture class and the context living together in the same namespace in the generated DLL. Hence, if you have issues with running the test and you get a message saying couldn't find context, then your first troubleshooting task should be to inspect the tests assembly DLL and check that the bootstrap context and test fixture have the same namespace. You can inspect the DLL using tools like CodeReflect. 

**Note:** In future Beanoh.NET releases, we might add some flexibility around how to define the bootstrap context once we get a better understanding how Beanoh.NET users want that to be. 


### b. Assert No Duplicate Object Definitions
Another feature of Beanoh.NET is it can point out if you have duplicate object definitions where you are using the same object id for defining two or more objects. We all ran into situations where we are trying to troubleshoot a bug where one of our objects are not behaving like they should, and after a painful debugging session we find out that we pulled in some context that defined another object with the same id and that's the one being used by our program. 

Now Beanoh.NET, can detect by creating a test fixture and bootstrap context like we did in the previous example. The only difference is that our test method is calling `BeanohTestCase.AssertUniqueObjectContextLoading()`.

    //our code will look like this 
    namespace SourceAllies.Beanoh.Duplicate
    {
        [TestFixture]
        class DuplicateObjectTest : BeanohTestCase
        {
            [Test]
            public void testDuplicates()
            {
               AssertUniqueObjectContextLoading();
            }
        }
    }


    <!-- our DuplicateObjectTest-BeanohContext.xml bootstrap is as follows -->
    <objects xmlns="http://www.springframework.net">
         <!-- load two contexts that might be defining duplicate object definitions --> 
         <import resource="FirstDuplicate-context.xml" />
         <import resource="SecondDuplicate-context.xml" />
    </objects>

Run the test, and it will fail if it finds duplicates. 

### c. Ignore Certain Object Definitions While Checking for Duplication
Of course, there are legitimate cases where intentionally want to override some object definitions. An example would be to replace one data source with another for testing purposes, or we want to discard an old object definition pulled from an old assembly that we cannot change and replace it with our own implementation. 

Beanoh.NET can take those cases into consideration very easily, all you need is to instruct your test fixture that you want to exclude the object ids in questions from the duplicate check as follow : 

    //our code will look like this 
    namespace SourceAllies.Beanoh.Duplicate
    {
        [TestFixture]
        class IgnoreOneDuplicateObjectTest : BeanohTestCase
        {
            [Test]
            public void testDuplicates()
            {
           	IgnoreDuplicateObjectNames("person");
		    AssertUniqueObjectContextLoading();
            }
        }
    }

The previous code will have the effect of excluding any object definition with id of `person` from our check, but it will catch all other duplicates. Of course, don't forget to name your bootstrap context appropriately as `IgnoreOneDuplicateObjectTest-BeanohContext.xml`.

### b. Override a IDbProvider During Verification
Beanoh.NET provides a convenient method of overriding any `IDbProvider` object definitions that you might have in contexts. It could be the case, that you don't want to be connecting to your database during Beanoh.NET verifications. 

All you need to do is to override that `IDbProvider` object definition with another one in the bootstrap context as follows (where "DbProvider" is the object id of the dbprovider we want to override and "AbstractBeanohDbProvider" is the id of the object definition provided by Beanoh.NET) : 

    <objects xmlns="http://www.springframework.net">
       <!-- 
         This bootstrap context will load a context that requires a dbProvider which shouldn't be 
         used in our tests.  Beanoh.NET provides a convenience data source definition that can be 
         used to override such dependencies 
       -->
      <import resource="DbProviderDefinition-context.xml"/>
      <object id="DbProvider" parent="AbstractBeanohDbProvider"/>
    </objects>


Now, you can run your tests even the `AssertUniqueObjectContextLoading()` calls without worrying that Beanoh.NET will say that "DbProvider" is duplicated because Beanoh.NET excludes objects defined in the bootstrap context. That's why we are restricting you to define only one bootstrap context and name it in a certain way so that Beanoh.NET can identify objects defined in it. 

## Displaying Beanoh.NET logs 
Beanoh.NET logs all pertaining messages using Common.Logging framework, for you to be able to see those log messages you need to add a reference to your favorite logging framework like log4net and also the appropriate connector framework that connects Common.Logging to your logging framework. In my case, it will be Common.Loggin.Log4Net. 

Then your app.config needs to include the following lines and define the log4net configuration in a separate file log4net.config : 

    <configuration>
    	.................. 
      <!-- configuring Common.Logging logger -->
      <configSections>
        <sectionGroup name="common">
          <section name="logging" type="Common.Logging.ConfigurationSectionHandler, Common.Logging" />
        </sectionGroup>
      </configSections>
      ................... 
       <!-- connecting Common.Logging to log4net-->
      <common>
        <logging>
          <factoryAdapter type="Common.Logging.Log4Net.Log4NetLoggerFactoryAdapter, Common.Logging.Log4net">
            <arg key="configType" value="FILE-WATCH" />
            <arg key="configFile" value="~/log4net.config" />
          </factoryAdapter>
        </logging>
      </common>
    </configuration>

For more information on this, refer to Common.Logging documentation. 

## License
Copyright (c) 2011  Source Allies

This library is free software; you can redistribute it and/or
modify it under the terms of the GNU Lesser General Public
License as published by the Free Software Foundation version 3.0.

This library is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
Lesser General Public License for more details.

You should have received a copy of the GNU Lesser General Public
License along with this library; if not, please visit 
http://www.gnu.org/licenses/lgpl-3.0.txt.