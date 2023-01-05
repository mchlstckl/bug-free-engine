# Filtering data-at-rest as well as in-flight using an AST

Some time ago I was working on a product that scans logs and identifies outliers, i.e. logs that didn't fit the usual patterns. 
These outliers were surfaced in some UI that allows the user to filter logs based on some criteria. 
These criteria were predefined and you could toggle them via some checkboxes. 
The filter expression was part of the request and forwarded to the backend that did a simple mapping to some SQL. 
We decided that perhaps we could make this aspect more flexible and enable users to freely query the logs. 
Unfortunatly there was no simple mapping from what is shown in the UI and the actual data schema. 
Additionally, it would be great if we could perform this type of filtering on the incoming log stream to create live views that automically update with whatever logs that match the filter expression.

Ok, cool. 
I wrote down some proposals on how we could approach this and asked colleagues for feedback. 
After a couple of iterations, the direction was set; create some AST of the filter expression that can be used in multiple contexts, such as querying Postgres, or in-stream processing for the live views. 
The AST representation should be readable from multiple languages, such as JavaScript and Go which we were using at the time.

What we wanted was something similar to the Lucene query syntax to define the filter. 
We were also considering using ElasticSearch eventually/maybe at the time. 
We would take a Lucene query, create the AST from said query, and generate SQL from the AST to query our existing Postgres instance. 

I started by iterating over the Lucene -> AST process. Which JS parser library I used to parse the Lucene syntax eludes me. 
It allowed me to write and test a parser in the browser and generated nice rail diagrams. 
I swear it was something SAP shared, but I (Google) couldn't find it anymore. 

The AST was generated using Protobuf. 
Each `message` was either some field that we could compare against some value, some logical operator, or a comparator. 
In order to check whether the AST *looked* like something reasonable, I created a formatter that printed the AST in Lisp format.
Putting all the pieces together: the Lucene to AST parser, the AST Protobuf definition and the Lisp formatter - we can do something like...

```javascript
let query = "score > 5 and system:payment"
let ast = parse(query)
print(ast)
```

```
(and 
  (gt score 5) 
  (eq system "payment"))
```

Exacly what the AST building blocks should be wasn't a given. 
In order to build something reasonable, I had to define what qualities an AST representation should posses for our purposes - mainly with regards to the developer experience. 
* Prefer flat over deep representations - easier to navigate the AST visually
* Strict in what comparators can be applied to what fields - prevent garbage AST's that do not produce a sane filter expression
* Able to be used in the multiple contexts - as mentioned above, support live views but primarily generate SQL 

Flat over deep meant that logical operators could take an array of expressions as opposed to just two corresponding to left-hand-side and right-hand-side. 
It took about three iterations of the AST building blocks to get something that felt good, something that another developer could use and not get annoyed.

Making sure that fields and comparators made sense was done in the parser itself, as far as I remember. 
For instance, we don't want to less-than on a field that is boolean.
The parser only allowed certain comparators for fields, anything that was outside of what the parser understood resulted in a parse error.

The multiple contexts requirement made me think about how we work with time windows. 
When constructing a SQL query, we tack on some `where created_at between x and y` clause that is part of the entire query.
However, this doesn't make sense in the context of operating in a stream where time is always now.
Looking at other query UI's, the time selection is usually not something you type in the query input, but stands by itself, separate from the query that you're about to execute.
Drawing inspiration from this, I cut out the time window part from the filter AST altogether and instead created the notion of a `Query` that contained the filter AST, as well as the time window data.

```javascript
// Dubious
let query = "score > 5 and system:payment and created_at > '2020-10-10T00:00:00' and created_at < '2020-10-11T00:00:00'"

// Less dubious
let filter = parse("score > 5 and system:payment")
let timeWindow = TimeWindow.last24h()
let query = { filter, timeWindow }
```

Next step was to generate the SQL given the AST so we could start using it to query the log data we had in Postgres.
As mentioned earlier, the naming in the UI didn't directly map to anything we had in the data schema.
I built a recursive traverser that walked over the AST and generated the SQL statements. 
Each `field` in the AST was handled with some non-trivial logic to build the corresponding SQL blocks.
This exercise made me wonder if this will actually work at all. 
A reasonable amount of self doubt crept in, but after a few days of making large changes, everything finally clicked into place.

At this stage, we had:
* A Lucene syntax parser that was used primarily for unit testing, exposing the Lucene like syntax to the user would come later (spoiler: never did)
* An AST representation of a log filter expression that could be used in multiple contexts (spoiler: only the SQL generation was used)
* An AST to SQL generator

Soon the company I worked for was sold and priorities changed which meant that a lot of the potential was not realized. 
Completely understandable, mainly due to the UI not coming along for the ride, instead using the UI that was exposed by the parent company.
