---
layout: post
title:  "LINQ In-Depth Part 1"
date:   2019-12-13 15:27:16 +0100
categories: [csharp, programming, dotnet]
comments: true
excerpt: "In part 1 of LINQ In-Depth we'll be taking a look at some of the common query operators and exploring topics such as Deferred Execution and Progressive Query Building, as well as writing queries using Fluent and Query Syntax..."
# classes: wide
---

<!-- | Level                              | Topics Covered                                                  |
|------------------------------------| ----------------------------------------------------------------|
|Beginner / Intermediate / Advanced  | `GroupBy`, `Select`, `Where`, `Count`, `Sum`, `Any`, `Distinct` | -->

When C# 3.0 was released in 2007 it marked a big change for the language. It was inspired by functional programming and was driven largely by the introduction of Language Integrated Queries (LINQ), a set of language and framework features for writing structured type-safe queries over local object collections and remote data sources. In fact LINQ was such a big a deal that most of the other new language features that shipped with C# 3 such as *extension methods*, *lambda expressions*, *anonymous methods*, and *expression trees* were added to support it.

This is the first in a series of articles where we'll be taking an in depth look at LINQ. In this post we'll be taking a look at some of the common query operators such as `Select`, `GroupBy` and `Where` and exploring topics such as *Deferred Execution* and *Progressive Query Building*, as well as writing queries using Fluent and Query Syntax. 

