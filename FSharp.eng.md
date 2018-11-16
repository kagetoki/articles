# F# spoiled me, or why I don't enjoy C# anymore
## I really used to like C#

This was my primary language, every time I compared it to other popular languages I was happy I accidentally chose it.
Python and Javascript lack static typing (assuming JS has any kind of typing), Java lacks proper generics, properties, events, value types, which brings up a mess of all those wrapper classes like `Integer` and so on.

I have to mention, that I was comparing just a languages themselves and a comfort of writing in them without covering the topic of tools and other environment, since those are decent enough for all of them to make enterprise development reasonable and comfortable.

And then I tried F# out of curiosity.

## Ok, so what's so good about it?

Briefly the following:
- Immutability by default
- Functional paradigm turned out to be more solid and concise than what we call OOP today
- Sum types or `Discriminated Unions`
- Computation expressions
- SRTP or Statically Resolved Type Parameters
- `null` safety
- laconic syntax
- type inference

Well, `null` part is an easy one: nothing fouls the code more than endless checks of returned values like `Task<IEnumerable<Employee>>`. So lets talk about both immutability and conciseness.
Consider the following POCO class:
```csharp
public class Employee
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public string Phone { get; set; }
    public bool HasAccessToSomething { get; set; }
    public bool HasAccessToSomethingElse { get; set; }
}
```

Brief, simple, nothing redundant. Couldn't be shorter. Now look at the F# equivalent:
```fsharp
type Employee =
{ Id: Guid
  Name: string
  Email: string
  Phone: string
  HasAccessToSomething: bool
  HasAccessToSomethingElse: bool }
```
Now there's *truly* nothing redundant. Useful information is contained in `type` keyword, type name, field names and types. In C# there are useless `public` and `{ get; set; }` in every line. Besides that in F# we've got `null` safety and immutability.
Well, actually we can get immutability in C# too, as for a `public` thing - it's not a big problem with an autocomplete:

```csharp
public class Employee
{
    public Guid Id { get; }
    public string Name { get; }
    public string Email { get; }
    public string Phone { get; }
    public bool HasAccessToSomething { get; }
    public bool HasAccessToSomethingElse { get; }

    public Employee(Guid id, string name, string email, string phone, bool hasAccessToSmth, bool hasAccessToSmthElse)
    {
        Id = id;
        Name = name;
        Email = email;
        Phone = phone;
        HasAccessToSomething = hasAccessToSmth;
        HasAccessToSomethingElse = hasAccessToSmthElse;
    }
}
```

Done! Although, the amount of code has tripled - all the fields are repeated twice. More than that, should we add a new field we can easily forget to add it to constructor parameters and/or initialize it, and *compiler won't say anything*.
In F# on the other hand when we add a new field all we have to do is to add it. That's it. And initialization looks like this:
```fsharp
let employee =
    { Id = Guid.NewGuid()
      Name = "Peter"
      Email = "peter@gmail.com"
      Phone = "8(800)555-35-35"
      HasAccessToSomething = true
      HasAccessToSomethingElse = false}
```
And if we miss a field the code won't compile.
Since this objects are immutable the only way to make a change is to create another one. But what if we need to change just one field?
Piece of cake!
```fsharp
let employee2 = { employee with Name = "Bob" }
```

How can you do it in C#? I guess you already know. Besides, that `{ get; }` thing is no good unless the inside object is immutable itself, so you gonna take care of them too. Should I even mention collections?

But do we really need this immutability so much?

I added those access fields on purpose. In real projects there's typically an access service, responsible for them, and quite often it receives a model and mutates it, setting access fields to `true` where needed. And so at some point in my program I get this model and access fields are set to `false`. But what does it mean? It could be that the model hasn't been put through that service or it could be that some fields had been forgotten to be taken care of, or maybe just an employee doesn't have an access - I don't know, I have to check it and read a lot of code.

But when the structure is immutable, I know that everything's fine since the compiler **forces me to fully initialize an object upon declaration**.
In other case upon adding a new field I have to:
- Check **all** the places where this object is being created - maybe I should fill those fields over there too
- Check all the services, mutating this object
- Write/update unit tests related to this object
- Possibly update mappings

Dealing with immutable objects you also can be certain that other code or threads won't corrupt their insides. But in C# it's so hard to get real immutability that writing immutable code doesn't worth the effort, you don't need immutability at that cost.

But enough with that, what else have we got? In F# we also got for free:
- Structural Equality
- Structural Comparison

So now we can do this:
```fsharp
if employee1 = employee2 then
//...
```
And this code would do *exactly* what we meant by that - real equality. `Equals` that compares objects by reference *is a pure garbage*, we've already got `Object.ReferenceEquals`, thanks.

