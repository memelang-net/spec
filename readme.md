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

A ***meme*** comprises key-value pairs
* Analogous to a database row
* `\s+` whitespaces separate pairs
* `m=id` starting pair (analogous to a primary key)
* `;` terminating semicolon

A key-value ***pair*** comprises four parts
* ***Key operator*** (optional)
	* `!` negates the key name
* ***Key*** (optional)
	* Alphanumeric string
	* Analogous to a database column
	* Empty *key* is a query wildcard
	* Comma-separated list is an OR query
* ***Value operator*** (required)
	* `=` `!=` `>` `<` `<=` or `>=` 
	* No spaces between keys and operators
* ***Value*** (optional)
	* Integer, float, unquoted alphanumeric string, or CSV-style quoted string
	* Empty *value* is a query wildcard
	* Comma-separated list is an OR query
* Examples `K1<=V1` `K1,K2=V1` `K1=` `=V1,"val ""too"" two"` `=`

A ***comment*** is prefixed with double forward slashes `//`

## EBNF

```EBNF
(* Memelang v6 *)
memelang	::= { meme } ;
meme		::= term { div+ term } [div] ';' [div] ;
div			::= WS+ | comment ;
term		::= pair | join | unjoin ;
pair		::= [[keyopr] keys] valopr [values] ; (* no whitespaces between *)
keys 		::= key {',' key}
values 		::= value {',' value}
keyopr		::= '!' ;
valopr		::= '=' | '!=' | '<' | '>' | '<=' | '>=' ;
join		::= [key] '[' [key] ; (* no whitespaces between; no need for ']' *)
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

## Queries

Queries are partial memes with empty parts as wildcards. All query pairs are returned in the results.
* Empty *value* (`K1=`) matches all pairs for that *key*
* Empty *key* (`=V1`) matches all pairs for that *value*
* Empty *key* and *value* (`=`) matches all pairs in the meme
* The `!` key operator negates all *keys* in the list.
* The `!=` value operator negates all *values* in the list.


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
population>=100000 place=;
rating<4 actor= role=;

// Negation
K1,K2!=V1,V2	// key= (K1 or K2) and value!=(V1 or V2)
!K1,K2=V1,V2	// key!=(K1 or K2) and value= (V1 or V2)
!K1,K2!=V1,V2	// key!=(K1 or K2) and value!=(V1 or V2)
!K1,K2>V1		// key!=(K1 or K2) and value> V1

// Query for actors who are not Mark Hamill or Carrie Fisher
actor!="Mark Hamill","Carrie Fisher" role= movie=;

// Query for Mark Hamill for all keys except actor and producer
!actor,producer="Mark Hamill" movie=;

// Example response
m=100 actor="Mark Hamill" movie="Star Wars";
```

## Variables

Queries are read strictly left-to-right. Variables back-reference prior query pairs' *key* names (string) or *values* (int, float, string). Variables *cannot* be assigned. Variables *cannot* refer to forward pairs or backward implicit pairs. Variables *cannot* be inside quotes. All variables persist until a semicolon.

* `@v` *value* from one pair back 
* `@k` *key* name from one pair back
* `@key` *value* from last `key=` pair
	* Case-insensitive
	* Only populated when *key* is certain literal string (no wild or `!` or `,`)

Appending `:n` references *n* pairs back.

* `@v:1` = `@v` *value* one pair back
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
	* `@key` not populated due to uncertainty


```memelang
// Actor of titular movie role
role= movie=@v actor=;
role= movie=@role actor=;
role= actor= movie=@v:2;
role= actor= movie=@role;

// Variables may be used in comma lists
role= movie=@v,"Star Wars";

// Swap value into key name (unusual)
K1= @v=V2;
K1= @K1=V2;

// Swap key name into value (unusual)
=V1 K2=@k;
```

## Joins

Distinct items (actors, movies, etc.) usually occupy distinct memes with unique `m=id` identifiers. Joins match multiple memes. Joining is controlled with the `m` *key* and `@m` *value*.

* `m=@m` and `m=@m:1` stay in current meme (implicit default)
* `m!=@m` join to a different meme
* `m= ` join to any meme (current or different)
* `m=@m:n` *unjoin* to a prior meme

Shorthand `[` join reduces tokens.
* `K1[K2` expands to the most common join `K1= m!=@m K2=@v:2`
* The `:n` depth is measured *after* expansion, therefore `@v:2`
* `K1` and/or `K2` may be empty, joining on any key names
	* `K1[` expands to `K1= m!=@m =@v:2`
	* `[K2` expands to `= m!=@m K2=@v:2`
	* `[` expands to `= m!=@m =@v:2`
