# Day #13 - Option type
What do you do when things are not going as expected? All's well as long as _some_ expected result has been produced, but if _none_ was reached, then you change the course of your logic.

You can do that using an `if-then_else` expression. Take for example a scenario where an application can be called with a parameter - and if that's missing a default is assumed.

```fsharp
[<EntryPoint>]
let main (argv:string array) = 
    let get_param = 
        if argv.Length > 0 then
            argv.[0]
        else
            "default"

    get_param |> printfn "%s"
    
    0
```

This is pragmatic - but hardly following the _Single Responsibility Principle_ (SRP). The expression in `get_param` does two things at the same time: it retrieves a value using a certain "API" (`argv`) and decides what to do if the value is missing.

If you want to separate the two responsibilities, you need to return an indicator with the parameter value, whether there was a parameter at all; and if there wasn't you generate a default value.

```fsharp
[<EntryPoint>]
let main (argv:string array) = 
    let get_param =
        if argv.Length > 0 then
            (true, argv.[0])
        else
            (false, "")
    
    let default_param (param_found, param_value) =
        if param_found then
            param_value
        else
            "default"
    
    get_param |> default_param |> printfn "%A"
    
    0
```

F#'s ability to create ad hoc tuples comes in handy here.

But then... This does not feel very comfortable. For such a common case it looks cumbersome to use a tuple.

That's why F# provides a solution out of the box; it's called the Option type.

## Option type
The Option type is a predefined _discriminated union_ type. But don't let that term bother you right now; you'll learn what union types are on another day. For now it's enough to know how simple it is to use the Option type.

It has only two values: one called `Some`, the other called `None`. It's almost like with the `bool` type or a C# `enum` type.

The `Some` value is used if there is some data, e.g.

```fsharp
let question = Some "What' the time?"
```

The `None` value on the other hand signifies no data, e.g.

```fsharp
let question = None
```

As you can the, in the `None` case there is no further data, in the `Some` case there is additional data (which can be of any type).

Please note how that's different from C# code like

```csharp
string question = "What's the time?";
```

and

```csharp
string question = null;
```

`question` in both cases has the type `string`. A user of question thus could always write

```csharp
if (question.Length > 0) {...}
```

and it would crash, if `question` was `null`.

In F# the type of `question` is not `string` but `string Option` (or `Option<string>`). There is no `null` value in F# (except for when dealing with object-orientation or interop with the .NET framework). F# strives to help you avoid suffering from a [billion dollar mistake](http://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare).

So `question`'s value either is `Some "..."` or `None`.

You can access Option type value's value through a property:

```fsharp
let text = question.Value
```

Also you can check for either "state" of an Option type value by calling `IsSome` or `IsNone` on it.

But that's not really how you should work with the Option type. It's too imperative, too OO-style. Rather use `match` to handle either case. Here's how the command line parameter problem can be solved nicely with the Option type:

```fsharp
[<EntryPoint>]
let main (argv:string array) = 
    let get_param =
        if argv.Length > 0 then
            Some argv.[0]
        else
            None

    let default_param = function
          Some p -> p
        | None -> "default"

    let process_value p =
        printfn "<%s>" p

    get_param |> default_param |> printfn "%s"
    
    0
```

`default_param` uses the `match` shortcut to decompose the parameter.

With the Option type you can clearly distinguish between the "data present" or "success" case and the "no data present" or "failure" case. Use an Option type when you feel tempted to return a tuple with one boolean element signaling a success (see above), i.e. use it to describe the outcome of an "experiment".

Remember the kata about converting numbers to/from roman numerals? It contained a decision about the type of number provided on the command line: is it roman or arabic? This was solved using a regular expression:

```fsharp
let is_roman_number n =
	Regex.Match(n, "^[IVXLCDM]*$").Success
```

If you're a regex buff, then that's of course easy. Mere mortals, though, might have sought a different solution, e.g. by trying to convert the number string to an integer using `TryParse()` of the `int` type.

```fsharp
let is_roman_number n =
    let (wasArabic, _) = System.Int32.TryParse(n)
    not wasArabic
```

This works nicely - but `TryParse()` returning a tuple is not very elegant, although it's a bit of F# magic converting a boolean return value and an `out`-parameter into a tuple.

But what can you expect from a C#-based function ;-) The .NET CLR/BCL functions don't know the Option type.

With the Option type and a `match_` the problem can be solved in a more natural F# manner.

First define a F# equivalent for the `TryParse()` function:

```fsharp
let tryparse_int t =
    match System.Int32.TryParse(t) with
      (true, i) -> Some i
    | (false, _) -> None
```

Then use this to determine the command line parameter type and act accordingly:

```fsharp
match tryparse_int n with
      Some i -> i |> convert_to_roman |> printfn "%s"
    | None -> n |> convert_to_arabic |> printfn "%s"
```

This works nicely, but is not very domain specific. What does `match tryparse_int n with` etc. mean?

If another level of indirection is the default solution for software development problems in general, then adding yet another function is the default solution in Functional Programming ;-) So why not wrap the obscure match into another function to assign meaning?

```fsharp
let choose_action_on_type n onArabic onRoman =
    match tryparse_int n with
    | Some i -> onArabic i
    | None -> onRoman n
```

Note the last two parameters. They are [continuations](https://en.wikipedia.org/wiki/Continuation-passing_style). The function not only checks the type of the parameter, but also chooses how to proccede - which actually is the reason for the type check.

Something has to be done if the command line parameter is a roman number, something else if it's an arabic number. What exactly should be done is of no concern to the function. That's why the function parameters have names `onArabic` and `onRoman`. They focus on the cases distinguished by the function, they are pertaining to the inner workings of the function, not the outside world.

When calling this function thus not only the command line parameter to check has to be passed in, but also two functions. One describing what's supposed to happen for roman numbers, another one to be called for arabic numbers.

```fsharp
choose_action_on_type n
    (fun a -> a |> convert_to_roman |> printfn "%s")
    (fun r -> r |> convert_to_arabic |> printfn "%s")
```

***

The Option type is very useful for many cases where you'd use tuples or even exceptions in C# or Java. Or even worse: `null` values. You should keep it handy because you'll need it quite often when working with F#.

But of course good old exceptions have their use, too. Tomorrow you'll learn about them.

### Read more
* Microsoft, [Options (F#)](https://msdn.microsoft.com/en-us/library/dd233245.aspx)
* Scott Wlaschin, [The Option type - And why it is not null or nullable](http://fsharpforfunandprofit.com/posts/the-option-type/)