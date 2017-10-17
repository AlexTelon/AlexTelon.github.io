---
layout: post
title: C# default in Generics
---

I was working on some code during work today and had a little issue. I
had created a internal library for parsing and handling arguments with
flags to my C# project.

The idea was to be able to parse arguments like this:

`--Person --name Alex --location -x 0 -y 13.4 --height 184`

I see the arguments as a tree with nodes and leaves. In the example
above there would be a person and location node, and a name, x, y and
height leaf. Nodes contain any number of children nodes and there are
special types of nodes which have no children called leafs. As you can
see in the example above leafs can have values like `int`, `double`
and `bool`.

Here is a very much simplified example of what I was working on and where my issue came to be.

{% highlight csharp %}
public class Node
{
    public string Name;

    public Node(string name)
    {
        Name = name;
    }
}

public class Leaf<Type> : Node
{
    Type Value;

    public Leaf(string name, bool isOptional = false, Type? defaultValue = null) : base(name)
    {
        if (isOptional && defaultValue != null)
        {
            Value = defaultValue.Value;
        }
    }
}
{% endhighlight %}

With this code I my plan was to be able to create a Leaf like this:

With a code like this my goal was to be able to handle the arguments like this:

{% highlight csharp %}
class Program
{
	class Location : Node
	{
		Flag<double> X = new Flag<double("x");
		Flag<double> Y = new Flag<double("y");

		// Methods relating to Nodes
	}

	class Person : Node
	{
		Flag<string> Name = new Flag<string>("name");
		Location location = new Location("location");
		Flag<double> Height = new Flag<double("height");

		// Methods relating to Nodes
	}

	public static void Main(string[] args)
	{
		public Node Person = new Node("person");
		Person.ParseArgs(args);

		Console.WriteLine(Person.Height); // prints 184
	}
}
{% endhighlight %}
`

I created two Nodes, one for location and one for person and then
there is methods that handle the parsing and converting from string to
the respective types and what not. Now the issue is, this does no in
any way compile right now.

The problem is the following piece of code shown in the first listing:

{% highlight csharp %}
    public Leaf(string name, bool isOptional = false, Type? defaultValue = null) : base(name)
{% endhighlight %}


The issue here is Type? (shorthand for Nullable<T>). This is because Nullable<T> only works for value types. This can be seen in how Nullable<T> was implemented, some of that code is shown below:

{% highlight csharp %}
namespace System
{
    public struct Nullable<T> where T : struct
    {
        public Nullable(T value);
	...
    }
}
{% endhighlight %}


`where T : struct` is an instance of what is called `generic
constraints` and states that `T` must be of type `struct` (which all
value types are).

**Task for the reader:** I strongly recomend pressing F12 on and `int`
and any other type you in your mind belive "work in the same way" in
visual studio to see that if all of those are indeed structs!


So back to the issue! To be able to use Nullable<T> in my Leaf<T>
class I have to have the same requirement, naimly that all type T must
be structs. This is well and good for `bool`, `int` and `double` which
are types that I need to be able to have in arguments, but it does not
work for strings! Pressing F12 in Visual Studio on a string gets you
to its definition and you see that it is actually a class!

So now you are where I was when I realised I had a problem. I however
did not really understand my problem at this point. I had a bunch of
code. I had at some point decided as a sidethought that I wanted to
have optional arguments and that hence default values would make a lot
of sense. I decided `Type? defaultValue = null` was the way to go and
when the compiler warned me about not being able to use Nullable just
on any type willy-nilly I added a `where T : struct` and continued on.

I was fine with this for a while until I realised I really did need
strings in my Leafs in certain situations. So I tried to solve it.

## How did I try to solve it at first?

I imagined two classes:
{% highlight csharp %}
class Leaf<T> where T : struct
class Leaf<T> where T : string
{% endhighlight %}

The idea was that I coulde use `Type? defaultValue = null` in one and
`type defaultValue = null` in the other. But you cannot have two classes like that. I guess it is because the condition does not count as a part of the class definition the same way a name of <Type> does. Also even if it were possible having two classes with only this minor change is a lot of code duplication.

Next I thought about just:
{% highlight csharp %}
class Leaf<T> where T : struct
class Leaf
{% endhighlight %}

But that felt bad since it would be quite confusing for users of the library, see example below:

{% highlight csharp %}
Leaf<bool> FileName = new Leaf<bool>("enabled");
Leaf FileName = new Leaf("fileName");
{% endhighlight %}

Other leafs have their type clearly marked, but the string version is
not which makes the code more difficult to understand. Also this would
again require having two almost identical sets of classes which is bad
for maintainability.

After some more thinking, some googling around and some thought about
"why can't I have classes like Name<T> and Name<T> where T : struct in
the same namespace". I figured this is the type of question one might
ask on Stackoverflow because you would kinda feel smart while asking
the question. So I started doing just that, feeling smart, and
prepping for writing my question.

I started with a minimal version of my code. I started a new CLI
project in Visual Studio and stripped out all the non-essential parts
(which amounted to even less than what you have seen here). I did this
untill I had my two Leaf classes and looking at those few lines of
code in my CLI application I saw what has been clear in this post all
along, that the reason I used `where T : struct` at all was that I
wanted default values. I realised that my problem was really not how
to have classes with the same name but different generic constraints
but how to get a default value for a generic type. That realisation
lead me to find the solution in 1 minute!

### In the darkest hour.. a hero emerges!

And here default comes to the rescue! Mind you been here for the rescue since C# 2, I just did now know about it!

`default(T)` (or just `default` in certain cituaions since C#7.1) returns the default value for the type. 0 for *value types* and `null` for *reference types*.

So for all this the solution was trivial, I just needed this:

{% highlight csharp %}
public class Leaf<Type> : Node
{
    Type Value;

    public Leaf(string name, bool isOptional = false, Type defaultValue = default) : base(name)
    {
        if (isOptional && defaultValue != null)
        {
            Value = defaultValue.Value;
        }
    }
}
{% endhighlight %}

I did end up using `default(T)` instead since i does not require C#
7.1. Student-Alex would have gone for `default` but now Im a
PROFESSIONAL and that means being a bit more careful with stuff like
that and since I have said "But it works on my computer" before when I
upgraded to C# 7.0 a while back which caused minor issues for Visual
Studio 2015 users. So upgrading to 7.1 to remove 3 chars feelt like a
pretty bad ROI.

# Conclusion

A real life solution has many parts to keep track of. The process of
boiling down your question really helps you to understand what the
problem really is. Once you have understood what the problem is the
solution is just a google search away!

Also I guess knowing the ins and outs of your programing language has
its benefits!