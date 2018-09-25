<p align="center">
    <img src="https://raw.githubusercontent.com/cloudyrock/dimmer-project/master/misc/logo.png" width="200" />
</p>

# Dimmer Project Examples! Lightweight Java library for flexible feature toggling
Dimmer is a lightweight Java library to manage customisable feature toggling. 
In addition to the traditional feature toggle on/off functionalities, Dimmer offers 
developers to configure a custom response for a feature with a simple Java annotation. 
This additional flexibility enables the developers to configure custom behaviours for specific environments.

# Table of contents
- [Why Dimmer?](#why-dimmer)
- [Some uses cases](#some-uses-cases)
- [How does Dimmer work?](#how-does-dimmer-work)
- [How does it look?](#how-does-it-look)
- [Prerequisites](#prerequisites)
- [Getting started](#getting-started)
- [Lets see some code!](#lets-see-some-code)
- [Throwing a default exception](#throwing-a-default-exception)
- [Returning a fixed value](#returning-a-fixed-value)
- [Behaviours: When exceptions and fixed values are not enough](#behaviours-when-exceptions-and-fixed-values-are-not-enough)
- [Throwing custom exceptions](#throwing-custom-exceptions)
- [Environments](#environments)
- [Conditional toggling](#conditional-toggling)
- [Logging](#logging)
- [Known issues](#known-issues)
    - [Aspectj libraries and Lombok project don't work well together in IDEs](#aspectj-libraries-and-lombok-project-dont-work-well-together-in-ides)
- [LICENSE](#license)





## Why Dimmer?
During the development lifecycle, there are some scenarios when enabling/disabling a particular 
feature in environment is needed. Feature toggling comes handy to grant this ability without 
code changes but, how can we achieve providing a custom behaviour of a feature for a particular 
environment without a production code change?

Sometimes is not about temporarily disabling some features, but providing
a custom behaviour for a specific environments. For example, when we would like to mock a 
3rd party remote service in an environment without depending on an external component, 
returning specific values or throwing exceptions for a specific feature in an environment.

Most of teams just use a home-made solution which normally implies the use of if-else 
in production code, which for obvious reasons is not ideal(production code should contain
just production code ;) ). Some other teams use some 3rd parties, but most of them are
either too big and complex, or too simple and don't provide the flexibility to provide
different behaviours depending on different scenarios.

## Some uses cases
- You are working on a feature that needs some service from your colleague, which is has the interface defined but is still under development(Release toggle)

- Your team is using trunk-based development and some feature is partially developed and don't want to expose it yet(Release toggle).

- You are using a 3rd party remote service which is only available in some environments(Ops toggle)

- You want to perform a canary release(Experiment toggle)

- or you want to do A/B testing(Experiment toggle)

- or you want temporally(or not) filter the invocation based on some factors(Permissioning toggle)



## How does Dimmer work?
Dimmer works by processing annotated methods(using aspects), containing the feature which is toggled off or on.
Dimmer works in a two-phases process. First it finds out if a feature is toggled on or of and then, when off, 
provides the mocked behaviour(which can variate from throwing an exception to performing an absolutely custom action).
Both phases are managed by a configuration builder(DimmerBuilder).

In the next section you can see how it looks a basic configuration. Please notice that
terminal build methods(build and buildWithDefaultEnvironment) will inject the given configuration
to the aspect, which then will be ready, working and able to process the annotated features.



## How does it look?
```java
public class Main {

    private static final String FEATURE_NAME = "feature_name";

    public static void main(String... args) {
        DimmerBuilder
                .local()
                .defaultEnvironment()
                .featureWithValue(FEATURE_NAME, "fake value")
                .buildWithDefaultEnvironment();

        final String result = new Main().runFeaturedMethod();
        System.out.println(result);
    }

    @DimmerFeature(FEATURE_NAME)
    private String runFeaturedMethod() {
        return "real value";
    }

}
```
## Prerequisites
In order to use Dimmer you need:
- At least [JDK 8 ](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
- Build automation tool like [Maven](https://maven.apache.org/) or [Gradle](https://gradle.org/) 


## Getting started

You can check the last version of the library in this [link](https://search.maven.org/search?q=a:dimmer-local)

Using Maven:
```xml
<dependency>
    <groupId>com.github.cloudyrock.dimmer</groupId>
    <artifactId>dimmer-local</artifactId>
    <version>LATEST_VERSION</version>
</dependency>
```

Using Gradle:
```groovy
compile group: 'com.github.cloudyrock.dimmer', name: 'dimmer-local', version: 'LATEST_VERSION'
```

In order to make aspects work(so Dimmer can do its magic), you need to add some trivial configuration to your 
build script.

Using Maven:
```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>aspectj-maven-plugin</artifactId>
    <version>1.4</version>
    <dependencies>
        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjrt</artifactId>
            <version>1.8.2</version>
        </dependency>
        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjtools</artifactId>
            <version>1.8.2</version>
        </dependency>
    </dependencies>
    <executions>
        <execution>
            <phase>process-classes</phase>
            <goals>
                <goal>compile</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <showWeaveInfo/>
        <forceAjcCompile>true</forceAjcCompile>
        <sources/>
        <weaveDirectories>
            <weaveDirectory>${project.build.directory}/classes</weaveDirectory>
        </weaveDirectories>
        <aspectLibraries>
            <aspectLibrary>
                <groupId>com.github.cloudyrock.dimmer</groupId>
                <artifactId>dimmer-local</artifactId>
			</aspectLibrary>
		</aspectLibraries>
		<source>1.8</source>
		<target>1.8</target>
	</configuration>
</plugin>
```



## Lets see some code!
Once you have ensured the few requirements, you are ready to write some code

## Throwing a default exception 
The most basic scenario is just throwing a default exception when a feature is called. 
The Dimmer configuration looks like this:
```java
DimmerBuilder
    .local()
    .defaultEnvironment()
    .featureWithDefaultException(FEATURE_NAME)
    .buildWithDefaultEnvironment();
```
Code explanation:
- `.local()` indicates Dimmer to take the local configuration. In next releases will be possible to use remote configuration
- `.defaultEnvironment()` and buildWithDefaultEnvironment() are used when we don't want to deal with different environments.
- `.featureWithDefaultException(FEATURE_NAME)` tells Dimmer to throw a default exception(DimmerInvocationException) whenever a 
method annotated with `@DimmerFeature(FEATURE_NAME)` is invoked.


Related to this section may be interesting [how to throw custom exceptions](#throwing-custom-exceptions) or [how to work with environments](#nvironments).

## Returning a fixed value
As shown in the section 'How does it work?', it uses 'featureWithValue'. This returns the value 
provided in the configuration every time an annotated method for the given feature id is called.
Please notice that the object can be an instance of any custom class. However, it is recommended 
to verify that the return types between the annotated method and the custom behaviour can be casted in 
order to avoid a DimmerConfigurationException (which will be thrown due to the casting error).



As shown in the section 'How does it work?', it uses 'featureWithValue'. This returns the value 
provided in the configuration every time an annotated method for the given feature id is called.
Please notice that the object can be an instance of any custom class. However, it is recommended 
to verify that the object provided in the configuration can be casted to the type returned by the 
annotated method in order to avoid a DimmerConfigurationException (which will be thrown due to the casting error).

In other words, in order to the following code not throwing an DimmerConfigurationException(due to a casting error),
ClassB should inherit ClassA(or, if ClassA is an interface, ClassB is an implementation of ClassA).

```java
...

final ClassB fixedObject = new ClassB();

DimmerBuilder
    .local()
    .defaultEnvironment()
    .featureWithValue(FEATURE_NAME, fixedObject)
    .buildWithDefaultEnvironment();
...

@DimmerFeature(FEATURE_NAME)
public ClassA methodIWantToToggleOff() {
    ...
}
```
Code explanation:
- `.local()` indicates Dimmer to take the local configuration. In next releases will be possible to use remote configuration
- `.defaultEnvironment()` and buildWithDefaultEnvironment() are used when we don't want to deal with different environments.
- `.featureWithValue(FEATURE_NAME, object)` tells Dimmer to return the object passed as second parameter when a method annotated 
with `@DimmerFeature(FEATURE_NAME)` is invoked.

Related to this section may be interesting [how to work with environments](#nvironments).

## Behaviours: When exceptions and fixed values are not enough
Sometimes throwing an exception or returning a fixed value is not flexible enough. You may need to return a dynamic value 
or, maybe, in some situations you want to return a value, while in others you want to throw an exception. Lets say you need a more customizable behaviour. 
That's ok! Dimmer gives you all the flexibility you need via what we call 'behaviours'. 

Before providing a sample,lets clarify some concepts first.

- *FeatureInvocation:* It's an object that encapsulates the information regarding the invocation(signature, arguments, etc.). 
Its structure is:
```java
/** Feature covering invoked method */
private final String feature;
/** Invoked method's name */
private final String methodName;
/** Owner class of the method */
private final Class declaringType;
/** Returning type of the method */
private final Class returnType;
/** The arguments which the method was invoked with */
    private final Object[] args;
```

- *Behaviour:* As its name suggests, provides the capability to perform some dynamic actions for a given feature.
In Dimmer, a behaviour is Java 8 Function, which takes a FeatureInvocation as input parameter.


So, providing a behaviour to a feature looks like this:
```java
public class Main {

    private static final String FEATURE_NAME = "feature_name";

    public static void main(String... args) {

        //This function represents the behaviour which will replace the method annotated
        //with DimmerFeature.
        //What it does is, if the method argument is not null, it will transform it to BigDecimal and 
        //return it. Throws an IllegalArgumentException otherwise.
        final Function<FeatureInvocation, BigDecimal> behaviour =
                featureInvocation -> {
                    final Integer input = (Integer)featureInvocation.getArgs()[0];
                    if(input == null) {
                        throw new IllegalArgumentException("Input cannot be null");
                    }
                    return new BigDecimal(input);
                };

        DimmerBuilder
                .local()
                .defaultEnvironment()
                .featureWithBehaviour(FEATURE_NAME, behaviour)
                .buildWithDefaultEnvironment();

        final BigDecimal result = new Main().runFeaturedMethod(100);
        System.out.println(result);//It prints 100 as BigDecimal


        new Main().runFeaturedMethod(null);// Throws exception
    }

    //When this method is invoked, Dimmer will run the behaviour instead.
    @DimmerFeature(FEATURE_NAME)
    private BigDecimal runFeaturedMethod(Integer intValue) {
        return null;
    }

}
```
Code explanation:
- `.local()` indicates Dimmer to take the local configuration. In next releases will be possible to use remote configuration
- `.defaultEnvironment()` and buildWithDefaultEnvironment() are used when we don't want to deal with different environments.
- `.featureWithBehaviour(FEATURE_NAME, behaviour)` this line tells Dimmer to execute the function passed as second parameter(behaviour) when a method annotated 
with `@DimmerFeature(FEATURE_NAME)` is invoked, instead of the actual method. Basically it replaces the method invocation with the behaviour.

Related to this section may be interesting [how to work with environments](#nvironments).


## Throwing custom exceptions
We have seen how to throw a default exception(DimmerInvocationException), but sometimes you
may prefer to throw your own exception. That's still possible with Dimmer, however it 
needs to be an unchecked exception(inherits from RuntimeException) and fulfill  at least one 
of the following requirements:

- Provides a constructor which takes a FeatureInvocation as parameter
- Provides a default constructor with no parameters

In case that the class provides both constructors, Dimmer will prioritize the one that takes a FeatureInvocation as parameter 
over the default constructor.
```java
public class Main {

    private static final String FEATURE_NAME = "feature_name";

    public static class MyException extends RuntimeException {

        private final String feature;
        private final String methodName;

        public MyException(FeatureInvocation featureInvocation) {
            this.feature = featureInvocation.getFeature();
            this.methodName = featureInvocation.getMethodName();
        }

        @Override
        public String getMessage() {
            return String.format("Feature %s called via method %s", feature, methodName);
        }
    }

    public static void main(String... args) {
        DimmerBuilder
                .local()
                .defaultEnvironment()
                .featureWithException(FEATURE_NAME, MyException.class)
                .buildWithDefaultEnvironment();

        new Main().runFeaturedMethod();
        //It will throw exception and print message 'Feature feature_name called via method runFeaturedMethod'
    }

    @DimmerFeature(FEATURE_NAME)
    private String runFeaturedMethod() {
        return "real value";
    }

}
```
Code explanation:
- `.local()` indicates Dimmer to take the local configuration. In next releases will be possible to use remote configuration
- `.defaultEnvironment()` and buildWithDefaultEnvironment() are used when we don't want to deal with different environments.
- `.featureWithException(FEATURE_NAME, MyException.class)` tells Dimmer to throw an instance of the exception MyException whenever a 
method annotated with `@DimmerFeature(FEATURE_NAME)` is invoked.


Related to this section may be interesting [how to work with environments](#nvironments).

## Environments 
Because any project has to deal with environments, Dimmer supports it too.
The idea is that you configure all the possible environments you may have, and then you build it with the current one.

It looks lie this:
```java
public class Main {

    private static final String FEATURE_NAME = "feature_name";

    private static final String ENV1 = "e1", ENV2 = "e2", ENV3 = "e3", ENV_NOT_CONFIGURED = "e_n_c";

    //args[0] provides the environment where the application is running
    public static void main(String... args) {
        DimmerBuilder
                .local()
                .environments(ENV1, ENV2)//ENV1 or ENV2
                .featureWithDefaultException(FEATURE_NAME)
                .environments(ENV3)//ENV3
                .featureWithValue(FEATURE_NAME, "value for environment ENV3")
                .build(args[0]);

        new Main().runFeaturedMethod();
    }

    @DimmerFeature(FEATURE_NAME)
    private String runFeaturedMethod() {
        return "real value";
    }
}
```
Code explanation:
- `.defaultEnvironment()` and buildWithDefaultEnvironment() are used when we don't want to deal with different environments.
- with the line `.environments(ENV1, ENV2)` followed by `.featureWithDefaultException(FEATURE_NAME)` we tell Dimmer to throw a default exception, when ENV1 or ENV2
- with the line `.environments(ENV3)` followed by `.featureWithValue(FEATURE_NAME, "value for environment ENV3")` we tell Dimmer to return 'value for environment ENV3', if ENV3
- and what happens if it's running on ENV_NOT_CONFIGURED?... It just calls the real method, so it will return 'real value'


## Conditional toggling
We have seen that environments provide flexibility in an easy way, to load different configurations.
However, sometimes this is not enough. Sometimes you want your feature depending on a dynamic property, instead of an static environment. Or maybe both.
To solve this, in all the methods to configure features(featureWithValue, featureWithBehaviour, etc.) provided by the builder you can
add a boolean flag which indicates if you want to provide the given behaviour to the feature(flag is true) or you just want to ignore it(flag is false)
```java
public void dimmerConfiguration(boolean toggledOff) {
    DimmerBuilder
        .local()
        .defaultEnvironment()
        .featureWithDefaultException(toggledOff, FEATURE_NAME)//Taken into account only if toggledOff is true
        .buildWithDefaultEnvironment();
}
```

## Logging
Dimmer uses slf4j to perform logging. If the importer project does not use slf4j or does not provide a binding implementation,
Dimmer(slf4j actually) will print some warnings. While this does not stop the application or Dimmer from working, is highly recommended
to use slf4j as wrapper framework for logging.


## Known issues
### Aspectj libraries and Lombok project don't work well together in IDEs
There is a known issue in IDEs like intellij when using any aspectj library together with Lombok. However, while the application can be run 
without any issue, the IDE won't compile properly, so you cannot debug your application in your IDE, for example. 
This is not something affecting only to Dimmer, is an issue between Aspectj and Lombok.

Workarounds:
- Create a submodule with all the classes that use lombok, compile it and the bringing to the project. It can be a maven/gradle submodule.
- Similar to the previous one, but in this case, instead of creating a submodule, just having the classes that use Lombok in a separated package and tell Intellij to compile with a different compiler. 
- Taking advantage of the full compatibility of Kotlin with Java, use Kotlin instead of Java for those classes using Lombok. Everything Lombok provides, Kotlin does too natively and easier.

## LICENSE
This project is licensed under the [GNU Global Public License](https://www.gnu.org/licenses/gpl-3.0.en.html), Version 3.0 - see the [LICENSE](./LICENSE.md) file for details