* Only join keys with similar values (person-person or year-year)
* *No* spaces between keys and bracket
* *No* need to close brackets, a semicolon closes all brackets.

The optional shorthand `]` *unjoins* so the query may start new joins from an earlier meme.
* `]` expands to `m=@m:2`, unjoins to the prior meme
* `]]` expands to `m=@m:3`, unjoins to two memes prior
* Number of unjoins must be <= number of joins; only unjoin *after* a join

After `[` and `m!=@m`, avoid redundant distinct conditionals like `actor!=@actor`.


```memelang
// Query for all of Mark Hamill's costars
actor="Mark Hamill" movie= m!=@m movie=@v:2 actor=;
movie= actor="Mark Hamill" m!=@m movie=@v:3 actor=;
actor="Mark Hamill" movie= m!=@m movie=@movie actor=;
movie= actor="Mark Hamill" m!=@m movie=@movie actor=;
actor="Mark Hamill" movie[movie actor=;

// Query for the actor's birthplace
actor= m!=@m person=@v:2 birthplace=;
actor= m!=@m person=@actor birthplace=;
actor[person birthplace=;

// Query for people born in the year their birthplace was founded
person= birthyear= m!=@m foundedyear=@v:2 place=;
person= birthyear= m!=@birthyear foundedyear=@birthyear place=;
person= birthyear[foundedyear place=;

// Query for the other movies Mark Hamill's costars have acted in
actor="Mark Hamill" movie= m!=@m movie=@v:2 actor= m!=@m actor=@v:2 movie=;
actor="Mark Hamill" movie= m!=@m movie=@movie actor= m!=@m actor=@actor movie=;
actor="Mark Hamill" movie[movie actor= actor[actor movie=;

// Query for the population of the birthplace of the Star Wars cast
movie="Star Wars" actor= m!=@m person=@v:2 birthplace= m!=@m place=@v:2 population=;
movie="Star Wars" actor= m!=@m person=@actor birthplace= m!=@m place=@birthplace population=;
movie="Star Wars" actor[person birthplace[place population=;

// Empty join keys: Query for every meme related to any value related to Mark Hamill
actor="Mark Hamill" = m!=@m =@v:2 =;
actor="Mark Hamill" [ =;
```

Joined queries return combined memes where each pair belongs to the preceding `m=` meme.

```memelang
m=100 actor="Mark Hamill" movie="Star Wars" m=101 movie="Star Wars" actor="Harrison Ford";
m=100 actor="Mark Hamill" movie="Star Wars" m=102 movie="Star Wars" actor="Carrie Fisher";
```

## Syntax Errors 

* Error `K1 = V1` spaces around value operator
* Error `K1=V1=V2` cannot chain values
* Error `K1=K2=K3=` missing spaces between pairs
* Error `K1=V1, V2` spaces after commas
* Error `K1, K2, K3=V1`	spaces after commas
* Error `K1[K2=X` errant value `=X`
* Error `K1=Y[K2` errant value `=Y`
* Error `K1=[K2` errant equals
* Error `K1[K2[K3` joins cannot chain
* Error `movie= m!=@m actor=@director` undefined variable `@director`

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
* Semantic Error `actor[birthplace` 
	* Must join similar values
	* Likely meant `actor[person birthplace=`
	* Correct similar joins `actor[actor` `actor[person` `birthyear=@foundedyear`
	* Incorrect dissimilar joins `actor[birthyear` `actor=@place` `rating= actor=@v`

## SQL Comparisons

Memelang queries are significantly shorter and clearer than equivalent SQL queries.

```
actor= role="Luke Skywalker","Han Solo" rating>4;
SELECT actor FROM movies WHERE role IN ('Luke Skywalker', 'Han Solo') AND rating > 4;

producer,actor="Mark Hamill","Harrison Ford" movie[movie actor=;
SELECT m1.actor, m1.movie, m2.actor FROM movies m1 JOIN movies m2 ON m1.movie = m2.movie WHERE m1.producer IN ('Mark Hamill', 'Harrison Ford') AND m1.actor IN ('Mark Hamill', 'Harrison Ford');
```

## Credits

Memelang was created by [Bri Holt](https://en.wikipedia.org/wiki/Bri_Holt) in a [2023 U.S. Provisional Patent application](https://patents.google.com/patent/US20250068615A1). ©2025 HOLTWORK LLC. Contact [info@memelang.net](mailto:info@memelang.net).