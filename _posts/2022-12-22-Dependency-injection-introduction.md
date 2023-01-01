---
layout: post
title:  "Dependency Injection, An Introduction"
tags: ['DEPENDENCY INJECTION']
---

Dependency injection is a pattern where dependencies required for a callee are provided by the caller (or a framework). This could be as straightforward as passing the dependencies as arguments to a function. Consider the following example

```java
class PayRoll{
    public double calculateSalary(EmployeeDetails employeeDetails){
        // calculate total salary
    }
}
```
The `EmployeeDetails` object required by  `calculateSalary` should be provided by the caller. We call such types of Dependency Injection  **Method Injection**.

We can also maintain `employeeDetails` as a field of `Payroll` class. In such cases, the value for `employeeDetails` should be provided either by a constructor or by a setter function.

### Constructor Injection

`employeeDetails` is passed as a parameter to the constructor

```java
class PayRoll{
    private EmployeeDetails employeeDetails;

    Payroll(EmployeeDetails employeeDetails){
        this.employeeDetails = employeeDetails;
    }

    public double calculateSalary(){
        // calculate total salary
    }
}
```


### Setter Injection
`employeeDetails` is passed as a parameter to the `setEmployeeDetails` method. This method should be called before calling `calculateSalary`.

```java
class PayRoll{
    private EmployeeDetails employeeDetails;

    public void setEmployeeDetails(EmployeeDetails employeeDetails){
        this.employeeDetails = employeeDetails;
    }

    public double calculateSalary(){
        // calculate total salary
    }
}
```
---
## Dependency Inversion Principle(DIP)

Though often confused with  Dependency Injection, the Dependency Inversion Principle has distinct differences from Dependency Injection.

A formal defenition of DIP can be derived from the following statements

* High-level policies should not depend on low-level details
* The low-level details should be hidden behind Domain-Relevant abstractions
* Abstractions should not depend on details 

Consider the following implementation of `calculateSalary`

```java
public double calculateSalary(EmployeeDetails employeeDetails){
    switch(employeeDetails.getRole()){
        case DIRECTOR :{
            return employeeDetails.getBasicPay() +
                employeeDetails.getTravelAllowance() +
                employeeDetails.getTransportationAllowance()
                employeeDetails.getHouseRentAllowance() -
                employeeDetails.getTax()
        }
        case MANAGER : {
            return employeeDetails.getBasicPay() +
                employeeDetails.getTravelAllowance() +
                employeeDetails.getHouseRentAllowance() -
                employeeDetails.getTax()
        }
        default : {
            return employeeDetails.getBasicPay() - 
                employeeDetails.getTax()
        }
    }
}
```

The salary calculation depends on the role of an Employee. If the employee is a Director, apart from basic pay, he is entitled to travel allowance, transportation allowance and house rent allowance. For a manager, the allowances include a travel allowance and house rent allowance. Tax has to be deducted from the total of basic pay and allowance.

Should the `calculateSalary` method be burdened with this much information? Afterall, the high-level policy for salary calculation seems to be pretty straightforward :

```
total salary = basic pay + allowances - deductions
```

If we are to follow DIP, the low-level details of obtaining allowances and deductions based on employee roles should be abstracted away from the high-level policy.

The following `Employee` interface could be a good candidate for such an abstraction

```java
interface Employee{
    public double getBasicPay();
    public double getTotalAllowances();
    public double getTotalDeductions();
}
```

The interface, though named `Employee`, does not provide access to all the details of the employee.This might seem a bit odd, since objects in Object Oriented Programming has to be modelled against real-world objects. Well, it need not always be the case. Objects should be modeled considering the domain. In the context of a payroll application, we are only interested those attributes of an employee that can affect his salary. Hence we may omit all other details.

To make it evident that this model of an employee is relevant only to the payroll domain, we may choose to place the `Employee` interface and its implementations within the payroll package like so :

```
com.companyname.payroll.Employee
```

We can now have two seperate implementations of `Employee` for Director and Manager

```java
public class Manager implements Employee {

    private EmployeeDetails employeeDetails;

    //Constructor Injection
    Manager(EmployeeDetails employeeDetails) {
        this.employeeDetails = employeeDetails;
    }

    @Override
    public double getBasicPay() {
        return employeeDetails.getBasicPay();
    }

    @Override
    public double getTotalAllowances() {
        return employeeDetails.getTravelAllowance() +
                employeeDetails.getHouseRentAllowance();
    }

    @Override
    public double getTotalDeductions() {
        return employeeDetails.getTax();
    }
}
```
The dependency(`EmployeeDetails`) required for calculating salary is wrapped within the `Manager` class. We depend on **Constructor Injection** to obtain this dependency.