One could argue that no one needs it, since in real life projects we compare objects so rarely, that it's no big deal to override `Equals` & `GetHashCode` manually. But I think that cause-effect relationship is going backwards here: we don't compare regular objects because manual overriding and *maintaining* of `Equals` and `Compare` takes enormous effort. However when it comes for free, the use can be found immediately: you can put them into `HashSet` or `SortedSet`, use them as a key in `Dictionary` and don't bother comparing objects by their ids, but just compare them (although id option is still available of course).

## Discriminated Unions

I suppose most of us have learned at our first teamlead's knees, that building a workflow on exceptions is wrong. For instance, instead of using `try { i = Convert.ToInt32("5"); } catch(Exception){}` it's better to use `int.TryParse`. But besides this primitive and decrepit example we constantly break this rule. User provided crooked up input? `ValidationException`! Out of array's bounds? `IndexOutOfRangeException`!

In the clever books they say that exceptions are for the exceptional, unpredictable situations, when something has gone wrong so badly that trying to continue our work doesn't make sense at all. Good examples are `OutOfMemoryException`, `StackOverflowException`, `AccessViolationException` and so on. But getting out of bounds in array is unpredictable? Really? I mean the indexer gets `Int32` as input, which has 2^32 possible values. In most cases arrays we work with don't exceed 10000 length. Rarely a million. Which means that there are **far more** values of `int` that cause exception than the ones that work correctly. Which means that given a random value of `int` it's far more likely to get into exceptional situation, than the normal one! Now, I know no one uses random index in arrays, everyone checks their length and so on. And no wonder! Still it's pretty representative. Imagine that we are talking about some abstract function that handles input of some type, but then you find out that **most** of those values cause an exception. Kinda annoying, isn't it?

Same thing goes for validation. User provided wrong data? **What a surprise.**

The reason for this dramatic abusing of exceptions is simple: the type system isn't powerful enough to represent a scenario like "If everything's fine then give me a result, otherwise return an error". Strict typing requires us to return the same type in every execution branch (fortunately). But adding `string ErrorMessage` & `IsSuccess` to all of our models is the last thing we need. Therefore in C# reality exceptions is probably the lesser evil. Of course you could do something like this:
```csharp
public class Result<TResult, TError>
{
    public bool IsOk { get; set; }
    public TResult Result { get; set; }
    public TError Error { get; set; }
}
```

But here again we need to write a lot of code in order to make invalid state unrepresentable. Otherwise we can initialize both result and error, forget to set `IsOk`, so it really brings more problems than it solves.

In F# you can define such things much easier:
```fsharp
type Result<'TResult, 'TError> =
    | Ok of 'TResult
    | Error of 'TError

type ValidationResult<'TInput> =
    | Valid of 'TInput
    | Invalid of string list

let validateAndExecute input =
    match validate input with // check validation result
    | Valid input -> Ok (execute input) // if valid return execution result
    | Invalid of messages -> Error messages // if not we return an error with the list of messages
```

It's that simple, concise and most importantly, the code is self-documented. You don't have to write xml doc and specify that method throws an exception on some scenarios, you don't have to wrap other method calls with `try/catch` just in case. In this kind of type system exceptions occur in truly dangerous and wrong situations.

When you throw exceptions here and there all the time you need a sophisticated error handling. Now you get yourself a `BusinessException` or an `ApiException`, then you have to inherit tons of exceptions from it and keep tracking that everyone is using those right exceptions, and if someone makes mistake - clients will get `500` instead of `404` or `403`. Now you've got a tedious logs reading and digging through stacktraces ahead of you.

F# compiler gives a warning if we didn't go through all the cases in `match` expression. Which is very convenient when you add a new case to your DU. In DU we *define scenarios*, for instance:
```fsharp
type UserCreationResult =
    | UserCreated of id:Guid
    | InvalidChars of errorMessage:string
    | AgreeToTermsRequired
    | EmailRequired
    | AlreadyExists
```

Now we see all the possible outcomes of our operation, which is far more representative than some list of exceptions. More than that, when we added a new case `AgreeToTermsRequired` according to new requirements, F# compiler gave as a warning where we handle this result.

I've never seen such a descriptive list of exceptions in a project, for obvious reasons. Those scenarios are defined in text messages for those exceptions instead. As a result we get duplicates occasionally, and the opposite, when developers get lazy and make those messages more abstract.

