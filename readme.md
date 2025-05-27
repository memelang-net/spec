# Memelang v6

Memelang is the most token-efficient language for querying structured data, knowledge graphs, and retrieval-augmented generation pipelines. See the [Python GitHub repository](https://github.com/memelang-net/memesql6/).

## Basics

```memelang
// Example memes for the Star Wars cast
m=123 actor="Mark Hamill" role="Luke Skywalker" movie="Star Wars" rating=4.5;
m=456 actor="Harrison Ford" role="Han Solo" movie="Star Wars" rating=4.6;
m=789 actor="Carrie Fisher" role=Leia movie="Star Wars" rating=4.2;
```

* A ***meme*** comprises key-value pairs separated by spaces analogous to a database row
* A key-value ***pair*** comprises `<key_opr><key><value_opr><value>`, commonly `key=value`
* A ***key*** is an alphanumeric string analogous to a database column
* A ***value*** is an integer, decimal, alphanumeric string, or CSV-style quoted string; analogous to a database cell value
* `m=id` is always the first pair in a meme; analogous to a database primary key
* `\s+` Whitespaces collapse to a single space
* `;` Semicolons terminate statements
* `,` commas form ***or*** lists for both *keys* and *values*
* `!` is an optional *key* operator that matches any keys *except* those listed
* `=` `!=` `>` `<` `<=` `>=` are *value* operators
* `//` Double forward slashes prefix comments

This EBNF formally describes the language for programatic parsing.

```EBNF
(* Memelang v6 *)
memelang		::= { meme } ;
meme			::= pair { div pair } [div] ';' ;
div				::= { ( WS | comment ) } ;
pair			::= kv_pair | join_sugar | unjoin_sugar ;
kv_pair			::= [ [ key_opr ] key_group ] value_opr [ value_group ] ;
key_group		::= key { ',' key } ;
value_group		::= value { ',' value } ;
key_opr			::= '!' ;
value_opr		::= '=' | '!=' | '<' | '>' | '<=' | '>=' ;
join_sugar		::= [ key_group ] '[' [ key_group ] ;
unjoin_sugar	::= ']' { ']' } ;
key				::= ALNUM+;
value			::= NUM | ALNUM+ | quoted | variable ;
variable		::= '@' ('v' | 'k' | 'm' | ALNUM+) [ ':' DIGIT+ ] ;
quoted			::= '"' { CHAR | '""' } '"' ;
comment			::= '//' { CHAR } '\n' ;
WS				::= { ' ' | '\t' | '\r' | '\n' } ;
DIGIT			::= '0'..'9' ;
ALNUM			::= 'A'..'Z' | 'a'..'z' | DIGIT | '_' ;
NUM				::= '-'? DIGIT+ ['.' DIGIT+];
CHAR			::= ? any Unicode character except '"' or newline ? ;
```

## Queries

Queries are partial memes with empty parts as wildcards. All explicit pairs in the query are returned in the results.
* Empty *value* (`key=`) matches all pairs for that *key*
* Empty *key* (`=value`) matches all pairs for that *value*
* Empty *key* and *value* (`=`) matches all pairs in the meme

```memelang
// Query for all movies with Mark Hamill as an actor
actor="Mark Hamill" movie=;

// Query for all relations involving Mark Hamill
="Mark Hamill";

// Query for all relations and values from all memes relating to Mark Hamill:
="Mark Hamill" =;

// Query for (actor OR producer) = (Mark OR "Mark Hamill")
actor,producer=Mark,"Mark Hamill";

// Value inequalities
height>=1.6;
width<2;

// Value negation: match actors who are not Mark Hamill or Mark
actor!="Mark Hamill",Mark;

// Key negation: match all keys except actor and producer for Mark Hamill
!actor,producer="Mark Hamill";
```

## Variables

Variables refer backward to explicit query pairs. Variables *cannot* refer forward to future pairs or refer to implicit pairs. Variables *cannot* be inside quotes.

* `@v` *value* from one pair prior
* `@k` *key* from one pair prior
* `@keyname` *value* from last seen case-insensitive `keyname=` pair

Appendding `:n` references *n* pairs prior.

* `@v:2` *value* two pairs prior
* `@k:3` *key* three pairs prior
* `@keyname:11` *value* of `keyname=` with 10 intervening `keyname=` pairs


```memelang
// Two different keys have the same value
K1= K2=@v;
K1= K2=@K1;

// Swap value into key name
K1= @v=V2;
K1= @K1=V2;

// Swap key name into value
=V1 K2=@k;

// Variables may be used in comma lists
K1= K2=Jeff,@v;
K1= K2=Jeff,@K1;

// A movie titled with an actor's name
actor= year= movie=@v:2
actor= year= movie=@actor
```

## Joins

Distinct items (actors, movies, etc.) usually occupy distinct memes with unique `m=id` identifiers. Joins match multiple memes. Joining is controlled with the `m` *key* and `@m` *value*.

* `m=@m` stay in current meme (implicit default)
* `m!=@m` join to a different meme
* `m= ` join to any meme (current or different)
* `m=@m:n` *unjoin* to a prior meme

Shorthand joins reduce tokens.
* `K1[K2` is shorthand for `K1= m!=@m K2=@v:2`
* `K1` and `K2` must have the same value domain (person, book, year, etc.)
* `]` is shorthand for `m=@m:2`, which unjoins to the prior meme
* `]]` is shorthand for `m=@m:3`, which unjoins to two memes prior


```memelang
// Join any memes where K1 and K2 have the same value
K1= m= K2=@v:2;

// Join two different memes, unjoin, join a third meme
K1= m!=@m K2=@v:2 m=@m:2 K3= m!=@m K4=@v:2;

// Unjoins may be sequential
K1= m!=@m K2=@v:2 K3= m!=@m K4=@v:2 m=@m:3 K5=;

// Join two different memes on K1=K2, unjoin, then join the first meme to another where K4=K5
K1= m!=@m K2=@v:2 K3= m=@m:2 K4= m!=@m K5=@v:2;

// Query for a meta-meme, K2's value is K1's meme ID
K1=V1 m= K2=@m;

// Query for all of Mark Hamill's costars
actor="Mark Hamill" movie= m!=@m movie=@v:2 actor=;

// Generic shorthand example
K1=V1 K2[K3 K4>V4 K5=;

// Query for all of Mark Hamill's costars
actor="Mark Hamill" movie[movie actor=;

// Query for all movies in which both Mark Hamill and Carrie Fisher act together
actor="Mark Hamill" movie[movie actor="Carrie Fisher";

// Query for anyone who is both an actor and a producer
actor[producer;

// Query for a second cousin: child's parent's cousin's child
child= parent[cousin parent[child;

// Keys omitted, joins any value from the present meme to that value in another meme
K1=V1 [ K2=V2;

// Join two different memes, unjoin, join a third meme
K1[K2 K3= ] K4[K5;

// Unjoins may be sequential
K1[K2 K3[K4 K5= ]] K6=;
```

Joined queries return combined memes where each pair belongs to the preceding `m=` meme.

```memelang
m=123 actor="Mark Hamill" movie="Star Wars" m=456 movie="Star Wars" actor="Harrison Ford";
m=123 actor="Mark Hamill" movie="Star Wars" m=789 movie="Star Wars" actor="Carrie Fisher";
```

## Pitfalls

* Use spaces between pairs `title= author= year=` not `title=author=year=`
* Exclude spaces in comma lists `X,"Y",Z` not `X, "Y", Z`
* After `[` or `m!=@m`, avoid redundant distinct conditionals like `actor!=@actor`
* Use correct variable depth `movie= m!=@m movie=@v:2` not `movie= m!=@m movie=@v`
* After a join, at least one variable must reference the prior meme `movie= m!=@m movie=@v:2 actor=` not `movie= m!=@m actor=`
* The correct join shorthand is `K1[K2` not `K1[K2=@v` not `K1[K2=X` not `K1[ K2` not `K1=[K2`
* Join only comparable domains such as `actor[director` not `actor[year`
* Variables only reference earlier explicit query pairs such as `birthplace= person="Mark Hamill" m!=@m place=@birthplace;` not `person="Mark Hamill" m!=@m place=@birthplace;`


## SQL Comparisons

Memelang queries are significantly shorter and clearer than equivalent SQL queries.

```memelang
actor= role="Luke Skywalker","Han Solo" rating>4;
SELECT actor FROM movies WHERE role IN ('Luke Skywalker', 'Han Solo') AND rating > 4;

producer,actor="Mark Hamill","Harrison Ford" movie[movie actor=;
SELECT m1.actor, m1.movie, m2.actor FROM movies m1 JOIN movies m2 ON m1.movie = m2.movie WHERE m1.producer IN ('Mark Hamill', 'Harrison Ford') AND m1.actor IN ('Mark Hamill', 'Harrison Ford');
```

## Credits

Memelang was created by [Bri Holt](https://en.wikipedia.org/wiki/Bri_Holt) in a [2023 U.S. Provisional Patent application](https://patents.google.com/patent/US20250068615A1). ©2025 HOLTWORK LLC. Contact [info@memelang.net](mailto:info@memelang.net).