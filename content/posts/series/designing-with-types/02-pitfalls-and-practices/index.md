---
title: "Designing with types #02: Pitfalls and Practices"
date: 2025-06-30T09:02:40Z
draft: false
tags: [ design, "type safety", java, kotlin, architecture, OO ]
series: "Designing with Types"
cover:
  image: "cover.png"
---

{{< summary >}}
In this installment of my Designing with Types series, we'll look at how some typical backend code is set up. We'll
identify some common pitfalls and identify best practices to avoid these pitfalls.
{{< /summary >}}

<!--more-->

## Architecture

Regardless of what architecture style we use, an application almost always consists of three basic tiers or layers:
_Presentation_, _Business_ and _Infrastructure_. Here's a diagram that shows two architecture styles using these three
layers.

{{< figure src="architecture.svg" class="dg-figure-border dg-figure-padding" width="700px" >}}

In the [Hexagonal Architecture](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software)), the _Business_ layer
defines ports. The _Presentation_ and _Infrastructure_ layer contain adapters that interact with the web framework and
the persistence framework.
The [Dependency Inversion Principle](https://en.wikipedia.org/wiki/Dependency_inversion_principle) (DIP) is used to make
the _Business_ layer easy to test, because infrastructure-dependent code can easily be replaced
by [test doubles](https://en.wikipedia.org/wiki/Test_double).

In the classic [N-Tier Architecture](https://en.wikipedia.org/wiki/Multitier_architecture), the responsibilities of the
layers are the same, only the dependency rules are different from the Hexagonal Architecture.

Let's look at some examples of code we often encounter in these three layers.

### Infrastructure

As mentioned in the diagram above, this is the layer where database communication usually happens. It's also commonly
known as the _Data Access_ or _Persistence_ layer. I'm calling it _Infrastructure_ because it can be used for more than
just persistence.

When used for data access, this layer usually consists of database entities and repositories. Here's an example in
Kotlin, using JPA annotations for object-relational mapping (ORM) and Spring Data's `CrudRepository` to generate a
repository implementation:

```kotlin
@Entity
data class TodoList(
    @Id val id: UUID,
    var name: String
) {
    @OneToMany(mappedBy = "todoList", cascade = [CascadeType.ALL])
    val items: MutableList<TodoItem> = mutableListOf()
}

@Entity
data class TodoItem(
    @Id val id: UUID,
    @ManyToOne
    @JoinColumn(name = "todolist_id", nullable = false)
    val todoList: TodoList,
    var title: String,
    var dueDate: LocalDateTime? = null,
    var done: Boolean = false,
)

@Repository
interface TodoListRepository : CrudRepository<TodoList, UUID>

@Repository
interface TodoItemRepository : CrudRepository<TodoItem, UUID> {
    fun findByTodoListId(id: UUID): List<TodoItem>
}
```

### Business

This layer contains the business logic of the application. Its responsibility is to make sure that business rules are
enforced in the application. It usually consists of services that perform a certain task, using repositories and
entities from the _Infrastructure_ layer.

Here's a simple service that can create and remove todo lists (again, in Kotlin):

```kotlin
class TodoListService(val repository: TodoListRepository) {
    fun createTodoList(name: String): TodoList {
        require(name.isNotBlank()) { "Name cannot be blank" }

        val todoList = TodoList(UUID.randomUUID(), name)
        repository.save(todoList)
        return todoList
    }

    fun removeTodoList(todoListToRemove: TodoList) {
        if (todoListToRemove.items.any { !it.done }) {
            throw TodoListNotRemovableException()
        }
        repository.delete(todoListToRemove)
    }
}
```

This is a common pattern in these services. They check some business rules and if everything is okay, an action is
performed. In this example, the business rules are that a todo list must have a name and that a todo list may only be
deleted if all its todo items are done.

Here's an example that creates a new todo item:

```kotlin
class TodoItemService {
    fun create(id: UUID, description: String, dueDate: LocalDateTime?): TodoItem {
        if (dueDate != null && dueDate.isBefore(LocalDateTime.now()))
            throw InvalidDueDateException()

        val todoList = todoListRepository.findById(id).orElseThrow { TodoListNotFoundException() }
        val todoItem = TodoItem(id, todoList, description, dueDate)

        todoList.items.add(todoItem)
        todoListRepository.save(todoList)

        return todoItem
    }
}
```

In this example, besides input validation, we also ensure that we don't try to add a todo item to a list that does not
exist.

### Presentation

In the variants mentioned earlier, the _Presentation_ layer contains presentation and application logic. Alternatively,
presentation and application logic can be assigned to separate layers. It contains functionality that is used by end
users and uses the _Business_ layer to validate and execute actions that users request. In a backend application that
exposes a REST API (for example for a frontend or external systems), this layer contains REST controllers.

Here's an example of a REST controller, using annotations from Spring Web:

```kotlin
@RestController
@RequestMapping("/api/v1/todo/list")
class TodoListResource(private val todoListService: TodoListService) {
    @PostMapping
    fun create(@RequestParam name: String): ResponseEntity<TodoListRestModel> {
        val todoList = todoListService.createTodoList(name)
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(TodoListRestModel.from(todoList))
    }
}
```

The _Business_ layer is used to create and store the new todo item. The controller makes sure that a HTTP response is
returned, containing the newly created todo item.

## Code review

These code examples are pretty simple and straightforward. That's because the problem domain and the associated business
rules are simple. You could argue that this code is perfectly fine for such a simple application, and I would agree.

Still, I'd like to review this code with one important question in mind: **Does it scale?** Features will be added and
existing features will be expanded. Systems grow bigger and more complicated, and so does the code. Ideally, we want our
code to be designed in such a way that it stays maintainable while the system grows.

### Duplication

While not directly obvious in the code above, we can see that we're setting ourselves up for code duplication. For
example, let's expand our `TodoListService` with the ability to rename an existing todo list:

```kotlin
fun createTodoList(name: String): TodoList {
    require(name.isNotBlank()) { "Name cannot be blank" }
    // ...
    return todoList
}

fun renameTodoList(id: UUID, name: String) {
    require(name.isNotBlank()) { "Name cannot be blank" }
    // ...
}
```

Now we have two functions that accept a name for a todo list. Obviously, a `String` can be blank, which we don't want,
so every time we use a `String` for accepting the name of a todo list, we need to check if that `String` contains a
valid name.

This problem arises when parameter types accept more values than the domain we're implementing allows for. If that's the
case, we need input validation to make sure our function is not used in an invalid way. This is an anti-pattern,
called [Primitive Obsession](https://refactoring.guru/smells/primitive-obsession). Be aware that this isn't limited to
the usage of primitives. In general, when we use overly permissive parameter types, we likely need input validation.

### Public mutable state

Some of the state of entities in the _Infrastructure_ layer can be changed by everyone. Take `TodoItem` for example:

```kotlin {hl_lines=["7-9"]}
@Entity
data class TodoItem(
    @Id val id: UUID,
    @ManyToOne
    @JoinColumn(name = "todolist_id", nullable = false)
    val todoList: TodoList,
    var title: String,
    var dueDate: LocalDateTime? = null,
    var done: Boolean = false,
)
```

The fields `title`, `dueDate` and `done` have a public setter. This means it's very easy to write code that violates
business rules that are normally enforced by the _Business_ layer. For example, we can easily introduce code that
changes a todo item to have an empty `title` or creates a new todo item with a `dueDate` in the past.

This is an anti-pattern known as [Inappropriate Intimacy](https://refactoring.guru/smells/inappropriate-intimacy) or
[Object Orgy](https://en.wikipedia.org/wiki/Object_orgy).

### Lack of encapsulation

The issue of public mutable state is a consequence of another design issue: Data and business logic are separated
between different classes. Data is modeled as entities, services make sure that the data conforms to the business rules.
The entities shown here are basically Data Transfer Objects (DTO's). They tell the Object-Relational Mapper (ORM) what
tables exist, which columns these tables consist of, et cetera.

These DTO's are used thoughout the entire code base. While the _Business_ layer should be enforcing business rules, it's
very easy to violate business rules by changing DTO's in a REST controller, for example.

We're using an object-oriented (OO) language, but by designing our code like this we don't benefit from one of the
main strengths of OO, which is that data and the associated behaviour are **encapsulated** in one class. This makes our
code more error-prone than it needs to be.

### There's a bug!

Maybe you already spotted it, there's a bug in `TodoItemService`. Can you find it?

```kotlin
fun create(id: UUID, description: String, dueDate: LocalDateTime?): TodoItem {
    if (dueDate != null && dueDate.isBefore(LocalDateTime.now()))
        throw InvalidDueDateException()

    val todoList = todoListRepository.findById(id).orElseThrow { TodoListNotFoundException() }
    val todoItem = TodoItem(id, todoList, description, dueDate)

    todoList.items.add(todoItem)
    todoListRepository.save(todoList)

    return todoItem
}
```

I've shown this code in several talks I've given, asking the participants to spot the bug. From my limited "testing" it
seems that the bug is surprisingly hard to find just by looking at the code.

Now, where's the bug? It's in this line:

```kotlin
val todoItem = TodoItem(id, todoList, description, dueDate)
```

The `id` that is passed to the constructor is the `id` of the containing `TodoList`! Let's assume the database table for
todo items has a proper primary key or a unique constraint. If that's the case, this code will fail as soon as we try to
add a second `TodoItem` to a `TodoList`, because a duplicate key will be inserted.

This is another example of Primitive Obsession. While `UUID` isn't a primitive type in the strict sense of the word, it
_is_ a type that doesn't tell us anything about what kind of ID it actually represents. When we see just a `UUID`, it
can be hard to determine if it's the ID of a todo list, a todo item, or something entirely unrelated to the domain.

_Fun fact_: This is a bug that I accidentally introduced when I was preparing this code for a talk. I decided to include
it in the talk as an example of how Primitive Obsession can easily lead to bugs that can be hard to find.

## Smells

To summarize, in these code examples we have identified a couple of code smells:

- **Primitive Obsession** -- By using primitives or other overly permissive types, we need to duplicate input
  validation.
  Also, it's very easy to introduce bugs when multiple domain concepts are implemented using the same primitive type.
- **Inappropriate Intimacy** -- Unrestricted write access to the state of an object can easily lead to violation of
  business rules.
- **Lack of encapsulation** -- Data and behaviour are separated into different classes.

## Principles

One way we can address the issues we found is by **designing better types**. In this chapter I'd like to provide some
high-level principles and practices to help in designing better types. In future installments, we'll explore these more
in-depth.

I'd like to mention two principles that I think are fundamental to designing with types. The principles are:

- Illegal states should be unrepresentable
- State changes should be encapsulated

### Illegal states should be unrepresentable [^1]

When writing code, the very first feedback we get about our code is from the compiler. This is the shortest possible
feedback loop we can have.

For example, when we try to assign a string to an integer variable, the compiler will immediately complain.

```kotlin
val i: Int = "Hello World!" // <-- Compiler error
```

So the idea behind this principle is that in order to make our feedback loop as short as possible, we **prefer
compile-time validation over runtime validation**. We do this by designing types in such a way that it is **impossible**
to write **compiling code** that introduces an illegal state.

### State changes should be encapsulated

Whenever we're using an object, we should be confident that the state of that object is valid. It's hard to be confident
about this when the state of an object is freely mutable by everyone. This is why we use **encapsulation** in OO.

So this principle means that we should make each class **exclusively** responsible for enforcing **its own** invariants.
This implies that each class should have **exclusive control** over its own state changes.

## Principles in practice

So how do we put these principles into practice? Let's look at two simple examples.

### Value Object pattern

This is such a simple design pattern, but it's oh so powerful! A value object represents one single value. This can be a
complex value consisting of multiple fields (i.e. amount and currency for money).

A value object must conform to the following rules:

- It is immutable
- It is self-validating
- The identity of the object is the value itself

Let's look at two examples:

```kotlin
data class Name(private val value: String) {
    init {
        require(value.isNotBlank()) { "Name cannot be blank" }
        require(value.lines().size == 1) { "Name must have exactly one line" }
    }
}

data class Description(private val value: String) {
    init {
        require(value.isNotBlank()) { "Description cannot be blank" }
    }
}
```

Here we have two value objects, `Name` and `Description`. Both classes encapsulate an immutable value. Both classes make
sure that no invalid value is accepted. This is arguably the simplest possible example of encapsulation, but there are
profound consequences:

- Functions that accept a `Name` and/or a `Description` as a parameter don't need input validation for these parameters,
  because that's what the constructors have already done. We can safely assume that the arguments are valid.
- We write fewer tests, because if we don't need input validation, we don't need to unit tests input validation.
- We can't confuse parameters. The following code does not compile, because we cannot pass a `Description` when a `Name`
  is expected.
  ```kotlin {hl_lines=["4"]}
  class TodoList(val id: TodoListID, name: Name)
  
  fun main() {
      val list = TodoList(TodoListID.create(), Description("Description\nwith multiple lines")) // <-- Compiler error
      // ...
  }
  ```

So here we see both principles in practice. We make illegal states unrepresentable because we cannot confuse values with
different domains. The values in the value objects are encapsulated and guaranteed to be valid, which simplifies our
code by eliminating the need for duplicated input validation.

### Encapsulation using sum types

Here is a slightly more advanced example using a sum type:

```kotlin
sealed interface TodoItem {
    val id: TodoItemID
    val description: Description

    data class Todo(override val id: TodoItemID, override val description: Description) : TodoItem {
        fun updateDescription(newDescription: Description) = this.copy(description = newDescription)
        fun markAsDone() = Done(id, description)
    }

    data class Done(override val id: TodoItemID, override val description: Description) : TodoItem
}
```

In this example, we model `TodoItem` as a value object. The states (`todo` and `done`) of a todo item are modeled as
implementations of the `TodoItem` interface. The signatures of these types reveal that the description of a todo item in
the `todo` state can be changed. Todo items that are `done` cannot be changed at all[^2].

This is another way we can make illegal states unrepresentable using types. In fact, we take it a step further because
thanks to the sum type, we also make illegal state **changes** unrepresentable. The following code does not compile:

```kotlin {hl_lines=["4"]}
val item: TodoItem = TodoItem.Done(TodoItemID.create(), Description("Do the laundry"))
val changed: TodoItem =
    when (item) {
        is TodoItem.Done -> item.updateDescription(Description("Do the dishes")) // <-- Compiler error
        is TodoItem.Todo -> item
    }
```

The sum type forces us to check the state of the todo item, before attempting to do anything with it. If we don't, the
code simply won't compile. Code that tries to change the description of a `done` todo item also doesn't compile.

## Conclusion

A lot of the code I've seen and written during my career suffers to some degree from the problems mentioned in this
article. As said, in simple, small systems there's little harm in having some Primitive Obsession or lack of
encapsulation. In such cases adding a lot of types can feel like overengineering. This is fine, as long as you are aware
that the antipatterns can become problematic when the system grows.

By designing better types, we can make our code **safer** to use and easier to understand by explicitly **revealing
intent**. In this article we've seen two simple examples of this.

Next up, we'll talk some more about **safety** and **revealing intent** and we'll look at more examples of using types
to our advantage.

[^1]: This phrase was coined
by [Yaron Minsky](https://blog.janestreet.com/effective-ml-revisited/). [Scott Wlaschin](https://fsharpforfunandprofit.com/posts/designing-with-types-making-illegal-states-unrepresentable/)
wrote a very nice article about this, as part of his article series on designing with types using F#.  
[^2]: For the Kotlin-savvy among you: Yes, I know about `copy` 😄. We'll address that in a future installment of this
series.
