# Embracing Types in Typescript

## Betraying Object Oriented Programming

Though Object-Oriented Programming is generally effective for designing applications in a waterfall style, it's not very useful to develop in past it's initial design. If at any time there's a change in scope for the project or something in the code has to be replaced or appended, grouping functionality and state into a single Object becomes painful to maintain and detrimental to programming velocity and fulfilling feature requests.

Typescript *technically* **does** allow you to wallow in the old styles of Object-Oriented Programming: It provides Classes, Interfaces, and Methods for you to program how you've been taught in university. However, it also hints at a much more fast and efficient future for programming large applications among a team of programmers, by presenting a new kind of paradigm - Types.

## Bring Forth the Types

Lets say you have a JSON format to send to your backend to your frontend and you know what the format is, you can declare that format as a Type:

``` typescript
export type CatSearch = {
    Name?: string,
    Gender?: "Male" | "Female",
    PredominantColor?: "Black" | "White" | "Grey" | "Orange",
    OtherFeatures?: {
        Pattern?: "Calico" | "Tabby" | "Seal-Point",
        Nickname?: string,
        Zodiac?: string,
    }
}
```

This declaration is super dense, but provides a lot of data that can be filled in during the course of the front-end's execution.

The `?` denotes an optional member. In the case of a search request, this would be appropriate since you don't need to provide all the search criteria to come up with some results.

Observe the heavy use of `|` to denote selecting between different types of member. If we can guarantee the result will either be `Male` or `Female`, then we can ensure that our code can't provide a search request for a Gender that isn't yet handled by the backend.

You will also notice some nesting of types in this within `OtherFeatures`: We can easily nest types with Typescript and make it super explicit what should be provided in these nested members. It would be easy enough to remove the nesting and flatten the structure later on if that would make your code appear more coherent.

When we're making a request from the frontend we may start with creating a variable like this:

``` typescript
let search: CatSearch = {};
```

and filling it in using data from JQuery or something:

``` typescript
search.PredominantColor = $('.PredominantColorField').value;
```

Or maybe use some lovely looking object merging to append some additional search criteria, and send it off:

``` typescript
search = {...search, OtherFeatures: { Pattern: "Tabby" }};
SendSearchRequest(search); // Send off that request!
```

Lets look at what the server might return as a result:

``` typescript
// CatSearchResult.ts
export type CatSearchResult = {
    Type: "Cat",
    Name: string,
    Age: number,
    Gender: "Male" | "Female",
} | {
    Type: "Photo",
    Url: string,
} | {
    Type: "DatabaseEntry",
    Format: "SPCA" | "Cat Cafe";
    Url: string,
}
```

Maybe your search can provide a set of different types of results, either a cat result stored directly in the server's database, a reference to a photo of a cat, or a reference to an external database that the front-end can poll later on using the provided format for polling. For this case I've concatenated 3 types together using `|` characters. 

Handling this combined type is pretty intuitive in VSCode:

``` typescript
// CatSearchHandler.ts
import { CatSearchResult } from "CatSearchResult.ts"

export function HandleCatResult(result: CatSearchResult) {
    switch (result.Type) {
        case "Cat":
            console.log(`The name of the cat found is ${result.Name}`)
            break;
        case "Photo":
            LoadPhotoFromURL(result.URL);
            break;
        ...
    }
}
```

Once we ascertain that the `Type` member `== "Photo"`, VSCode will treat the `result` in this `case` context as *exclusively* the `{ Type: "Photo", Url: string }` type. So if we attempted to get an `Age` member from the result will be marked as a syntax error.

## Refactoring Made Easy

Ok we wrote our initial solution... but we get feedback that spurs us into adding more features and making changes to existing code. Refactoring Types within Typescript is fast and concise. Let's say that later on you find the need to split up the CatSearchResult type into smaller subtypes since it's getting pretty big, well it's *super* quick to accomplish:

``` typescript
// CatSearchResult.ts
export namespace ResultTypes { // Group sub-types in a namespace
    export type Cat = {
        Type: "Cat",
        Name: string,
        Age: number,
        Gender: "Male" | "Female",
    }
    export type Photo = {
        Type: "Photo",
        Url: string,
    }
    export type DatabaseEntry = {
        Type: "DatabaseEntry",
        Format: "SPCA" | "Cat Cafe";
        Url: string,
    }
}

export type CatSearchResult = // Keep the original type together by combining the sub-types
    ResultTypes.Cat |
    ResultTypes.Photo |
    ResultTypes.DatabaseEntry;
```

Here, `namespace` is used to allow us to use simple names like Cat and Photo without worrying about naming conflicts in another area of our code.

