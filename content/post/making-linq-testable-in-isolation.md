+++
aliases = ["/blog/making-linq-testable-in-isolation"]
author = "Scott Guymer"
date = "2018-03-12T10:53:21Z"
tags = ["c#", "linq"]
title = "Unit testing complex LINQ statements"

+++
LINQ is amongst the greatest features of c# and when it was released people instantly took to using it. It forms the basis of may ORMs and other object manipulation tools. It can form a great abstraction when loading and manipulating data.

It can however like a lot of great tools be horrendously misused by a lot of people. I have seen 10+ line LINQ statements dropped into the middle of large blocks of business logic. All without any unit tests or even many comments on what the statement actually does.

There is no way you are going to be able to make it any better without breaking it so you often just leave it! Who would want to risk touching that and breaking the complex stuff its doing.

The first thing you need to do is find some way of abstracting out the huge statement from the rest of the code sat around it and constructing some archeology style after the fact tests to test what you think it should do. You might then have a fighting change of making it more performant or even just a bit easier to understand!

## Enter the ISelect Pattern!

_Disclaimer: I didn't come up with this myself  I'm not claiming ownership of this pattern. Its something I have picked up over the years and have found very useful._

I have found this pattern very useful for abstracting away complex LINQ statements you uncover in code. Of course it also useful when you are starting afresh an have a LINQ statement that you need to write and test.

The first thing we need to do is define an interface for all of our queries. This interface uses a fair bit of generics and basically allows you to define an implementation where the in and out types are fixed. It is built on the premise that you have an `IQueryable<T>` of elements that you want to operate on which forms the input of the `Run` method. You are then free to define what sort of type you want to return. It could be a single object or another `IList<T>` or even another `IQueryable<T>`.

```csharp
    using System.Linq;
    
    /// <summary>
    /// An Interface to define a selector query that is used to return data from the database.
    /// </summary>
    /// <typeparam name="TIn">The type of the input to the query.</typeparam>
    /// <typeparam name="TOut">The type of the output from the query.</typeparam>
    public interface ISelect<in TIn, out TOut>
    {
        TOut Run(IQueryable<TIn> objects);
    }
```

So we now have an interface that we can use to define our query implementations. To do this we simply define a class that implements the interface `ISelect<in TIn, out TOut>` as simple example of this is below. Where we are expecting a collection of posts in an a filtered collection of posts returned.

```csharp
    using System.Linq;
    
    public class ExampleQuery : ISelect<in Post, out IEnumerable<Post>>
    {
        public IEnumerable<Post> Run(IQueryable<Post> objects)
        {
          return objects.Where(x => x.Published);
        }
    }
```

This seems pretty simple and you might be wondering what the point it. So now we move on to an example of how we might test this LINQ in isolation from any other code it might execute within. You can see from the test below that we create a list of test data, then instantiate the subject under test and execute the `Run()` method. We can then perform any assertions we want about the result, checking the size of the returned set or even the contents themselves. We can have as many tests as we need to test the functionality of the LINQ query.

```csharp
    using System.Linq;
    
    [TestFixture]
    public class TestASelector
    {
        [Test]
        public void TestSelector()
        {
            var data = new List<Post> { new Post() { Published = true } };
            
            var selector = new ExampleQuery();
            
            var result = selector.run(data.AsQueryable());
            
            // assert things about the collection here
        }
    }
```

Because the `ISelect` implementation doesnt retain any state we can quite easily slot it into any code that we might need to wrap it in. Below is an example of how we would run our selector within a piece of code that may have some logic either side of the selector. You can see in this example we are pumping a table from a Entity Framework DBContext into the run method. This could quite easily be any other ORM or even just a collection of POCO objects.

```csharp
    using System.Linq;
    
    public class UseASelector
    {
        private DBContext db;
        
        public void DoWork()
        {
            var selector = new ExampleQuery();
            
            var result = selector.run(db.myTable);
        }
    }
```

Ok so thats pretty strightforward you might say. What about if i need to be able to influence the result with some sort of variable. Taking the posts example above we might want to filter by all posts published after a certain date. We can extend our implementation of `ISelect` further by introducing the concept of parameters that can be used to influence the query. These parameters are passed as part of a constructor and are kept private inside the implementation to prevent any unwanted behaviour. You can see below we pass in a `DateTime` and use this as part of the where clause in our LINQ statement.

```csharp
using System.Linq;

public class ExampleQueryWithParams : ISelect<in Post, out IEnumerable<Post>>
{
	private DateTime startDate;

	public ExampleQueryWithParams(DateTime startDate)
	{
		this.startDate = startDate;
	}

	public IEnumerable<Post> Run(IQueryable<Post> objects)
	{
		return objects.Where(x => x.PublishedDate > StartDate);
	}
}
```

This isnt just useful on legacy code. It can be a very handy pattern to start out with in your code base. Each and every LINQ statement you use in your code can use this pattern. 

A great product of this is that it makes our LINQ statements re-usable across different parts of the codebase. So if there is some important business logic that resides as part of a query then it can be isolated, tested and re-used. You can even inherit from it, decorate it or any other pattern you might use with any normal abstraction.