I think one of the best ways to learn something is by actually doing it so I encourage you to open up Visual Studio (or your IDE of choice) and run the code as we go. 
You can find the full source code for this article in the accompanying GitHub repo [here](https://github.com/paulburkinshaw/linq-in-depth-part-1).

---

## Problem domain

For the purpose of this article imagine we are working for a client in the construction industry building an eCommerce application. We'll work through a set of imaginary feature requests of increasing complexity.  

## Tech Stack

In this demo we'll be writing code using *local* queries (termed LINQ to Objects) that target local object collections. Use of *intepreted* queries which are used for querying remote data sources by way of intermediate LINQ providers such as LINQ to SQL and Entity Framework are not covered in this article.

<details>
    <summary><i>Under the hood</i></summary>
    <blockquote>
        <P markdown="1">
        Local queries, which operate over collections implementing `IEnumerable<T>` resolve to LINQ query operators in the `System.Linq.Enumerable` class. 
        The delegates that they accept—whether expressed in query syntax, fluent syntax, or traditional delegates, are fully local to Intermediate Language (IL) code.
        Interpreted queries are descriptive. They operate over sequences that implement `IQueryable<T>` and they resolve to the query operators in the `Queryable` class, which emit expression trees that are interpreted at runtime.
        </P>
    </blockquote>
   
</details>

<br>

The classes and collections we'll be working with are as follows:

{% highlight c# %}
using System;
using System.Collections.Generic;

public class Product
{
    public string Name { get; set; }
    public string Category { get; set; }
    public decimal Price { get; set; }
}

var products = new List<Product>
{
    new Product { Name = "Wrench", Category = "Hand Tools", Price = 6.00m },
    new Product { Name = "Claw Hammer", Category = "Hand Tools", Price = 4.00m },
    new Product { Name = "Drill", Category = "Power Tools", Price = 40.00m },
    new Product { Name = "Jigsaw", Category = "Power Tools", Price = 60.00m },
    new Product { Name = "Padlock", Category = "Ironmongery", Price = 2.50m },
    new Product { Name = "Silicone", Category = "Sealants", Price = 15.00m }
};
{% endhighlight %}


Given the above lets get started on our first feature.

#### Feature 1 - Products broken down by category
The client wants a page showing a list of all products organised by category in the following format:
```
Category: <category_name>
 - Product: <product_name>
 - Product: <product_name>
Category: <category_name>
 - Product: <product_name>
 - Product: <product_name>
```

To achieve the above we can do the following in LINQ:  

{% highlight csharp %}
using System.Linq;

// Fluent / Method syntax
var result = products.GroupBy(p => p.Category);

{% endhighlight %}

Here we use the `GroupBy` operator, which organizes a flat input sequence into sequences of groups by dividing the source collection into chunks based on a given key.
This allows us to group collections of related data by specific properties and perform aggregation on the grouping.

<details>
    <summary><i>Under the hood</i></summary>
    <blockquote>
        <P markdown="1">
        If we check the signature of the `GroupBy` method we'll see that it accepts a `keySelector` delegate as `Func<Product,TKey>` this delegate is applied to each element in the input sequence to obtain a key which is what determines which group the element is associated with. 
        <br>
        The lambda expression we provide `p => p.Category` satisfies the delegate requirement.
        <br>
        `GroupBy` then reads the input elements into a temporary dictionary of lists so that all elements with the same key end up in the same sublist. <!--<sup>[1](#myfootnote1)</sup>-->
        <br>
        It then emits a sequence of groupings of type `IEnumerable<IGrouping<string,Product>>` 
        <br>
        A grouping is a sequence with a `Key` property, here the `Key` is `Product.Category`.
        <br>
        If we take a look at the contents of `result` in the VS Code Debug Console we can see this:
        <br>
        ![Alt](/assets/images/linq-in-depth-part1/linq_groupby_debug_1.PNG "linq_groupby_debug_1")
        <br>
        Where `GroupedEnumerable` is the internal concrete type `GroupBy` returns at runtime
        </P>
    </blockquote>
   
</details>

<br>

We used **Fluent Syntax** (sometimes referred to as Method Syntax) to write the above query which is the most fundamental way to write LINQ queries, the other way to write LINQ queries is to use **Query Syntax** (sometimes referred to as The LINQ query comprehension syntax). 

Here’s our preceding query written as a query expression using query syntax: 

{% highlight csharp %}
// Query syntax
var result = from p in products
             group p by p.Category;

{% endhighlight %}

Although they look similar, query expressions are not the same as SQL statements. Query expressions are in fact a syntactic shortcut for writing LINQ queries and the above query expression compiles down to the same IL code we wrote in the preceding query as the compiler converts query syntax into fluent syntax at compile time.

Lets enumerate the result:

{% highlight csharp %}

foreach (var grouping in result)
{
    Console.WriteLine($"Category: {grouping.Key}");
    foreach (Product product in grouping)
        Console.WriteLine($" - Product: {product.Name}");
}

/*Output:
Category: Hand Tools
 - Product: Wrench
 - Product: Claw Hammer
Category: Power Tools
 - Product: Drill
 - Product: Jigsaw
Category: Ironmongery
 - Product: Padlock
Category: Sealants
 - Product: Silicone
*/

{% endhighlight %}

As you can see this gives us the desired output.
<br> 

<!--<sub> *note the use of a nested foreach loop, we would'nt write that sort of code in a production application as it's not very efficient especially when working with large data sets, it was used here just to quickly show the results. </sub>-->

---

#### Feature 2 - Product and Price Breakdown
Ok so the client gave us something nice and easy for the first feature, now their requirements have got a little more complex, this time they want a page showing the categories with a breakdown of *total number* of products and *total price* in the following format:

```
Category: <category_name>
 - No Products: <total_products>
 - Total Price: $<total_product_price>
Category: <category_name>
 - No Products: <total_products>
 - Total Price: $<total_product_price>
```

To achieve this we can use `GroupBy` again along with some other LINQ operators: 

{% highlight csharp %}

var result = products.GroupBy(product => product.Category)
                     .Select(grouping => new
                     {
                         Category = grouping.Key,
                         ProductCount = grouping.Count(),
                         TotalPrice = grouping.Sum(x => x.Price)
                     });

{% endhighlight %}

Here we use `Select` to project the input sequence into a new output sequence: a sequence of __Anonymous types__

<details>
    <summary><i>Under the hood</i></summary>
    <blockquote>
        <P markdown="1">
        The `Select` operator emits a sequence where each input element is transformed or projected with a given lambda expression. 
        With Select, you always get the same number of elements that you started with, each element, however, can be transformed in any manner by the lambda function.
        Here we provide a lambda expression that projects each element into an Anonymous type which is created using an anonymous object creation expression.
        Anonymous types allow us to build objects that can be refered to in a statically typed way without having to declare a type beforehand
        <br>
        Like all the other query operators, `Select` never alters the input sequence; instead, it returns a new sequence. This is consistent with the functional programming paradigm, from which LINQ was inspired.
        </P>
    </blockquote>
   
</details>

<br>
We also take advantage of **Query Operator Chaining** which allow us to build more complex queries; data flows from left to right through the chain of operators, so the data is first grouped in the `GroupBy` method then result is passed into the `Select` method, which is then projected into our new sequence of Anonymous types. 

The above could also be written as two seperate queries:

{% highlight csharp %}
var grouped = products.GroupBy(product => product.Category);

var result = grouped.Select(grouping => new
{
    Category = grouping.Key,
    ProductCount = grouping.Count(),
    TotalPrice = grouping.Sum(x => x.Price)
});
{% endhighlight %}

This is known as **Progressive Query Building** and the resultant query is the same that you would get from a single-expression query, this is due to **Deferred Execution** which means the operators are executed not when they are constructed but when *enumerated*. 
Writing queries progressively however can sometimes make queries easier to write and can be more readable.

<details>
    <summary><i>Under the hood</i></summary>
    <blockquote>
        <P markdown="1">
        Query operators provide deferred execution by returning decorator sequences which are wrappers holding reference to the input sequence, lambda expression and any other arguments supplied. The wrapped input sequence is only enumerated when the decorator is enumerated.
        </P>
    </blockquote>
   
</details>

<br>
The contents of each Anonymous type element is made up of the grouping key (the category) along with a count of the products and the total price of products in each group, to get these values we use the `Count` and `Sum` operators.
<br>
The `Count` operator simply enumerates over a sequence, returning the number of items (here the sequence is the elements in the group). `Count` also has an overload that accepts a predicate (a delegate accepting an object and returning true or false) but that's not necessary here.

<details>
    <summary><i>Under the hood</i></summary>
    <blockquote>
        <P markdown="1">
        The internal implementation of the `Count` operator tests the input sequence to see whether it implements `ICollection<T>`. If it does, it simply calls `ICollection<T>.Count`, otherwise, it enumerates over every item, incrementing a counter. Here the former occurs as the input sequence is of type `System.Linq.Grouping` which implements `ICollection`
        </P>
    </blockquote>
   
</details>

<br>
The `Sum` operator is an aggregation operator that computes the sum of numeric values in a sequence. It has an overload that accepts a `selector` delegate which is used to transform each element, it can also be used to select an element to perform a summation on, which is what we do here with `x => x.Price`, we would get an error if we didn't supply this expression as `Sum` only works on numeric types.

<details>
    <summary><i>Under the hood</i></summary>
    <blockquote>
        <P markdown="1">
        `Sum` is fairly restrictive in it's typing, it's definition is hardwired to each of the numeric types (int, long, float, double, decimal, and their nullable versions).
        <br>
        Internally the sum of the sequence of values is obtained by simply looping round the sequence adding each element value (or result of the `selector` function) to a `result` variable.   
        </P>
    </blockquote>
   
</details>

<br>
The equivalent looks like this in query syntax:

{% highlight csharp %}

var result = from product in products
             group product by product.Category
             into grouping
             select new
             {
                 Category = grouping.Key,
                 ProductCount = grouping.Count(),
                 TotalPrice = grouping.Sum(x => x.Price)
             };

{% endhighlight %}

The `into` keyword here signals *query continuation* which allows us to continue a query after the `group` clause (a query ends after a `group` or `select` clause unless a *query continuation* clause is added).
<br>
You can also use `into` after a `select` clause, where it effectively “restarts” a query, allowing fresh `where`, `orderby`, and `select` clauses to be introduced.

The above could also be written using a Progressive Query Building approach:

{% highlight csharp %}

var grouped = from product in products
              group product by product.Category;

var result = from grouping in grouped
             select new
             {
                 Category = grouping.Key,
                 ProductCount = grouping.Count(),
                 TotalPrice = grouping.Sum(x => x.Price)
             };

{% endhighlight %}


>**Query Operator Chaining**, **Progressive Query Building**, and the `into` keyword are all **Chaining** strategies that produce identical runtime queries. 


Enumerating and printing the elements of `result` gives us the desired output:

{% highlight csharp %}

foreach (var grouping in result)
{
    Console.WriteLine($"Category: {grouping.Category}");
    Console.WriteLine($" - No Products: {grouping.ProductCount}");
    Console.WriteLine($" - Total Price: ${grouping.TotalPrice}");
}

/*Output:
Category: Hand Tools
 - No Products: 2
 - Total Price: $10.00
Category: Power Tools
 - No Products: 2
 - Total Price: $100.00
Category: Ironmongery
 - No Products: 1
 - Total Price: $2.50
Category: Sealants
 - No Products: 1
 - Total Price: $15.00
*/

{% endhighlight %}

---

#### Feature 3 - Search API  
The client wants a new search API building that returns products for a given search query. 
The API requirements are as follows:

- The API should allow users to search for products by one or more categories
- The results should include a summary section showing the total products, total categories and total price 
- The results should not include products that don't meet the search criteria

The API response should be in the following format:

```
{
  "products": [
    {
      "Name": "string",
      "Category": "string",
      "Price": 0
    }
  ],
  "totals": {
    "total_products": 0,
    "total_categories": 0,
    "total_price": 0
  }
}
```

For example given the search query: `["Hand Tools", "Sealants"]` 
<br>
The response should be:

```
{
  "products": [
    {
        "Name": "Wrench",
        "Category": "Hand Tools",
        "Price": 6.00
    },
    {
        "Name": "Claw Hammer",
        "Category": "Hand Tools",
        "Price": 4.00
    },
    {
        "Name": "Silicone",
        "Category": "Sealants",
        "Price": 15.00
    }
   ],
   "totals": {
    "total_products": 3,
    "total_categories": 2,
    "total_price": 25.00
   }
}
```

First lets look at how we can filter the products based on the search criteria. 
We can do that using the `Where` and `Any` operators:

{% highlight csharp %}
// searchQuery contains { "Hand Tools", "Sealants" }
              
var filteredProducts = products.Where(p => searchQuery.Any(c => c == p.Category));

{% endhighlight %}
The `Where` operator returns the elements from the input sequence that satisfy the given predicate - the `Any` operator in our case which we use to perform a *subquery* over each element in the input sequence individually. 

<details>
    <summary><i>Under the hood</i></summary>
    <blockquote>
        <P markdown="1">
        A subquery is a query contained within another query’s lambda expression. Subqueries are permitted because you can put any valid C# expression on the righthand
        side of a lambda. A subquery is simply another C# expression.This means that the rules for subqueries are a consequence of the rules for lambda expressions (and the behavior of query operators in general).
        </P>
    </blockquote>
   
</details>

<br>
The `Any` operator is a *Quantifier Operator* that returns true if the given expression is true *for at least one element*, in this case it returns true if any item in the `searchQuery` collection is equal to the value in the `Category` field of the element it is fed by `Where`.

So with the above query each element in the `products` collection is checked to see if the value in it's `Category` field is equal to any item in the `searchQuery` collection, if it is then it's included in the output of `Where`.

The above query looks like this in query syntax:

{% highlight csharp %}
// searchQuery contains { "Hand Tools", "Sealants" }

var filteredProducts = from p in products
                       where searchQuery.Any(c => c == p.Category)
                       select p;

{% endhighlight %}

That will give us what we need for the `products` part of the API response


For the `totals` part of the response we need to perform some aggregation on the results of our `Where` operation, we can use `Count` and `Sum` again for this:

{% highlight csharp %}
var filteredProducts = from p in products
                       where searchQuery.Any(category => category == p.Category)
                       select p;

var totalProducts = filteredProducts.Count();
var totalCategories = filteredProducts.Select(p => p.Category).Distinct().Count();
var totalPrice = filteredProducts.Sum(p => p.Price);

var result = new
{
    products = filteredProducts,
    totals = new
    {
        total_products = totalProducts,
        total_categories = totalCategories,
        total_price = totalPrice
    }
};

{% endhighlight %}

We've gone for a mixture of query and fluent syntax here to show how they can be combined. We've also used Progressive Query Building as it keeps the code readable and keeps the intention of the code clear.
We've also introduced anothe operator: `Distinct` which returns an input sequence stripped of duplicates, here we use it to remove duplicates of `Category` so that we can count the number of unique occurances of categories in the search results.

Lets have a look at the end result, to stay true to the API theme of this feature we'll serialize the resulting object into a JSON string using the `JsonSerializer` class which is part of the `System.Text.Json` namespace (.NET Core 3.0 onwards) and write it out to the console.

{% highlight csharp %}
using System.Text.Json;

var json = JsonSerializer.Serialize(result, new JsonSerializerOptions() { WriteIndented = true });

Console.WriteLine(json);

/*Output:
{
  "products": [
    {
      "Name": "Wrench",
      "Category": "Hand Tools",
      "Price": 6.00
    },
    {
      "Name": "Claw Hammer",
      "Category": "Hand Tools",
      "Price": 4.00
    },
    {
      "Name": "Silicone",
      "Category": "Sealants",
      "Price": 15.00
    }
  ],
  "totals": {
    "total_products": 3,
    "total_categories": 2,
    "total_price": 25.00
  }
}
*/

{% endhighlight %}

---

## In Summary

In this article we covered some of the common LINQ operators putting them to the test in a number of imaginary business scenarios.
We talked about how LINQ queries can be written using fluent syntax and query syntax and how ultimately both methods complement each other, compiling to the same IL code.

We also touched upon Query Operator Chaining and how this along with Progressive Query Building and the `into` keyword can be used to build more complex queries as well as how, as a consequence of Deferred Execution these chaining strategies produce identical runtime queries.

<!-- LINQ is a huge topic however and we have barely scratched the surface with this article, as mentioned at the beginning other areas such as intepreted queries will be covered in upcoming posts. -->

<!--Although a lot more additional features have been added to C# in the years since it was released, I think LINQ and it's supporting technologies are still as useful and relevant today as they were when they were first released back in 2007. -->

Well thats it for now. If you liked what you read or have any requests for other topics you'd like covered feel free to leave a comment below.

---

<br>

Bibliography 
============

<a name="myfootnote1">1</a>: Skeet, J. (2019) C# In Depth: Fourth Edition, Shelter Island, Manning Publications Co.
<br>
<a name="myfootnote2">2</a>: Albahari, J & B. (2017) C# 7.0 in a Nutshell: The Definitive Reference, USA, O'Reilly Media, Inc.
