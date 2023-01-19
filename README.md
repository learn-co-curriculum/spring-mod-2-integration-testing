# Integration Testing

## Learning Goals

- Create integration tests in Spring.

## Introduction

The time has come for us to step up our testing game and try our hand at
integration testing. **Integration testing** is a type of testing that looks at
multiple components of a program and tests how well these modules may work
together. The components that are involved in integration testing are usually
unit tested already with a goal to see if any issues are exposed when they are
integrated together. This phase of testing will start to bring in some Spring
framework functionality.

## Code-Along: Add Integration Tests

### Controller Integration Tests

We will start with basic integration testing, which will allow us to test
whether our endpoints are properly exposed, but will stop short of testing
**all** the infrastructure that the Spring Framework provides.

Let's start by creating a new test class for the `DemoController`. This time, we
will add `Integration` to the name of the test class to differentiate it from
our previous `UnitTest` class. We can generate this new class just as we did
before by right-clicking in the blank space within the `DemoController` class
--> Choosing "Generate..." --> Then "Test...":

```java
package com.example.springtestingdemo.controller;

import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class DemoControllerIntegrationTest {

    @Test
    void hello() {
    }

    @Test
    void getCatFact() {
    }
}
```

Now for this test, we want to actually test whether the endpoints we have
defined work properly. To do this, we'll add the annotation, `@WebMvcTest`, to
the class definition:

```java
package com.example.springtestingdemo.controller;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;

import static org.junit.jupiter.api.Assertions.*;

@WebMvcTest(DemoController.class)
class DemoControllerIntegrationTest { }
```

The `@WebMvcTest` annotation asks the Spring Framework to initialize its web
context **only** and to include just the specified controller. This is helpful
for a couple of reasons:

1. It does not initialize other aspects of the Spring Framework, like database
   connections.
2. It does not initialize other controllers.

In a real-life application, we may have a large number of controllers and other
aspects of the Spring Framework associated within the application context. This
will narrow it down to solely focus on the controller we are testing within this
integration test.

With the `@WebMvcTest` annotation, we will get a bean that can be autowired by
Spring to provide us access to a `MockMvc` instance that we can then use to make
actual HTTP calls to our endpoints. Let's go ahead and add an autowired
`MockMvc` instance in our integration test:

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.junit.jupiter.api.Assertions.*;

@WebMvcTest(DemoController.class)
class DemoControllerIntegrationTest {

   @Autowired
   private MockMvc mockMvc;
   
