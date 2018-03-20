---
layout: post
title: Parallel JUnit Test Execution
---

JUnit test parallel execution can bring down the test execution time considerably. The only condition is that your test cases must be independent. The state shall not be shared across test cases. Today, I shall share a maven project with parallel test execution configured using maven surefire plugin.

The complete source code can be found in [Github.](https://github.com/sanjuthomas/junit-parallel-execution/tree/master/junit-parallel-execution)