By the way, array indexation now also looks prettier, none of those `if/else` and length checks:
```fsharp
let doSmth myArray index =
    match Array.tryItem index myArray with
    | Some elem -> Console.WriteLine(elem)
    | None -> ()
```
Here we use the type from the standard lib:
```fsharp
type Option<'T> =
    | Some of 'T
    | None
```
Which is a better alternative to `null` or missing value case. It's better, because whenever we see it, we know, that the value can be missing **according to requirements**, not because of developers mistake. And again, compiler keeps an eye on you, making you to go through all the possible cases.

## Solid Paradigm

Pure functions and expression-based language design force us to write extremely stable code. Pure functions satisfy following criteria:
- Function has NO side effects, the only execution result is evaluated output
- Function always produces the same output for the same input

Add the totality (when function produces correct output for every possible input) on top of that, and you'll get solid, predictable thread-safe code that always works right.

Expression-based design tells us that everything is an expression and everything has an execution result. For instance:
```fsharp
let a = if someCondition then 1 else 2
```

Compiler forces us to write all the possible branches, you can't stop at `if`, you need `else`, or the code won't compile, so it's impossible to forget a scenario.

In C# you have ternary operator which does the same thing, but it's also quite easy to write unsafe code, when you define something, then you mutate some part of it, and then you missed something.

## Away from familiar OOP

Common case: you've got a service, which depends on a few other services and a repository. Those services have dependencies of their own. Now all that gets mixed together into a nasty cocktail of functionality with a mighty DI framework and handed over to a web controller.

Each dependency of that service has 2-5 dependencies in average, and each of them, including our guy, has 3-5 methods in average. Most of them aren't used in every specific scenario, of course. Out of all this giant method tree in every specific case we need 1-2 methods of every (?) dependency, but anyway we tie it all together and create a lot of objects. And mocks, of course. What would you think? We do have to test all this beauty, don't we? So now I want to test a method of my service. In order to call it, I need an object of that service. And to create that object I have to pass the mocks. The trick is to know what specific mocks I need: some of those mocks aren't being called here, so I don't need them, others are, but just a couple of their methods. So every time I've got myself a tedious setup for my test with specifying return values and all that stuff. Then I want to test another case *in the same method*. Setup again! Sometimes there's more code in tests for a method, than in the method itself. Oh, and yes, *for every other method* I have to dig inside its guts and take a look on what specific dependencies I'm gonna need this time. By the way there goes encapsulation.

And it kicks us in other ways too: whenever I need just 1 method of some service, I have to satisfy all of its dependencies, even if I don't actually need them. Of course it's being handled by DI frameworks, but still I have to go and register all of them. Often it can be a problem, if some of those dependencies declared in another assembly and now we have to reference it. In some cases it can screw our architecture up, so now we have to mess with inheritance or move some piece of code to a separate service, increasing the number of components in our system. Doable of course, but still quite unpleasant.

In functional world it's being done in a different way. The coolest guy here is pure function, not an object. And mostly we deal with immutable values, not mutable variables. Besides, functions can be easily composed, so in most cases we don't even need *objects* of services at all. A repository gets from DB something you need? Well then get it and pass that value, not the repo itself!

A simple case could look like that:
```fsharp
let getReport queryData =
    use connection = getConnection()
    queryData
    |> DataRepository.get connection // connection dependency is being injected in function, not in constructor
    // and now we don't need keep tracking on lifecycle of dependencies
    |> Report.build
```
For those who's unfamiliar with `|>` operator and currying, this is equivalent to the following code:
```fsharp
let gerReport queryData =
    use connection = getConnection()
    Report.build(DataRepository.get connection queryData)
```
In C#:
```csharp
public ReportModel GetReport(QueryData queryData)
{
    using(var connection = GetConnection())
    {
        // Report here is a static class. F# modules are translated to them
        return Report.Build(DataRepository.Get(connection, queryData));
    }
}
```

And since functions are easily composed, we can do it like that:
```fsharp
let getReport queryData =
    use connection = getConnection()
    queryData
    |> (DataRepository.get connection >> Report.build)
```

Now please note, that `Report.build` can be tested far more easily. You don't need mocks at all. More than that, there's a framework `FsCheck`, which can generate hundreds of input parameters and run your tests with them, showing you which of them have broken it. Now that's a proper kind of testing, 'cause tests like that truly try to crucify your system, when unit-tests more like gently tickle it.

All you have to do to run those tests is to define a generator for your type. Why is it better than mocks? Because generators are universal, they apply to **all** your future tests, and you don't need to know implementation of anything for creating them.

By the way, your business logic assembly doesn't reference an assembly with repositories, nor the one with their interfaces. Which means if you wanna switch, for example, from EntityFramework to Dapper, your BL assembly **won't be affected at all**.

## Statically Resolved Type Parameters

