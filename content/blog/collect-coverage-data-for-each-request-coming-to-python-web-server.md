---
title: "Getting code coverage data for each request coming to a python web server"
date: 2024-04-01T21:46:28+05:30
draft: true
---

In this blog, we will demonstrate how to get the coverage data for each incoming request on a python web server built using any web framework.

### What is Code Coverage ?

Code coverage is a metric used in software testing to measure the extent to which the source code of a program has been executed during testing. It indicates the percentage of code that has been covered by the test cases. Code coverage analysis helps developers understand how thoroughly their tests exercise the codebase.

Code coverage tools are used to collect data on code execution during testing and generate reports showing the coverage metrics. These reports help developers identify areas of the code that are not adequately covered by tests, allowing them to write additional tests to improve coverage and increase confidence in the software's correctness and reliability.

### What does it mean to get coverage data for each request ?

Obtaining coverage data for each request coming to a web server offers several benefits:

1. **Granular Insights**: By capturing coverage data for each request, developers gain detailed insights into which parts of the codebase are executed in response to different types of requests. This level of granularity allows for a deeper understanding of the application's behavior under various conditions.
    
2. **Identifying Untested Code Paths**: Coverage data helps identify areas of the code that are not adequately covered by tests. By analyzing coverage reports, developers can pinpoint specific code paths that need additional testing, ensuring comprehensive test coverage across the entire codebase.
    
3. **Getting coverage data for e2e test suites:** e2e test suites can be run against the modified application with capturing logic for each request in place.
    

### Obtaining coverage data for servers built with different web frameworks

To obtain the coverage data, we would be using the coverage.py library. coverage.py is mostly used through CLI. But it provides API to use it programmatically.

We will define a middleware through which every incoming request would pass. In our "coverage" middleware before passing control to other parts of our application, we will call `start` function from coverage library. Coverage measurement is only collected in functions called after `start()` function is invoked, so if this middleware is scheduled to run first then coverage of other middleware would also be captured along with main application code.

Once the application returns then we would stop collecting coverage data. We can then fetch the data and further process it.

The below is a code snippet for the coverage middleware which can be used in servers built using django web framework:

![coverage-middleware](/coverageMiddleware.png)

The code for middleware function for other framework is highly similar, only the function signature differs depending on the framework used.

This is how with very little code change you can collect coverage data for each incoming requests and prioritise increasing coverage for most frequent requests.