Once the type is split up this way, we could have smaller search result handling functions that specifically handle the `CatSearchResult.DatabaseEntry` sub-type:

``` typescript 
// CatSearchHandler.ts
...

import { ResultTypes } from "CatSearchResult.ts"

export function HandleDatabaseEntryResult (result: ResultTypes.DatabaseEntry) {
    const format = result.Format;
    const url = result.Url;
    if (format == "SPCA") {
        // Poll the database for cat info using SPCA format
    } else {
        // format == "Cat Cafe" here
    }
}
```

Finally, if you need to add some more to your existing types, it's possible to quickly Frankenstein together some new sub-types by using some niche `&` type declarations, if you're short on patience and time, or don't like copy-pasting code:

```typescript
// CatSearchResult.ts
export namespace ResultTypes {
    ...
    export type WildCat = Cat & {
        Type: "WildCat",
        Species: "Bobcat" | "Puma" | "Leopard",
        NaturalRange: string,
    }
}
```

Which is the same as a more verbose declaration - manually combining the members of Cat and the newly mentioned WildCat members:

``` typescript
export type WildCat = {
    Type: "WildCat",
    Species: "Bobcat" | "Puma" | "Leopard",
    Name: string,
    Age: number,
    Gender: "Male" | "Female",
    NaturalRange: string,
}
```

## Inlining Type Declarations

There's one more typing secret that you can leverage to make writing your code easier out-of-the-gate: You need not go out of scope to declare new types and refine down sub-types.

``` typescript
function HandleActions(action: {
    Type: "Pet",
    Speed: number,
    Location: "Back" | "Head" | "Nose";
} | {
    Type: "Admire",
    Duration: number,
}) {
    switch (action.Type) {
        case "Pet":
            const speed = action.Speed;
            const location = action.Location;
            // Petting behaviour
        case "Admire":
            ...
    }
}
```

Here the type is essentially anonymous and is described directly alongside the `action` parameter. This is great for types that will be used in only one place and exist for smaller functions. You could imagine having to go back later and split out this declaration into it's own separate `Action` type declaration, which is fine! It's just nice to be able to write it down as you think about it and reform your code later to be more extendable if there's a demand.

## Philosophy

So why bother pursuing Types versus just staying with Classes, Methods and Interfaces?

For me, I find that the ability to quickly mock up code that works and refactor safely later is a huge boost to my productivity. If you've ever been successful writing essays in multiple passes where you dump your ideas down first and determine structure and organization second, you understand this kind of programming style. I find that Typescript is a unique language in how it supports this kind of habit.

I still find myself gravitating towards classes on occasion, don't get me wrong, for data structures like lists and dictionaries they are a perfect match. **However**, for business logic, the core unique defining functionality of your apps, I don't see classes being very useful. You will be changing your code, requirements seldom remain constant, so that over time the class structures and hierarchies you so neatly defined will become a prison that feels nigh impossible to break out of.

In contrast, writing code with Functions and Types has a very different vibe, where the dependency structure can be very fluid and free. Though I may still be on my Honeymoon phase, having only written maybe 8000 lines of Typescript code versus hundreds of thousands of lines of C# code.

One thing that is a clear advantage of classes is the ability for basically all IDEs including VSCode to suggest things that an object can do [Methods], as well as their properties. For those learning a new programming language or API, being able to have VSCode tell you what you can do with certain objects is great as it opens up the programmer to learning more about the tools available to them. As such, I think there's an extension that could be made here: Adding a shortcut / tooltip that searches your codebase for all Functions that take a given object as a parameter. Having that kind of automatic guidance would mean the world to people that are trying to make their way around a codebase they didn't write.

## Summary

The easiest way to figure out how Typing can serve you in Typescript is to start writing code and seeing what syntax errors and member suggestions pop up as you code. The ideas and suggestions shown here are far from all that can be done with enough experimentation in VSCode. Also, read the Typescript documentation! With any luck you will find yourself more empowered to program in ways you're not normally used to and increase the number of tools in your toolbelt.

## Resources

* Typescript Docs, Basic Types - https://www.typescriptlang.org/docs/handbook/basic-types.html
* Typescript Docs, Advanced Types - https://www.typescriptlang.org/docs/handbook/advanced-types.html

## References / Inspiration
* The Wrong Abstraction - https://www.sandimetz.com/blog/2016/1/20/the-wrong-abstraction
* All the Little Things - https://www.youtube.com/watch?v=8bZh5LMaSmE
* Object-Oriented Programming is Bad - https://www.youtube.com/watch?v=QM1iUe6IofM
