+++
author = "Scott Guymer"
categories = ["c#", "linq"]
date = "2018-03-12T10:53:21+00:00"
description = ""
draft = true
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "Making LINQ testable in isolation"
type = "post"

+++

Interface for ISelect an abstraction over complex LINQ queries

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

An implementation of  ISelect that assumes a list of Posts that have a bool Published

    using System.Linq;
    
    public class ExampleQuery : ISelect<in Post, out IEnumerable<Post>>
    {
        public IEnumerable<Post> Run(IQueryable<Post> objects)
        {
          return objects.Where(x => x.Published);
        }
    }

An example of how we might test this LINQ in isolation from any other code it might execute within

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

An example of how we would run our selector within a piece of code that may have some logic either side of the selector

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

We can extend this further by introducing the concept of parameters that can be used to influence the query

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