It's better to show, than tell:
```fsharp
let inline square
     (x: ^a when ^a: (static member (*): ^a -> ^a -> ^a)) = x * x
```
This function works with *every* type, which has a multiplication operator with satisfying signature. And it works not only for operators, but for usual methods too!
```fsharp
let inline GetBodyAsync x = (^a: (member GetBodyAsync: unit -> ^b) x)

open System.Threading.Tasks
type A() =
    member this.GetBodyAsync() = Task.FromResult 1

type B() =
    member this.GetBodyAsync() = async { return 2 }

A() |> GetBodyAsync |> fun x -> x.Result // 1
B() |> GetBodyAsync |> Async.RunSynchronously // 2
```
You don't need to make wrappers and interfaces for them, you just need those types to have the right method! I don't know how you can do it in C#.

## Computation Expressions

We used an example with `Result` type. Consider, we have a cascade of operations, each of them returns that `Result`. And if any of them results to an error, we wanna stop the execution at that point.

Instead of writing endless ladder like this:
```fsharp
let res arg =
    match doJob arg with
    | Error e -> Error e
    | Ok r ->
        match doJob2 r with
        | Error e -> Error e
        | Ok r -> ...
```

We can define this once:
```fsharp
type ResultBuilder() =
    member __.Bind(x, f) =
        match x with
        | Error e -> Error e
        | Ok x -> f x
    member __.Return x = Ok x
    member __.ReturnFrom x = x

let result = ResultBuilder()
```
And use it everywhere like this:
```fsharp
let res arg =
    result {
        let! r = doJob arg
        let! r2 = doJob2 r
        let! r3 = doJob3 r2
        return r3
    }
```

Now in every line with `let!` in case of `Error e` we return error. If everything is ok, we return that very thing `Ok r3`.
And you can do things like that *for anything* including custom operations with custom names. It's a great tool for making a DSL.

By the way there's a thing like that for asynchronous programming, even two of them - `task` & `async`. The first one is for familiar tasks, the second one -- for working with `Async` class. This thing is from F#, main difference from task is that is has a cold start, but it also has an integration with Tasks API. You can build complex async workflows with parallel and/or cascade execution and run them only when they're ready. Just like this:
```fsharp
let myTask =
    task {
        let! result = doSmthAsync() // like await Task
        let! result2 = doSmthElseAsync(result)
        return result2
    }

let myAsync =
    async {
        let! result = doAsync()
        let! result2 = do2Async(result)
        do! do3Async(result2)
        return result2
    }

let result2 = myAsync |> Async.RunSynchronously

let result2Task = myAsync |> Async.StartAsTask

let result2FromTask = myTask |> Async.AwaitTask
```

## File structure in projects

Since records and discriminated unions definitions are very short and you don't have much of other kinds of types in your project usually, the number of files in project is reduced dramatically. All the domain types can be defined in 1 file.

Also, in F# file order and code order matters: **by default** you can use in a given line of code only something you declared higher. It's done by design and it's extremely great, 'cause it prevents us from making cycle dependencies. And it also helps greatly during code review: file order reveals design mistakes. If some highlevel component is defined high in this hierarchy, some one screwed up dependencies. And you can tell it with a glimpse, now imagine how long would it take when you're dealing with C#.

## To sum up

Once you've got these powerful tools and you've got used to them, you begin to solve problems much faster and gracefully. Most of your code when written once and tested once works forever. Going back to C# means for me to lose my productivity. Here I was riding a motorcycle, now I'm back to riding a bicycle. I mean C# is good, but F# is awesome. And why would you need something good when you have an awesome one, right?
Yeah, C# is slowly getting some of those features too - nullable reference types, pattern matching, maybe even records. But those features come with a huge delay and they are much weaker comparing to F#. Nullable reference types are good, but `Option<'T>` is far better for several reasons, pattern matching is not as powerful, records I guess won't have `with` like syntax. And there's still a paradigm problem, which strips us of code stability and property-based tests. Those tests actually revealed to me design mistakes several times before even making a commit. It would take a really long time for a QA team to find something like that.

Unit tests on the other hand usually reveal to me that I forgot to update test configuration. And yes, sometimes they show me that I missed something in code. Something that would not even compile in F#.

I would say that biggest problem of F# is it's hard to sell it to C# devs. But if you try it it's gonna be easy, there are plenty of awesome books and there's absolutely wonderful website: [F# for fun and profit](https://fsharpforfunandprofit.com/learning-fsharp/).

There's also wonderful [russian community in telegram](https://t.me/fsharp_chat), but english speakers are also welcome.

So I strongly encourage you to give F# a try, it's gonna be fun!