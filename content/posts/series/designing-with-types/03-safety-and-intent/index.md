---
title: "Designing with types #03: Safety and Intent"
date: 2025-07-09T09:38:34Z
draft: false
tags: [ design, "type safety", java, kotlin, OO ]
series: "Designing with Types"
cover:
  image: "cover.png"
---

{{< summary >}}
In the previous installment of this series, we concluded that we can write code that is **safe** and that **reveals
intent** by making illegal states unrepresentable and by encapsulating state changes. Today, we'll try to come up with
proper definitions of safety and revealing intent. Also, we'll take a look at more examples of applying these principles
in practice.
{{< /summary >}}

<!--more-->

## Safety

What does **safety** mean in the context of software development in general and designing types in particular? Here's my
attempt at a "formal" definition:

{{< summary >}}
We consider an API to be **safe** if and only if **everything that's public** can be used **freely**, without the risk
of introducing an illegal state.
{{< /summary >}}

With this definition I'm trying to stay as language agnostic as possible. Reason for this is that the concept of safety
can be applied (to some extent) to every programming language and every paradigm.

In this series, we're focusing mainly on strong-typed, object-oriented languages like Kotlin and Java. In those
languages **everything that's public** is either a public class, a public method, a public field or (in Kotlin) a public
top-level function. To be more specific, this definition applies to everything that causes a state change. Things like
read-only fields or pure functions are always safe according to this definition, because those things don't modify
anything.

So what does this mean in practice for things that do modify something? Let's look at each case individually.

- **Public mutable fields** can only be freely mutable if there exist **zero** situations where a possible
  value of such a field can be considered invalid. If this is not the case, a field cannot be publicly mutable according
  to this definition.
- **Public methods and top-level functions** that change state may **never** change that state to something invalid.
  According to this definition, such methods or functions must either:
    - change the state to something valid;
    - throw an exception;
    - do nothing;
    - or return a value that indicates a failure.

## Revealing intent

Code is written once, but it's read many times. Therefore we want the code to be easy to understand. For this article
series, I'll use a pretty narrow definition of revealing intent:

{{< summary >}}
Code that reveals intent is **explicit** about what it **does**, what it **expects** and what it **results** in.
{{< /summary >}}

