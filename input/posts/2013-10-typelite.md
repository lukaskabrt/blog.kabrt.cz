Title: TypeLITE - TypeScript interfaces from .NET classes
Published: 10/3/2013
Tags: [TypeScript, TypeLITE]
---
In the last few weeks I was doing some client-side development in [TypeScript](https://www.typescriptlang.org/). It is amazing tool, that increased my productivity in client-side development dramatically.

To streamline the integration between TypeScript and C# code on the server I wrote a simple utility that generates TypeScript interfaces from our POCO classes. It help us a lot in keeping our client-side code in sync with the server side-code - when a server-side class changes, TypeScript interface is automatically updated and if the TypeScript code isn't compatible with the updated interface, TypeScript compilation fails with nice, descriptive message - a huge difference form previous experience with pure JavaScript.

I think this utility might be useful for other developers, so I made it available under MIT license. It's called TypeLITE.

^^^
FIGure
^^^ and its caption

### Example

**POCO classes:**

```csharp
public class Person {
    public string Name { get; set; }
    public int YearOfBirth { get; set; }

    public Address PrimaryAddress { get; set; }
    public List<address> Addresses { get; set; }
}

public class Employee : Person {
    public decimal Salary { get; set; }
}

public class Address {
    public string Street { get; set; }
    public string Town { get; set; }
}
```


**Generated TypeScript interfaces:**

```typescript
interface Person {
    Name: string;
    YearOfBirth: number;
    PrimaryAddress: Address;
    Addresses: Address[];
}

interface Employee extends Person {
    Salary: number;
}

interface Address {
    Street: string;
    Town: string;
}
```

Usage instructions are available on [Bitbucket](https://bitbucket.org/LukasKabrt/typelite) or on the [project page](http://type.litesolutions.net). Any comments or suggestions are welcome.