The `Director` class would also have a similar implementation, except for the `getTotalAllowances` method, which would be implemented like so:

```java
   @Override
    public double getTotalAllowances() {
        return employeeDetails.getTravelAllowance() +
                employeeDetails.getHouseRentAllowance() +
                employeeDetails.getTransportAllowance();
    }
```

Now that we have moved the low-level details into implementations of `Employee`, the `calculateSalary` method can be refactored to focus more on high-level policy.

```java
public double calculateSalary(EmployeeDetails employeeDetails){
    Employee employee = null;
        
    // plugging in low-level details based on employee role
    switch(employeeDetails.getRole()){
        case DIRECTOR : employee = new Director(employeeDetails);
        break;
        case MANAGER : employee = new Manager(employeeDetails);
    }
        
    // high-level policy
    return employee.getBasicPay() +
            employee.getTotalAllowances() -
            employee.getTotalDeductions() ;
}
```
Based on the employee role, we select which concrete implementation of `Employee` to use. Notice that we used **constructor injection** to provide `employeeDetails` to `Director` and `Manager`. After obtaining an instance of `Employee`, salary can be computed as :

```
employee.getBasicPay() + employee.getTotalAllowances() - employee.getTotalDeductions()
```
Perfect! our `calculateSalary` method is now compliant to most of the dictations of DIP

* The low-level details are abstracted away from the high-level policy
* The low-level details are hidden in a domain-relevant abstraction(Employee)
* The abstraction(Employee) does not depend on details - As long as the Director and Manager classes honor the interface it implements, we are not bothered about the innards of these classes

But wait! Doesn’t the switch statement in `calculateSalary` hint a code-smell ? Apart from violating the **Open-Close Principle** (We will have to modify the method every time a new employee role gets added to the system), it also raises another question - Should the `calculateSalary` method be concerned about how the concrete implementations of `Employee` are obtained? Is there, by any chance, a way to abstract these details into a module. Could this module provide provisions to  **inject** the required implementation of `Employee` to `calculateSalary`?

Well,Its the **Simple Factory Pattern** that you are looking for.

---
## Simple Factory Pattern

Simple factory pattern is a creational design pattern. It decides what object to build based on the conditions given.

A simple factory for employee can be constructed as follows:

```java
public class EmployeeFactory {

    // Return concrete implementation based on role
    public static Employee getFrom(EmployeeDetails employeeDetails){
        switch(employeeDetails.getRole()){
            case MANAGER : return new Manager(employeeDetails);
            case DIRECTOR: return new Director(employeeDetails);
            default: throw new NoImplementationFoundException();
        }
    }

    // A custom exception to throw, in case a concrete implementation
    // cannot be obtained
    public static class NoImplementationFoundException extends RuntimeException{
        NoImplementationFoundException(){
            super("No implementation found");
        }
    }
}

```

An instance of `Employee` can be obtained as :
```java
EmployeeFactory.getFrom(employeeDetails)
```

Now, the `calculateSalary` method can be refactored as:

```java
public class Payroll {
    public double calculateSalary(EmployeeDetails employeeDetails){
        
        Employee employee = EmployeeFactory.getFrom(employeeDetails);
        
        return employee.getBasicPay() +
                employee.getTotalAllowances() -
                employee.getTotalDeductions() ;
    }
}
```

As you can see, we have delegated the creation of `Employee` objects to the
`EmployeeFactory`.We can now call the `EmployeeFactory` to resolve the correct dependency required for `calculateSalary` based on `employeeDetails`.  

The overall design of our system would look like this:


![](https://drive.google.com/uc?export=view&id=1ZdS3NdYSaGJCdaHM7WV5qoykZe27xZSy)

---

**What do all these have to do with Dependency Injection?** You might ask. Asking a simple factory for an object does not seem like dependency injection. Moreover, it does not conform with any of the categories(constructor, setter, and method injections) of the dependency injection we discussed in the introduction. Well, Injecting dependencies is not the only responsibility of a Dependency Injection library. A part of its duty is to manage the dependencies for us, which the Simple Factory pattern is good at. As we have seen in the previous example, the simple factory pattern took the responsibility of creating objects away from the caller. Now the `calculateSalary` method need not be concerned about the employee roles and the implementation details of each of these roles. A new role can be added or an existing role can be edited without modifying `calculateSalary`.

The whole exercise aimed to point out the need for dependency management coming up naturally during the course of refactoring. As with the simple factory, we will shortly look into a different version of this pattern, the `Bean Factory`, which would help us to create dependency injections belonging to the categories discussed in the introduction.