I don't think this definition needs much of an explanation. It doesn't just apply to code. Take a
typical [use case template](https://en.wikipedia.org/wiki/Use_case#Examples) for example. Such a template will typically
have room for us to document the goal, the preconditions and the postconditions. In order to properly understand a piece
of functionality, this is what we typically need to know, regardless of whether we're reading code or a use case.

This series is about designing with types, so here we'll be focusing on revealing intent using types. As we'll see,
types are a very powerful way of revealing intent. This is because types allow us to give important concepts a proper
**name** and because by defining types, we can leverage the **power of the compiler** to enforce intended usage.

## Examples

With the principles from the [previous article](../02-pitfalls-and-practices) in mind, let's look at some practical
examples. Every example will start by reviewing a piece of code, evaluating whether it's safe and if it reveals intent.

### Properties

#### Code review

Let's look at this `Customer` class:

```kotlin
class Customer(
    var id: Long,
    var name: String,
    var emailAddress: String,
)
```

There's no context in this example, but it's not hard to see that this code probably isn't safe:

- The `id` field is publicly mutable, but it's generally not recommended to change the primary key of an entity.
- The `id` field is a plain `Long`, but negative values are probably not allowed.
- The `name` and `emailAddress` fields are plain `String`s, but it's highly unlikely that every possible `String`
  is a valid value for these fields.

Let's assume that in this case, it's safe that `name` and `emailAddress` are publicly mutable.

The code does reveal intent to some degree, but it's the bare minimum. It uses helpful names for the properties, but
the property types don't reveal any intent at all. The constraints for the `id`, `name` and `emailAddress` are not
obvious and it's not clear where we can find those constraints.

#### Refactoring

The first and most obvious thing we can do to improve the safety of this code, is to make `id` immutable. By doing this,
we eliminate one way to introduce an illegal state.

```kotlin {hl_lines=["2"]}
class Customer(
    val id: Long,
    var name: String,
    var emailAddress: String,
)
```

We can make `id`, `name` and `emailAddress` safe by
introducing [Value Objects](../02-pitfalls-and-practices#value-object-pattern).

```kotlin
class Customer(
    val id: CustomerID,
    var name: CustomerName,
    var emailAddress: EmailAddress,
)

data class CustomerID(private val value: Long) {
    init {
        require(value >= 0L)
    }
}

data class CustomerName(private val value: String) {
    init {
        require(value.isNotBlank())
    }
}

data class EmailAddress(private val value: String) {
    init {
        require(isValidEmailAddress(value)) // let's pretend we have this function available somewhere 😉
    }
}
```

Just like in the previous article, we can see the power of the Value Object pattern. With one refactoring, we made two
improvements:

1. The code is now safe. There is simply no way we can write compiling code that introduces an illegal state. The
   code will either not compile or we'll get an exception for trying to introduce an illegal state. We cannot
   accidentally swap `name` and `emailAddress`, because that code won't compile:
   ```kotlin
   val customer = Customer(id = CustomerID(0), name = EmailAddress("name@example.com"), emailAddress = CustomerName("John Johnsson"))
   ```
   We cannot give `id` an invalid value, because that will throw an exception:
   ```kotlin
   val id = CustomerID(-1)
   ```
2. The code reveals intent much more clearly. The constructor arguments of `Customer` all have a type with a proper
   name, which makes it obvious that acceptable values for those arguments belong to a certain domain. It's also easy to
   find the constraints for `id`, `name` and `emailAddress`. We simply use our IDE to navigate to the type definitions
   of the corresponding Value Objects and we'll have all the information we need.

### Simple business rules

#### Code review

Here's an `Order` class:

```kotlin
class Order(
    val id: OrderID,
    val customerId: CustomerID,
    var status: OrderStatus,
    var paymentId: PaymentID?,
)
```

Here's a service that makes sure that orders that have not been payed cannot be completed:

```kotlin
class OrderService {
    fun completeOrder(order: Order) {
        if (order.paymentId != null)
            order.status = OrderStatus.COMPLETED
    }
}
```

The `Order` class uses Value Objects and `id` and `customerId` are immutable, so it's pretty safe, except that
`status` is publicly mutable. We can easily create an order with status `OrderStatus.COMPLETED` while it has not been
payed. We just circumvent `OrderService` and call the `Order` constructor directly:

```kotlin
// This code compiles and throws no exceptions
val order = Order(OrderID(1), CustomerID(2), OrderStatus.COMPLETED, null)
```

The code above communicates intent via `OrderService`, but `Order` and `OrderService` are different classes which are
usually defined in different source files, which could belong to different packages. It would be clearer if the data and
the business rule were close together.

#### Refactoring

We can improve this by introducing more encapsulation. We do this by merging `OrderService` and `Order`, so we can make
the setters for `status` and `paymentId` private.

```kotlin
class Order(val id: OrderID, val customerId: CustomerID) {
    var status: OrderStatus = OrderStatus.PENDING
        private set
    var paymentId: PaymentID? = null
        private set

    fun complete() {
        if (paymentId != null)
            status = OrderStatus.COMPLETED
        else
            throw IllegalStateException("Cannot complete an order that has not been payed")
    }
}
```

With this refactoring we made multiple improvements:

- The code is now safe, because all state changes changes are encapsulated inside the `Order` class. This doesn't
  compile, because of the private setter for `status`:
  ```kotlin {hl_lines=["2"]}
  val order = Order(OrderID(1), CustomerID(2))
  order.status = OrderStatus.COMPLETED // <- Compiler error
  ```
  And this raises an exception:
  ```kotlin {hl_lines=["2"]}
  val order = Order(OrderID(1), CustomerID(2))
  order.complete() // <- BOOM!
  ```
- Intent is clearer, because the data and the business rule are defined in one class instead of two.
- Intent is also more clear, because the `Order` constructor contains only fields that are relevant for creating an
  order in the initial state. All state changes are done using methods like `complete` (I omitted other methods for
  brevity).

### Statuses

#### Code review

Coming back to the `Order` class, let's zoom in on the order status:

```kotlin
class Order {
    var status: OrderStatus = OrderStatus.PENDING
        private set
}
```

What can we say about such a small piece of code? Well, first of all it looks pretty safe, because the `status` field
has a private setter. What else? Let's look at how we would instantiate this class:

```kotlin
val order = Order()
```

We've lost some information compared to the class definition. From this constructor invocation alone, it's not obvious
what the status of the newly created order is. This is a limitation of constructors in general, because we can't give
constructors a descriptive name.

#### Refactoring

We can improve the communication of intent by introducing a factory method.

```kotlin
class Order(status: OrderStatus) {
    var status: OrderStatus = status
        private set

    companion object {
        fun createPending(): Order = Order(OrderStatus.PENDING)
    }
}
```

Now, we can instantiate a new `Order` using the `createPending` factory:

```kotlin
val order = Order.createPending()
```

This is clearly an improvement. By using the factory we are explicit about the state of a newly created `Order`.
Unfortunately, there's a downside: We now have a public constructor that accepts any `OrderStatus`. We have potentially
introduced the same safety issue we addressed in the previous example.

Fortunately, the solution is simple. We'll just make the constructor `private`:

```kotlin {hl_lines=["1"]}
class Order private constructor(status: OrderStatus) {
    var status: OrderStatus = status
        private set

    companion object {
        fun createPending(): Order = Order(OrderStatus.PENDING)
    }
}
```

Now, the design is safe again. There's only one way to create a new `Order`, which is via the `createPending` factory.

## Conclusion

We identified two requirements and two design principles to help us write code that is robust and maintainable. The
requirements are:

{{< summary >}}

We should design API's so that

- they are always **safe** to use;
- they **reveal intent**.

{{< /summary >}}

We can achieve this by using the design principles we discussed in
the [previous article](../02-pitfalls-and-practices#principles):

{{< summary >}}

- Illegal states should be unrepresentable.
- State changes should be encapsulated.

{{< /summary >}}

We've seen various examples of how to put this into practice. Until now, the code examples we looked at were rather
simple and straightforward. In the next installment we'll conclude this series by looking at a few examples that are
less straightforward.
