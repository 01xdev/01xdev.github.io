---
layout: post
title:  "Dependency Injection, An Introduction"
tags: ['DEPENDENCY INJECTION']
---

Dependency injection is a pattern where dependencies required for a callee are provided by the caller( often a framework). This could be as simple as passing the dependencies as arguments to a function. Consider the following example

```java
    Double calculateSalary(Employee employee){
        return employee.getBasicPay();
    }
```
