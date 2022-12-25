---
layout: post
title:  "Dependency Injection, An Introduction"
tags: ['DEPENDENCY INJECTION']
---

Dependency injection is a pattern where dependencies required for a callee are provided by the caller (often a framework). This could be as straightforward as passing the dependencies as arguments to a function. Consider the following example

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
        case employeeRoles.DIRECTOR :{
            return employeeDetails.getBasicPay() +
                employeeDetails.getTravelAllowance() +
                employeeDetails.getTransportationAllowance()
                employeeDetails.getHouseRentAllowance() -
                employeeDetails.getTax()
        }
        case employeeRoles.MANAGER : {
            return employeeDetails.getBasicPay() +
                employeeDetails.getTravelAllowance() +
                employeeDetails.getHouseRentAllowance() -
                employeeDetails.getTax()
        }
        default : {
            return employeeDetails.getBasicPay() - 
                employee.getTax()
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
