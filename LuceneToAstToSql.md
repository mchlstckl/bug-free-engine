# Filtering using AST to SQL

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
In order to check whether the AST *looked* like something reasonable, I created a formatter that printed the AST in lisp format.
Putting all the pieces together: the Lucene to AST parser, the AST Protobuf definition and the lisp formatter - we can do something like...

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



