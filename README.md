Multi-model databases (with an emphasis on ArangoDB)
====================================================

This repository contains the presentation slides and examples for a talk I gave at the Gothenburg Tech Radar meetup in Gothenburg, SE in January 2019.

Files in this repository
------------------------

* [presentation.pdf](https://github.com/jsteemann/GTR/blob/master/presentation.pdf) contains the slides for the presentation.
* [presentation-including-foxx-example.pdf](https://github.com/jsteemann/GTR/blob/master/presentation-including-foxx-example.pdf) contains the presentation plus some extra slides that I removed to keep the length of the talk under control :smile:
* [dump.tar.gz](https://github.com/jsteemann/GTR/blob/master/dump.tar.gz) contains a dump of the database I used for the examples
* [employees_0.0.1.zip](https://github.com/jsteemann/GTR/blob/master/employees_0.0.1.zip) contains the Foxx demonstration app used in the presentation with the Foxx examples. It can be installed locally to play with the example Foxx app.


How to setup the example data
-----------------------------

To test the examples I used in the talk you can restore the dump from `dump.tar.gz` into a running ArangoDB database.
First, extract the tar archive of the dump to a directory of your choice. Then run

```bash
arangorestore --server.endpoint tcp://127.0.0.1:8529 --server.username root /path/to/dump/directory --include-system-collections true
```
(please adjust server endpoint and credentials as necessary). 


Example queries used in the presentation
----------------------------------------

```
/*
   get all data for the categories "books" and "kitchen"
   SQL equivalent:
   SELECT c.* FROM categories c WHERE c._key IN ... 
*/
FOR c IN categories
  FILTER c._key IN [ 'books', 'kitchen' ]
  RETURN c
```

```
/* 
   run a projection on the categories and sort them
   SQL equivalent:
   SELECT c._key, c.title FROM categories c ORDER BY c.title
*/
FOR c IN categories
  SORT c.title
  RETURN {
    _key: c._key,
    title: c.title
  }
```

```
/* 
   joining data from two collections
   note: in real life, the "category" attribute should be indexed to speed up the query!
   SQL equivalent:
   SELECT c.* AS category, p.* AS product FROM categories c, products p WHERE p.category = c._key ORDER BY c.title, p.name
*/
FOR c IN categories
  FOR p IN products
    FILTER p.category == c._key 
    SORT c.title, p.name
    RETURN {
      category: c,
      product: p
    }
```

```
/*
   subquery example
   this will return an array of product details per category
   SQL equivalent:
   no supported in all relational databases, because this will return a nested data structure
   (a variable-length array of products for each category)
*/
FOR c IN categories
  RETURN {
    category: c.title,
    products: (
      FOR p IN products
        FILTER p.category == c._key
        SORT p.name
        RETURN { _key: p._key, name: p.name }
    )
  }
```

```
/* 
   aggregation example
   this will return the sales per year and product and add them up
   SQL equivalent:
   SELECT s.year, s.product, SUM(c.count) AS count, SUM(s.amount) AS amount FROM sales s WHERE s.year BETWEEN 2017 AND 2019 GROUP BY s.year, s.product
*/
FOR s IN sales
  FILTER s.year >= 2017 AND s.year <= 2019
  COLLECT   year    = s.year,
            product = s.product
  AGGREGATE count   = SUM(s.count),
            amount  = SUM(s.amount)
  RETURN {
    year, product: DOCUMENT('products', product), count, amount
  }
```

```
/* 
   data modification example
   update all documents which don't have a "lastModified"
   attribute, store the current date in it, and return the _keys
   of the documents updated
   SQL equivalent:
   not easily possible to add an attribute at runtime using a DML
   statement. this would rather require an ALTER TABLE command in SQL.
*/
FOR p IN products
  FILTER !HAS(p, "lastModified")
  UPDATE p WITH { 
    lastModified: DATE_ISO8601(DATE_NOW()) 
  } IN products
  RETURN OLD._key
```

```
/*
   data removal example
   delete documents of the specified categories, and return
   all data of the deleted documents in the same statement
   SQL equivalent:
   DELETE FROM products WHERE category IN ["toys", "kitchen"]
   (however, this will not return the removed products)
FOR p IN products
  FILTER p.category IN [ "toys", "kitchen" ]
  REMOVE p IN products RETURN OLD
```

```
/* 
   graph traversal example
   find all employees that "sofia" is a direct manager of
*/
FOR emp IN 1..1 OUTBOUND 'employees/sofia' isManagerOf
  RETURN emp._key
```

```
/* 
   graph traversal example
   find direct and indirect subordinates of "sofia"
*/
FOR emp IN 1..5 OUTBOUND 'employees/sofia' isManagerOf
  RETURN emp._key
```

```
/*
   graph traversal example
   find employees that "olaf" is indirectly connected to (level 2)
*/
FOR emp IN 2..2 ANY 'employees/olaf' isManagerOf
  RETURN emp
```

```
/*
   graph traversal example
   find all line managers of employee "olaf", calculating distance to him
*/
FOR emp, edge, path IN 1..5 INBOUND 'employees/olaf' isManagerOf
  RETURN {
    who: emp._key,
    level: LENGTH(path.edges)
  }
```

Some more information on ArangoDB
---------------------------------

The documentation for ArangoDB can be found on the [ArangoDB documentation website](https://www.arangodb.com/documentation/).

The documentation for AQL can be found [here](https://docs.arangodb.com/3.4/AQL/).

Some online courses for ArangoDB can be found in the [ArangoDB training center](https://www.arangodb.com/arangodb-training-center/).
Specific graph-related training material can be found [here](https://www.arangodb.com/arangodb-training-center/graphs/).

Please note that for taking the courses it may be required to register on the ArangoDB website first.
