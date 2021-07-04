---
layout: post
title: "Object-oriented Design"
author: Kevin Cang
tags: ['craftsmanship']
excerpt_separator: <!--preview-->
---
I was introduced to Object-oriented design as a paradigm that was about modeling the real world in a computer program in my intro to programming class.
It seemed to make sense to me at the time - thinking about your programs in terms of objects and what kind of actions and values are associated with those objects.

However, I stumbled upon [this talk from Robert C. Martin](https://youtu.be/zHiWqnTWsn4?t=760) and learned that there's more to the story of it than I was originally introduced to. The content of this post will essentialy be a re-telling of his talk in a more condensed manner.
<!--preview-->
# Why do we write bad code?
It wouldn't be too far a stretch to say that anyone that has worked in a fairly large or old enough code repository has probably seen some bad code lying around.
Bad code tends to exhibit common cursory symptoms: bad naming, bloated logic and lack of clarity just to name a few. As for the reason, almost always the reason will be something along the lines of "we needed to move quickly" a.k.a rushing through the problem. Any experienced developer probably knows that nothing good comes from rushing through a problem.

The aforementioned symptoms of bad code are merely entry-level red flags as they can be easily remedied (or we hope so) by some minor refactoring and having corresponding unit tests that can verify their intended behavior. However, there are some red flags that are just as common but are insidious; these symptoms are the ones cause the real troubles in building software.

# Symptoms of bad code
## Rigidity
Rigidity is the tendency for code to change in a cascading manner.

 A simple scenario would be like this: you decide to make a simple change in `class A`. But then you realize that there's other places within the code base that uses `class A` in a way such that it will break because of the simple change. 
 So now you have to make changes in those classes, which themselves could potentially break other classes that uses them. Hence, you see a cascading effect that stems from just one change.

## Fragility
Fragility is the tendency for code to break due to a change in a module that might not be conceptually related to what was broken.

A scenario would be you make a change to how the pay of employees are calculated based on their wage type and that ends up breaking a functionality that prints out a report for auditing purposes. Conceptually, these two functionalities shouldn't be related to each other but a change seems to have impacted one of the functionalities in some bizarre way.

## Non-reusability
Non-reusability is the phenomenon of not being able to re-use desirable code because it is closely integrated with code that is not desirable.

A good example of this would be business-logic code that contains logic that interfaces with a very specific database technology i.e. MariaDB. You would like to be able to extract the business-logic code so you can instead leverage a different database technology instead but you find that you are not easily able to do so because some of the business-logic itself might potentialy depend on specific features of MariaDB that might not be present in the other database technology.

# The crux of building software
In all of the symptoms covered above, the common problem among them is __coupling__. Coupling can also be thought of as bad dependencies.
- Rigidity happens because code __depends__ on other code in undesirable ways
- Fragility happens because code __depends__ on data structures in undesirable ways
- Non-reusability happens because deseriable code __depends__ on undesirable code

Dependency management is always a problem when building software, especially in an agile model where change is constant.

To illustrate this, let's consider a function call tree as follows:

![function-call-tree-illustration](/assets/images/2021/Jul/function-call-tree.jpg){:width="400px"}
{: style="text-align: center;"}

Every application has one of these and the general structure is the same where there's a `main()` function or similar to kick things off. From `main()` it calls the high-level modules which then calls the middle-level modules which then calls lower-level modules and so on and so forth. It can be easily observed that __the flow of control goes does downwards, from the higher-level to the lower-level__.

We can derive another observation by asking the question: which of these modules knows about the other? If we consider the most simple example as follows:

![modules-example-illustration](/assets/images/2021/Jul/modules-example.jpg){:width="400px"}
{: style="text-align: center;"}

We have both modules `A` and `B` and a function `f` in `B`. It's observed that because module `A` calls function `f`, the flow of control goes from `A` to `B` and we can say that `A` knows about `B`. But how do we know that for sure? The answer to that is there's a source code dependency that runs from `A` to `B` because in order for `A` to call `f`, it has to know about the existence of `f` in `B`. In practice, this is why we import other modules - so we can know about the existence of other functions external to the one we are currently on. Therefore, it can be stated that `A` knows about `B`.

![modules-example-flows-illustration](/assets/images/2021/Jul/modules-example-flows.jpg){:width="400px"}
{: style="text-align: center;"}

The important observation to takeaway from this is that the __compile-time dependency runs in the same direction as the flow of control__. This is a univeral trait: in order for one module to call another, it has to know about the other.

Keeping mind these two observations, our function call tree can now be illustrated as:

![function-call-tree-enhanced-illustration](/assets/images/2021/Jul/function-call-tree-enhanced.jpg){:width="400px"}
{: style="text-align: center;"}

All of this was done to illustrate that higher-level modules know about lower-level ones, and that is actually a **bad** thing. High-level modules knowing about lower-level modules actually violates the principle of inversion of control. In more simpler terms, would you want your high-level business logic being polluted with low-level details? That's one of the biggest contributors of making code difficult to read. Adding onto the woes, what if you had to make a change to a low-level detail? It's quite possible that a change in detail could potentially impact your high-level policies. That again is a bad thing because high-level policies should not depend on details.

Relating this back to the three symptoms:
- Rigidity happens when there's coupling down towards detail
- Fragility happens when a change happens at a low-level module can impact something in a higher-level module
- Non-reusability happens when a high-level module is tightly coupled to unwanted low-level modules or details.

# What is OO (really)?
Nowadays, nearly every programming language you can think is an object-oriented language. There was a time in the past where these languages weren't the norm and today it seems to be the standard. To understand why, it helps to borrow a bit from our CS101 curriculum when they talked about the three features of an OO language and consider them in the context of a non-OO langauge, specifically the C programming language. It's mostly a stated fact that C is not considered to be an object-oriented language but rather a procedural language. Let's observe whether or not C demonstrates the 3 features of an OO language.

## Encapsulation - did C have it?
In C, you can forward-declare your functions and variables in a header (`.h`) file, and then implement them in a `.c` file. Anyone that was interested in using your code would simply `#include` the `.h` file.

**foo.h**
```c
#ifndef FOO_H
#define FOO_H

int foo(int x);

#endif // FOO_H
```

**foo.c**
```c
#include "foo.h"

int foo(int x)
{
    return x + 2;
}
```

**main.c**
```c
#include <stdio.h>
#include "foo.h"

int main()
{
    int y = foo(3);
    return 0;
}
```

This paradigm is pretty much textbook encapsulation as none of the implementation details or the values of your data structures are exposed and your users only see the bare minimum in order to use your code.

## Inheritance - did C have it?
A common approach that was done to implement inheritance was _type pruning_, which is the practice of declaring data structures in a way such that the differing members were at the end and then we would cast pointer going from the child to parent type.

```c
struct base_class {
    int x;
};

struct derived_class {
    struct base_class base;
    int y;
};

struct derived_class2 {
    struct base_class base;
    int z;
};

int main() {
    struct derived_class d;
    struct dervied_class2 d2;

    struct base_class *b1, *b2;

    b1 = (struct base_class *)&d;
    b2 = (struct base_class *)&d2;
    b1->x = 10;
    b2->x = 100;
}
```

Although inheritance wasn't really built into C, it could be implemented in a straight-forward manner which simulated the power of having polymorphic objects in your code. It wasn't convenient but we can't really say that inheritance was not achievable in C.

## Polymorphism - did C have it?
Citing an example from Robert C. Martin's talk, we'll take a look at an over-simplified version of the UNIX copy program.
```c
// UNIX copy program simplified
// Copies from STDIN to STDOUT, one character at a time
void copy() {
  while((c = getchar()) != EOF) {
    putchar(c);
  }
}
```
Two things to keep in mind:
- `getchar()` reads a character from `STDIN`
- `putchar(c)` writes a character to `STDOUT`

All this program does is it reads from `STDIN` and writes to `STDOUT`. In a UNIX-based operating system, `STDIN` defaults to the keyboard and `STDOUT` defaults to the console. However, an important detail is that even though `STDIN` and `STDOUT` has their defaults respectively, __it can really be anything__. Instead of reading input from the keyboard, we could opt to read in input from a file. Instead of writing output to the console, we could write the output to a file. In other words, `getchar()` and `putchar(c)` are polymorphic methods.

But how is this polymorphism implemented? It turns out this polymorphism actually has a dependency on the operating system in the following way.

Every I/O driver written in C needed to have these 5 following functions:
- `read()`
- `write()`
- `open()`
- `close()`
- `seek()`

The operating system would store pointers to these functions and put them in a table (more specifically the virtual table). Hence, one could manipulate this table to point to a different I/O driver's function in order to read from or write to a different source. Therefore, it was possible to change the behavior of the above `copy()` program without changing the program itself.

To illustrate (roughly) what this looks like:

![diagram-of-polymorphism-in-c](/assets/images/2021/Jul/polymorphism-in-c.jpg){:width="400px"}
{: style="text-align: center;"}

The danger of implementing polymorphism in this manner is that it relied on everyone to be a "good citizen". We had to rely on creating and loading a table of function pointers and made sure that functions were called through that table. Anyone that didn't do that would have caused a huge problem. Most of all, there was no way of actually enforcing this behavior within the language itself. It is for this reason that polymorphism wasn't done much in C and ultimately the reason why C is not considered an OO language.

## How does polymorphism help us?
With the introduction of OO languages, polymorphism was built into directly into them and it suddenly because cheap, easy, and safe to use. Polymorphism is very interesting because it gave us the ability to do something about the problem of high-level modules knowing about low-level modules.

Let's take a look at what polymorphism lets us do about that:

![modules-example-alternate-design-illustration](/assets/images/2021/Jul/modules-example-alternate-design.jpg){:width="400px"}
{: style="text-align: center;"}

Rather than having module `A` call the lower-level module `B` directly, we can instead declare an interface `I` that declares the function `f`. Module `A` calls the function `f` and module `B` will derive from the interface and implement `f`. The flow of control still flows from `A` to `B` but now the compile-time dependency points opposite of the flow of control. This is what polymorphism enables. 

If given this capability then our function call tree from before can now look like this:

![function-call-tree-oo](/assets/images/2021/Jul/function-call-tree-oo.jpg){:width="400px"}
{: style="text-align: center;"}

We now have the power to be able to invert any compile-time dependency within the architecture of our software. This means that we can avoid the three symptoms of bad code.

In summary, __object-oriented design is about managing dependencies; by selectively inverting key dependencies within your architecture, you can avoid rigidity, fragility, and non-reusability__. In a later post, I will write about in more detail a set of principles referred to as the SOLID principles that revolves around the specific idea object-oriented design.
