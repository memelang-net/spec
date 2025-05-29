# Memelang v6

Memelang is the most token-efficient language for querying structured data, knowledge graphs, and retrieval-augmented generation pipelines. See the [Python GitHub repository](https://github.com/memelang-net/memesql6/).


## Examples

```memelang
m=100 actor="Mark Hamill" role="Luke Skywalker" movie="Star Wars" rating=4.5;
m=101 actor="Harrison Ford" role="Han Solo" movie="Star Wars" rating=4.6;
m=102 actor="Carrie Fisher" role=Leia movie="Star Wars" rating=4.2;

m=110 actor="Mark Hamill" role=Joker movie="Batman: Mask of the Phantasm" rating=4.7;
m=111 actor="Harrison Ford" role="Indiana Jones" movie="Raiders of the Lost Ark" rating=4.8;
m=112 actor="Carrie Fisher" role=Marie movie="When Harry Met Sally" rating=4.3;

m=200 person="Mark Hamill" birthyear=1951 birthplace="Oakland, CA";
m=201 person="Harrison Ford" birthyear=1942 birthplace="Chicago, IL";
m=202 person="Carrie Fisher" birthyear=1956 birthplace="Burbank, CA";

m=300 place="Oakland, CA" population=433000 climate=Mediterranean foundedyear=1852;
m=301 place="Chicago, IL" population=2740000 climate="Humid Continental" foundedyear=1833;
m=302 place="Burbank, CA" population=105000 climate=Mediterranean foundedyear=1887;
```

## Basics

A ***meme*** comprises key-value pairs, like a database row
* `\s+` whitespaces separate pairs
* `m=id` starting pair (like a primary key)
* `;` terminating semicolon

A key-value ***pair*** comprises four parts
* ***Key operator*** is *empty* or `!` (negates the key name)
* ***Key*** is *empty*, alphanumeric string, or comma-separated OR-list (like a database column)
* ***Value operator*** must be `=` `!=` `>` `<` `<=` or `>=` 
* ***Value*** is *empty*, integer, float, alphanumeric string, CSV-style quoted string, or comma-separated OR-list
* No whitespaces between keys/values/operators/commas in one pair

A ***comment*** is prefixed with double forward slashes `//`

## Queries

Queries are partial memes. Empty parts are wildcards. Each query pair is returned in the result.
* Empty *value* (`K1=`) matches all pairs for that *key*
* Empty *key* (`=V1`) matches all pairs for that *value*
* Empty *key* and *value* (`=`) matches all pairs in the meme
* Comma-separated *keys* match any listed *key*
* Comma-separated *values* match any listed *value*
* The `!` key operator negates all *keys* in the list
* The `!=` value operator negates all *values* in the list


```memelang
// Query for all movies with Mark Hamill as an actor
actor="Mark Hamill" movie=;

// Query for all relations and values from all memes relating to Mark Hamill:
="Mark Hamill" =;

// Value inequalities
population>100000 place=;
rating>=4.3 rating<=4.7 actor= role=;

// OR lists
K1,K2=V1,V2		// key= (K1 or K2) and value= (V1 or V2)
K1,K2!=V1,V2	// key= (K1 or K2) and value!=(V1 or V2)
!K1,K2=V1,V2	// key!=(K1 or K2) and value= (V1 or V2)
!K1,K2!=V1,V2	// key!=(K1 or K2) and value!=(V1 or V2)
!K1,K2>V1		// key!=(K1 or K2) and value> V1

// Query for (actor OR role) = ("Luke Skywalker" OR "Mark Hamill")
actor,role="Luke Skywalker","Mark Hamill" movie=;

// Query for actors who are not Mark Hamill or Carrie Fisher
actor!="Mark Hamill","Carrie Fisher" role= movie=;

// Query for Mark Hamill for all keys except actor and role
!actor,role="Mark Hamill" movie=;

// Example response
m=100 actor="Mark Hamill" movie="Star Wars";
```

## Simple Joins

Distinct items (actors, movies, etc.) usually occupy distinct memes with unique `m=id` identifiers. Joins match multiple memes. Using `K1[K2` joins a first meme to a second meme where the value of `K1` equals the value of `K2`. *No spaces* between the keys and brackets. *No need to close brackets*, a semicolon closes all brackets. Only join keys with similar values like `actor[actor` `actor[person` *not* dissimilar values like `actor[birthyear` `role[place`.

```memelang
// Query for all of Mark Hamill's costars
actor="Mark Hamill" movie[movie actor=;

// Query for the actor's birthplace
actor[person birthplace=;

// Query for people born in the year their birthplace was founded
person= birthyear[foundedyear place=;

// Query for the other movies Mark Hamill's costars have acted in
actor="Mark Hamill" movie[movie actor= actor[actor movie=;

// Query for the population of the birthplace of the Star Wars cast
movie="Star Wars" actor[person birthplace[place population=;
```

Joined queries return combined memes where each pair belongs to the preceding `m=` meme.

```memelang
m=100 actor="Mark Hamill" movie="Star Wars" m=101 movie="Star Wars" actor="Harrison Ford";
m=100 actor="Mark Hamill" movie="Star Wars" m=102 movie="Star Wars" actor="Carrie Fisher";
```

## Variables

Passing strictly left to right, each query pair pushes its matched *key* (string) and *value* (int, float, string) onto the variable stack. Rightward query pairs can back-reference prior matches. Variables *cannot* be assigned or forward-reference. Variables *cannot* be inside quotes. Variables stack until a semicolon wipes the stack. Variables almost always back-reference a prior meme in a complex join (below). However, they occasionally back-reference a pair from the current meme.

* `@v` *value* from one pair back 
* `@k` *key* name from one pair back
* `@key` *value* from last `key=` pair
	* Case-insensitive
	* Only stacked when query pair *key* is single true string (no wild or `!` or `,`)

Appending `:n` references *n* pairs back. One pair back is `:1` so `@v` = `@v:1`.

* `@v:2` *value* two pairs back
* `@k:3` *key* three pairs back
* `@key:11` *value* of `key=` with 10 intervening `key=` pairs

Examples variable values after the given pair.

* `movie="Star Wars"`
	* `@k` will be *movie*
	* `@v` and `@movie` will be *Star Wars*
* `movie=`
	* `@k` will be *movie*
	* `@v` and `@movie` will be the returned *value* like *Star Wars*
* `="Star Wars"` 
	* `@k` will be the returned *key* name like *movie*
	* `@v` will be *Star Wars*
	* `@key` not stacked due to non-string *key*


```memelang
// These are unusual examples of variable back-references within one meme

// Actor of titular movie role
role= movie=@v actor=;
role= movie=@role actor=;
role= actor= movie=@v:2;
role= actor= movie=@role;

// Variables may be used in comma lists
role= movie=@v,"Star Wars";

// Swap value into key name (very unusual)
K1= @v=V2;
K1= @K1=V2;

// Swap key name into value (very unusual)
=V1 K2=@k;
```

## Complex Joins

By default, a query pair stays within the current meme. However, after an `m` *key* query pair, the next query pair may match a different meme. Complex joins can be made using the `m` *key* and `@m` *value*. The `@m` variable is automatically populated with the current meme's `id`. 

* `m=@m` and `m=@m:1` stay in current meme (implicit default)
* `m!=@m` join to a different meme
* `m= ` join to any meme (current or different)

Typically, immediately after an `m` pair, the following pair back-references the prior meme. In fact, `K1[K2` is shorthand for the most common join `K1= m!=@m K2=@v:2`, where `K2`'s value in the new meme is `@v:2` which back-references two pairs to `K1`'s value in prior meme (variable depth is counted *after* expansion).

The `[` and `m!=@m` join to *different* memes, so trailing distinct conditionals like `actor!=@actor` are redundant and unnecessary.

```memelang
// Query for all of Mark Hamill's costars
actor="Mark Hamill" movie[movie actor=;
actor="Mark Hamill" movie= m!=@m movie=@v:2 actor=;
movie= actor="Mark Hamill" m!=@m movie=@v:3 actor=;
actor="Mark Hamill" movie= m!=@m movie=@movie actor=;
movie= actor="Mark Hamill" m!=@m movie=@movie actor=;

// Query for the actor's birthplace
actor[person birthplace=;
actor= m!=@m person=@v:2 birthplace=;
actor= m!=@m person=@actor birthplace=;

// Query for people born in the year their birthplace was founded
person= birthyear[foundedyear place=;
person= birthyear= m!=@m foundedyear=@v:2 place=;
person= birthyear= m!=@birthyear foundedyear=@birthyear place=;

// Query for the other movies Mark Hamill's costars have acted in
actor="Mark Hamill" movie[movie actor= actor[actor movie=;
actor="Mark Hamill" movie= m!=@m movie=@v:2 actor= m!=@m actor=@v:2 movie=;
actor="Mark Hamill" movie= m!=@m movie=@movie actor= m!=@m actor=@actor movie=;

// Query for the population of the birthplace of the Star Wars cast
movie="Star Wars" actor[person birthplace[place population=;
movie="Star Wars" actor= m!=@m person=@v:2 birthplace= m!=@m place=@v:2 population=;
movie="Star Wars" actor= m!=@m person=@actor birthplace= m!=@m place=@birthplace population=;

// Inequality join: Query for cities older than Burbank and with populations smaller
place="Burbank, CA" foundedyear= population= m!=@m population>@v:2 foundedyear<@v:4 place=;
place="Burbank, CA" foundedyear= population= m!=@m population>@population foundedyear<@foundedyear place=;
```

## Exotic Joins

These joins are very rare.

In `K1[K2`, `K1` and/or `K2` may be empty, joining on any key names.
* `K1[` expands to `K1= m!=@m =@v:2`
* `[K2` expands to `= m!=@m K2=@v:2`
* `[` expands to `= m!=@m =@v:2`

Brackets need not be closed. However, the optional shorthand `]` *unjoins* so the query may start new joins from an earlier meme. The number of unjoins must be less than number of joins; only unjoin *after* a join.
* `m=@m:n` unjoins to *n-1* memes prior
* `]` expands to `m=@m:2`, unjoins to one meme prior
* `]]` expands to `m=@m:3`, unjoins to two memes prior


```
// Empty join keys: Query for every meme related to any value related to Mark Hamill
actor="Mark Hamill" [ =;
actor="Mark Hamill" = m!=@m =@v:2 =;

// Query for each actor's birth place, unjoin, query for other roles in that movie 
actor[person birthplace= ] movie[movie role=
actor= m!=@m person=@v:2 birthplace= m=@m:2 movie= m!=@m movie=@v:2 role=
actor= m!=@m person=@actor birthplace= m=@m:2 movie= m!=@m movie=@movie role=
```

## EBNF

```EBNF
(* Memelang v6 *)
memelang	::= { meme } ;
meme		::= term { div+ term } [div] ';' [div] ;
div			::= WS+ | comment ;
term		::= pair | join | unjoin ;
pair		::= [[keyopr] keys] valopr [values] ; (* no spaces between *)
keys 		::= key {',' key} (* no spaces between *)
values 		::= value {',' value} (* no spaces between *)
keyopr		::= '!' ;
valopr		::= '=' | '!=' | '<' | '>' | '<=' | '>=' ;
join		::= [key] '[' [key] ; (* no spaces between; no need for ']' *)
unjoin		::= ']'+;
key		 	::= ALNUM+ | var ;
value		::= NUM | ALNUM+ | quoted | var ;
var			::= '@' ALNUM+ [':' DIGIT+] ;
quoted		::= '"' ( CHAR | '""' )* '"' ;
comment		::= '//' CHAR* ('\n' | EOF) ;
DIGIT		::= '0'..'9' ;
ALNUM		::= 'A'..'Z' | 'a'..'z' | DIGIT | '_' ;
NUM		 	::= ['-'] DIGIT+ ['.' DIGIT+] ;
WS 			::= ' ' | '\t' | '\r' | '\n' ;
CHAR		::= ? any Unicode character except '"' or '\n' ? ;
```

## Syntax Errors 

* Error `K1 = V1` space around value operator
* Error `K1=V1=V2` cannot chain values
* Error `K1=K2=K3=` missing spaces between pairs
* Error `K1=V1, V2` space after comma
* Error `K1, K2, K3=V1`	space after comma
* Error `K1[K2=X` errant value `=X`
* Error `K1=Y[K2` errant value `=Y`
* Error `K1=[K2` errant equals
* Error `K1[K2[K3` joins cannot chain

## Logic Errors 

* Warning `K1[ K2` 
	* Errant space after bracket
	* Likely meant `K1[K2`
* Warning `movie= movie=@v` 
	* Missing `m=` pair in join
	* Likely meant `movie= m!=@m movie=@v:2`
* Warning `actor="@v"`
	* Variable inside quotes
	* Likely meant `actor=@v`
* Warning `movie= m!=@m actor=` 
	* Missing variable reference to prior meme
	* Likely meant `movie= m!=@m movie=@v:2 actor=`
* Warning `movie= m!=@m movie=@v`
	* Wrong variable depth
	* Likely meant `movie= m!=@m movie=@v:2`
* Warning `movie= m!=@m movie=@v:1`
	* Wrong variable depth
	* Likely meant `movie= m!=@m movie=@v:2`
* Warning `actor[birthplace` 
	* Must join similar values
	* Likely meant `actor[person birthplace=`
	* Correct similar joins `actor[actor` `actor[person` `birthyear=@foundedyear`
	* Incorrect dissimilar joins `actor[birthyear` `actor=@place` `rating= actor=@v`
* Semantic Error `movie= m!=@m actor=@director`
	* Undefined variable `@director`
	* Likely meant `director= movie= m!=@m actor=@director`

## SQL Comparisons

Memelang queries are significantly shorter and clearer than equivalent SQL queries.

```
actor= role="Luke Skywalker","Han Solo" rating>4;
SELECT actor FROM movies WHERE role IN ('Luke Skywalker', 'Han Solo') AND rating > 4;

person,actor="Mark Hamill","Harrison Ford" movie[movie actor=;
SELECT m1.actor, m1.movie, m2.actor FROM movies m1 JOIN movies m2 ON m1.movie = m2.movie WHERE m1.person IN ('Mark Hamill', 'Harrison Ford') AND m1.actor IN ('Mark Hamill', 'Harrison Ford');
```

## Credits

Memelang was created by [Bri Holt](https://en.wikipedia.org/wiki/Bri_Holt) in a [2023 U.S. Provisional Patent application](https://patents.google.com/patent/US20250068615A1). ©2025 HOLTWORK LLC. Contact [info@memelang.net](mailto:info@memelang.net).