   // other methods
}
```

Let's put the `MockMvc` instance to use by implementing the `hello()` test
method:

```java
package com.example.springtestingdemo.controller;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.junit.jupiter.api.Assertions.*;
import static org.hamcrest.Matchers.equalTo;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(DemoController.class)
class DemoControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void hello() throws Exception {
        mockMvc.perform(get("/hello"))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo("Hello World")));
    }
    
    // test getCatFact method
}
```

Notice that we are chaining multiple calls on the `mockMvc` instance we created.
This makes it easier and more readable than assigning the return value of each
method call and then using that value to make the next method call. Let's break
down this chain of calls:

- The `perform()` method lets us pass in an HTTP verb, along with the
  appropriate parameters for that call. In this case, we are asking for a GET
  request to be executed. We'll pass in the URL to which it should be submitted.
  Spring will then take that URL and find the endpoint we have defined in the
  `hello()` method of our controller. Note that the test method, `hello()` now
  throws an `Exception`. This is due to the fact that the `perform()` method
  can throw an `Exception`.
- The `andDo(print())` will print the results of the `perform()` method to the
  console. We can use this to diagnose potential issues.
- The `andExpect(status().isOk())` call tells `mockMvc` that we want an HTTP
  status code of 200 to be returned as a result of the `perform()` method call.
- The `andExpect(content().string(equalTo("Hello World)))` tells `mockMvc` that
  we want the content of the response to be equal to a `String` with a value of
  "Hello World".

To summarize, we are asking the `mockMvc` to hit the endpoint `/hello` and see
if the response is what we are expecting - which in this case is a message
saying "Hello World".

Let's try running our `DemoControllerIntegrationTest` class now and see what
happens.

Uh oh. It looks like something failed. If we scroll down in the console, we will
see an error that looks like this:

`Parameter 0 of constructor in com.example.springtestingdemo.controller.DemoController required a bean of type 'com.example.springtestingdemo.service.CatFactService' that could not be found.`

We even see a recommended action for us to take to resolve this issue:

```text
Action:

Consider defining a bean of type 'com.example.springtestingdemo.service.CatFactService' in your configuration.
```

Hmm... it seems like it is looking for the `CatFactService` in its limited
application context that we defined with the `@WebMvcTest`. Even though the
`/hello` endpoint does not require a `CatFactService` instance, it is still
necessary to have as a bean in its application context since the
`DemoController` needs one to run properly. This is the case for another mock
object! But instead of using Mockito, we'll take advantage of the `@MockBean`
annotation. This annotation will not only create a mock object of the service,
but it will also add it to the application context:


```java
package com.example.springtestingdemo.controller;

import com.example.springtestingdemo.service.CatFactService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.web.servlet.MockMvc;

import static org.junit.jupiter.api.Assertions.*;
import static org.hamcrest.Matchers.equalTo;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(DemoController.class)
class DemoControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private CatFactService catFactService;

    @Test
    void hello() throws Exception {
        mockMvc.perform(get("/hello"))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo("Hello World")));
    }
    
    // test getCatFact method
}
```

Now if we run the integration test it should pass!

Let's work on implementing the `getCatFact()` test method now:

```java
package com.example.springtestingdemo.controller;

import com.example.springtestingdemo.dto.CatFactDTO;
import com.example.springtestingdemo.service.CatFactService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;

import static org.junit.jupiter.api.Assertions.*;
import static org.hamcrest.Matchers.equalTo;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(DemoController.class)
class DemoControllerIntegrationTest {

  @Autowired
  private MockMvc mockMvc;

  @MockBean
  private CatFactService catFactService;

  @Test
  void hello() throws Exception {
    mockMvc.perform(get("/hello"))
            .andDo(print())
            .andExpect(status().isOk())
            .andExpect(content().string(equalTo("Hello World")));
  }

  @Test
  void getCatFact() throws Exception {
    CatFactDTO catFact = new CatFactDTO();
    String fact = "In ancient Egypt, when a family cat died, " +
            "all family members would shave their eyebrows as a sign of mourning.";
    catFact.setFact(fact);
    when(catFactService.getFact()).thenReturn(catFact);

    MvcResult response = mockMvc.perform(get("/cat-fact"))
            .andDo(print())
            .andExpect(status().isOk())
            .andReturn();

    ObjectMapper objectMapper = new ObjectMapper();
    CatFactDTO catFactResponse =
            objectMapper.readValue(response.getResponse().getContentAsString(), CatFactDTO.class);

    assertEquals(catFact, catFactResponse);
  }
}
```

Here, we define again, a hardcoded value of what we want the mock
`CatFactService` to return when we call the `getFact()` method. Then, just like
we did above, we'll have the `mockMvc` hit the `/cat-fact` endpoint. But this
time, we'll have it return the response with the `.andReturn()` method call.
this will give us a `MvcResult`. From there, we'll grab the content and use the
`ObjectMapper` to read the value from the JSON that we got back in response.
Once we convert the JSON back to the `CatFactDTO`, we can assert the two objects
to ensure that they both are equal.

If we run this test, it should also pass!

### Service Integration Tests

We should have now noticed that we still have a potential gap, which is that
all the tests we've written so far could pass even if the `getFact()` method
on the `CatFactService` never actually interfaced with the cat fact API.

Before we generate a test for this class, make sure to un-comment the code we
commented out previously in the last lesson within the `CatFactService` class:

```java
    public CatFactDTO getFact() {
        CatFactDTO catFact = restTemplate.getForObject(FACT_URI, CatFactDTO.class);
        log.info("Retrieved cat fact from external source. Returning the DTO");
        return catFact;
    }
```

Let's create an integration test of the service class. We can generate this new
class just as we did before by right-clicking in the blank space within the
`CatFactService` class --> Choosing "Generate..." --> Then "Test...". We'll call
this class `CatFactServiceIntegrationTest`. Once the class has been generated,
it should look like this:

```java
package com.example.springtestingdemo.service;

import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class CatFactServiceIntegrationTest {

    @Test
    void getFact() {
    }
}
```

Let's rename the test `getFact()` method to something a little more descriptive,
like "getFactFromExternalApi":

```java
    @Test
    void getFactFromExternalApi() {
    }
```

Now we'll implement it to call the `CatFactService`, which in turn, will call
the external cat fact API. This gets tricky since the cat facts are random. We
cannot test for a specific value back from the service, so we will do two things
instead:

1. Test that the return value is not null.
2. Make sure that we don't get the same fact on two consecutive calls, ensuring
   that the fact is indeed "random".

```java
package com.example.springtestingdemo.service;

import com.example.springtestingdemo.dto.CatFactDTO;
import org.junit.jupiter.api.Test;
import org.springframework.web.client.RestTemplate;

import static org.junit.jupiter.api.Assertions.*;

class CatFactServiceIntegrationTest {

    @Test
    void getFactFromExternalApi() {
        CatFactService catFactService = new CatFactService(new RestTemplate());
        CatFactDTO firstCatFact = catFactService.getFact();
        assertNotNull(firstCatFact);
        CatFactDTO secondCatFact = catFactService.getFact();
        assertNotNull(secondCatFact);
        assertNotEquals(firstCatFact, secondCatFact);
    }
}
```

Note: With a true random service, it's possible that the same return value could
be returned from 2 consecutive calls. If we wanted to guard against this, we
could conditionally make a third call if the 2 first calls resulted in the same
value being returned. The more consecutive calls we make, the less likely they
are to all return the same value.

If we run the `CatFactServiceIntegrationTest`, we should see it passes.
Note: This is not a "unit" test because it actually lets the real service (not
mocked) make a request to the real API and tests the actual response (albeit not
the actual precise value, for the reasons we discussed).

## Code Check

Check the project structure and code in each class to ensure your code matches
what was covered in this lesson.

### Project Structure

```text
├── HELP.md
├── mvnw
├── mvnw.cmd
├── pom.xml
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── example
    │   │           └── springtestingdemo
    │   │               ├── SpringTestingDemoApplication.java
    │   │               ├── controller
    │   │               │  └── DemoController.java
    │   │               ├── dto
    │   │               │  └── CatFactDTO.java
    │   │               └── service
    │   │                   └── CatFactService.java
    │   └── resources
    │       ├── application.properties
    │       ├── static
    │       └── templates
    └── test
        └── java
            └── org
                └── example
                    └── springtestingdemo
                        ├── SpringTestingDemoApplicationTests.java
                        ├──controller
                        │  ├──DemoControllerIntegrationTest.java
                        │  └──DemoControllerUnitTest.java
                        └── service
                            └── CatFactServiceIntegrationTest.java
```

### DemoControllerIntegrationTest.java

```java
package com.example.springtestingdemo.controller;

import com.example.springtestingdemo.dto.CatFactDTO;
import com.example.springtestingdemo.service.CatFactService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;

import static org.junit.jupiter.api.Assertions.*;
import static org.hamcrest.Matchers.equalTo;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(DemoController.class)
class DemoControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private CatFactService catFactService;

    @Test
    void hello() throws Exception {
        mockMvc.perform(get("/hello"))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo("Hello World")));
    }

    @Test
    void getCatFact() throws Exception {
        CatFactDTO catFact = new CatFactDTO();
        String fact = "In ancient Egypt, when a family cat died, " +
                "all family members would shave their eyebrows as a sign of mourning.";
        catFact.setFact(fact);
        when(catFactService.getFact()).thenReturn(catFact);

        String expected = "{\"fact\":\"" + fact + "\"}";

        MvcResult response = mockMvc.perform(get("/cat-fact"))
                .andDo(print())
                .andExpect(status().isOk())
                .andReturn();

        assertEquals(expected, response.getResponse().getContentAsString());
    }
}
```

### CatFactServiceIntegrationTest.java

```java
package com.example.springtestingdemo.service;

import com.example.springtestingdemo.dto.CatFactDTO;
import org.junit.jupiter.api.Test;
import org.springframework.web.client.RestTemplate;

import static org.junit.jupiter.api.Assertions.*;

class CatFactServiceIntegrationTest {

    @Test
    void getFactFromExternalApi() {
        CatFactService catFactService = new CatFactService(new RestTemplate());
        CatFactDTO firstCatFact = catFactService.getFact();
        assertNotNull(firstCatFact);
        CatFactDTO secondCatFact = catFactService.getFact();
        assertNotNull(secondCatFact);
        assertNotEquals(firstCatFact, secondCatFact);
    }
}
```

## Conclusion

As we saw in this lesson, integration testing tests the functionality once
two or more components are integrated. For example, we saw in the unit
testing lesson that we purely tested the controller's methods without
considering the endpoints. In the `DemoControllerIntegrationTest` class, we
considered the user actually hitting the endpoint to call the method that we
unit tested prior. The goal to this type of testing is to ensure that
everything still works as expected and properly once integrated.

## Resources

- [WebMvcTest Annotation Documentation](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/test/autoconfigure/web/servlet/WebMvcTest.html)
- [MockBean Annotation Documentation](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/test/mock/mockito/MockBean.html)
- [Baeldung: Mockito.mock vs @Mock vs @MockBean](https://www.baeldung.com/java-spring-mockito-mock-mockbean)
