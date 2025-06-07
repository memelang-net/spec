# Memelang v7

Memelang is an AI-optimized language for querying structured data, knowledge graphs, and retrieval-augmented generation pipelines.

## Memes
* Comprises key-value pairs `<keyopr><keyset><valopr><valset>`
* Pairs separated by whitespaces `\s+`
* Terminates with semicolon `;`

### Pairs
* `<keyopr>` is either *(none)* or `!`
* `<key>` is alphanumeric string `movie`
* `<keyset>` is either
	* Single `<key>`
	* Single lone wildcard to match *any* key `*`
	* Variable (explained below)
	* CSV-style-comma-separated OR list `movie,role,@1`
* `<valopr>` is either `=` `>` `<` `>=` `<=` `!=`
* `<value>` is either
	* integer `-123`
	* or float `45.6`
	* or unquoted alphanumeric string `Leia`
	* or CSV-style double quoted string `"Anakin ""Ani"" Skywalker"`
* `<valset>` is either
	* Single `<value>`
	* Single lone wildcard to match *any* value `*`
	* Variable (explained below)
	* CSV-style-comma-separated OR list `Leia,"Anakin ""Ani"" Skywalker",@1`

### Others
* `->` shorthand for `m!=@m` (below)
* Comments start with `//` and end with `\n`

### Example Memes
* Stored meme pairs are always certain `<key>=<value>`
* First pair is always `m=<id>`

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

### Example Queries
* At least one pair in a query contains uncertainty

```memelang
// All movies with Mark Hamill as an actor
actor="Mark Hamill" movie=* rating>4 role=*;

// Response
m=100 actor="Mark Hamill" movie="Star Wars" rating=4.5 role="Luke Skywalker";
m=110 actor="Mark Hamill" movie="Batman: Mask of the Phantasm" rating=4.7 role=Joker;
```
```memelang
// All keys and values from all memes relating to Mark Hamill
// *=* matches all pairs in the meme
*="Mark Hamill" *=*;
```

### Example OR/Negation Queries
* `!` keyopr negates all **keys** in the list
* `!=` valopr negates all **values** in the list

```memelang
// OR–list semantics in set notation
K1,K2=V1,V2;	// (key ∈ {K1,K2}) ∧ (value ∈ {V1,V2})
K1,K2!=V1,V2;	// (key ∈ {K1,K2}) ∧ (value ∉ {V1,V2})
!K1,K2=V1,V2;	// (key ∉ {K1,K2}) ∧ (value ∈ {V1,V2})
!K1,K2!=V1,V2;	// (key ∉ {K1,K2}) ∧ (value ∉ {V1,V2})
!K1,K2>V1;		// (key ∉ {K1,K2}) ∧ (value > V1)
```
```memelang
// (actor OR role) = ("Luke Skywalker" OR "Mark Hamill")
actor,role="Luke Skywalker","Mark Hamill" movie=*;
```
```memelang
// Actors who are not Mark Hamill or Carrie Fisher
actor!="Mark Hamill","Carrie Fisher" role=* movie=*;
```
```memelang
// All keys, except actor and role, for Mark Hamill
!actor,role="Mark Hamill" movie=*;
```
```memelang
// FAILURES
K1=V1=V2;		// Error: cannot chain values
K1=*K2=*K3=X;	// Error: missing spaces between pairs
K1=*,V1,V2;		// Error: cannot mix wildcard and commas
actor="*";		// Warning: wildcard inside quotes is literal "*"
```

## Variables
Variables let later pairs retrieve the **keys** (list of strings) and **values** (list of integers/floats/strings) returned from prior pairs.
* Variables *cannot* be assigned.
* Variables *cannot* future-reference.
* Variables *cannot* reference objects or properties.
* Variables *cannot* be inside quotes.

### Variable Arrays
Simplified variable model in python:
```python
variables = {'K':[],'V':[]}

def var_add(keys: list, values: list):
	variables['K'].append(keys)
	variables['V'].append(values)
	if len(keys)==1: variables.setdefault(keys[0].lower(), []).append(values)

def index_get(n: str) -> int:
	if len(n)<1: return 1
	return int(n)

def var_get(tok):
	for sigil, mult, offset in (('#',1,-1), ('@',-1,0)):
		if tok.startswith(sigil):
			if tok.startswith(sigil+sigil):	return variables['K'][mult*index_get(tok[2:])+offset]
			if tok[1:].isdigit(): 			return variables['V'][mult*index_get(tok[1:])+offset]
			name, _, p = tok[1:].partition(':')
			return variables[name.lower()][mult*index_get(p)+offset]
```

How the variable arrays grow while processing the query:
```memelang
movie="Star Wars"	// 𝑝=1; variables={'K': [['movie']], 'V': [['Star Wars']], 'movie': [['Star Wars']]}
role=*				// 𝑝=2; variables={'K': [['movie'], ['role']], 'V': [['Star Wars'], [V1, V2, …]], 'movie': [['Star Wars']], 'role': [[V1, V2, …]]}
*>4.5;				// 𝑝=3; variables={'K': [['movie'], ['role'], [K1, K2, …]], 'V': [['Star Wars'], [V1, V2, …], [4.5]], 'movie': [['Star Wars']], 'role': [[V1, V2, …]]}
```

### Forward-Indexes
* Sigils `#` and `##` use absolute forward-indexing 𝑝
* Forward-index 𝑝 is a pair's absolute left-to-right query position
* Easy for AIs to count, hard for humans to maintain
* 𝑝≥1 and 𝑝 < current pair
* 𝑝=0 at `;` or BOF
* 𝑝++ for every query pair (including `m` and `->` pairs below)
* 𝑝=1 is the first pair
* 𝑝=2 is the second pair

#### Forward-Index Values
* `#𝑝` retrieves the **values** from the 𝑝th pair
* `#1` retrieves the **values** from the first pair 
* `#2` retrieves the **values** from the second pair

#### Forward-Index Keys
* `##𝑝` retrieves the **keys** from the 𝑝th pair
* `##1` retrieves the **keys** from the first pair
* `##2` retrieves the **keys** from the second pair

#### Forward-Index Keyname Values
* `#keyname:𝑝` retrieves the **values** from the 𝑝th `keyname` pair
* `#keyname` and `#keyname:1` retrieve the **values** from the first `keyname` pair
* *Only* populates when query **keys** is a single string (no `*` `!` `,`)
* Case-insensitive

### Backward-Indexes
* Sigils `@` and `@@` use relative backward-indexing 𝑞
* Backward-index 𝑞 counts backward from the current pair's query position 𝑞 = 𝑝₂ − 𝑝₁
* Harder for AIs to count, easier for humans to maintain
* 𝑞≥1 and 𝑞≤𝑝-1
* 𝑞=0 is current pair (*illegal*)
* 𝑞++ for every query pair back (including `m` and `->` pairs below)
* 𝑞=1 is the first pair back
* 𝑞=2 is the second pair back

#### Backward-Index Values
* `@𝑞` retrieves the **values** from the 𝑞th pair back
* `@1` retrieves the **values** from the first pair back 
* `@2` retrieves the **values** from the second pair back

#### Backward-Index Keys
* `@@𝑞` retrieves the **keys** from the 𝑞th pair back
* `@@1` retrieves the **keys** from the first pair back
* `@@2` retrieves the **keys** from the second pair back

#### Backward-Index Keyname Values
* `@keyname:𝑞` retrieves the **values** from the 𝑞th prior `keyname` pair
* `@keyname` and `@keyname:1` retrieve the **values** from the last `keyname` pair
* *Only* populates when query **keys** is a single string (no `*` `!` `,`)
* Case-insensitive

### Example Variables
Variables primarily retrieve prior pairs in joins (section below). However, variables occasionally retrieve pairs from the current meme (expert only, avoid).

```memelang
// Titular roles whose value equals the movie title
// Every variable retrieves role=*
role=* actor=* movie=@role;
role=* actor=* movie=#role;
role=* actor=* movie=@2;
role=* actor=* movie=#1;
actor=* role=* movie=@role;
actor=* role=* movie=#role;
actor=* role=* movie=@1;
actor=* role=* movie=#2;
```
```memelang
// Variable in comma list
role=* movie=@1,"Star Wars";
```
```memelang
// Variable swap value into key name
K1=* @1=V2; // @1 must be alphanumeric
K1=* @K1=V2;
```
```memelang
// Variable swap key name into value
*=V1 K2=@@1;
*=V1 K2=##1;
```
```memelang
// FAILURE
actor="@person"; // Warning: variable inside quotes
actor=@person; // Likely meant
```

## Joins
Distinct items (actors, movies, etc.) usually occupy distinct memes with unique `m=id` identifiers. By default, a pair stays within the current meme. A join query matches multiple memes by specifying a pair with **key** `m`, allowing the next pair to match a different meme.
* `m` **key** in a pair specifies which memes to join next
* `@m` is automatically populated with the current `m=<id>`
* `m=@m` and `m=@m:1` stay in current meme (implicit default)
* `m!=@m` (shorthand `->`) join to a different meme
* `m=*` join to any meme (current or different)

### Join Variables
***ALWAYS*** after an `m` pair, the following pair retrieves an uncertain pair from the prior meme.
* Preferred `@keyname`
	* `keyname` pair ***MUST*** be explict in prior meme
* Alternative `@2` is usually the correct **value** variable for the prior meme's last pair
* ***ONLY*** join keys with semantically similar **values**
	* YES `person=@author` or `birthyear=@foundedyear`
	* NO `person=@birthyear`
* `m!=@m` joins to *distinct* memes, so trailing distinct conditionals like `actor!=@actor` are generally redundant

```memelang
// The most common join is distinct
K1=*	// pair in first meme
m!=@m	// next pair must match a distinct meme
K2=@K1;	// pair in second meme, where K2 and K1 share a value
```
```memelang
// Wildcard m allows the current meme in the second joined meme 
K1=*	// pair in first meme
m=*		// next pair may match any meme
K2=@K1;	// pair in second meme, where K2 and K1 share a value
```
```memelang
// Wildcards in keys
*=*		// every pair in first meme
->		// (shorthand for m!=@m) next pair must match a distinct meme
*=@2;	// pair in second meme, any key has value from first meme
```

### Example Joins
```memelang
// All of Mark Hamill's costars - see how variables retrieve movie=* in their respective queries
// Each variable retrieves movie=*
actor="Mark Hamill" movie=* -> movie=@movie actor=*;
movie=* actor="Mark Hamill" -> movie=@movie actor=*;
actor="Mark Hamill" movie=* -> movie=@2 actor=*;
movie=* actor="Mark Hamill" -> movie=@3 actor=*;
actor="Mark Hamill" movie=* -> movie=#2 actor=*;
movie=* actor="Mark Hamill" -> movie=#1 actor=*;

// Response - join queries return combined memes, each pair belongs to the preceding m=<id>
m=100 actor="Mark Hamill" movie="Star Wars" m=101 movie="Star Wars" actor="Harrison Ford";
m=100 actor="Mark Hamill" movie="Star Wars" m=102 movie="Star Wars" actor="Carrie Fisher";
```
```memelang
// People born in the year their birthplace was founded - join on int birthyear is year = foundedyear is year
person=* birthyear=* -> foundedyear=@birthyear place=*;
person=* birthyear=* -> foundedyear=@2 place=*;
```
```memelang
// Every meme related to any value related to Mark Hamill - wild join keys (expert, avoid)
actor="Mark Hamill" *=* -> *=@2 *=*;
```

### Example Multi-joins
```memelang
// Other movies Mark Hamill's costars have acted in - multi join
actor="Mark Hamill" movie=* -> movie=@movie actor=* -> actor=@actor movie=*;
actor="Mark Hamill" movie=* -> movie=@2 actor=* -> actor=@2 movie=*;
```
```memelang
// Population of the birthplace of the Star Wars cast
movie="Star Wars" actor=* -> person=@actor birthplace=* -> place=@birthplace population=*;
movie="Star Wars" actor=* -> person=@2 birthplace=* -> place=@2 population=*;
```
```memelang
// Cities older than Burbank and with larger populations - inequality join
place="Burbank, CA" foundedyear=* population=* -> population>@population foundedyear<@foundedyear place=*;
place="Burbank, CA" foundedyear=* population=* -> population>@2 foundedyear<@4 place=*; // @4 = 1887 (Burbank foundedyear)
```

### Failure Joins
```memelang
movie=* movie=@1; // Warning: missing m= pair in join
movie=* -> movie=@2; // Likely meant
```
```memelang
movie=* -> actor=*; // Warning: missing retrieved variable
movie=* -> movie=@2 actor=*; // Likely meant
```
```memelang
movie=*->movie=@2; // Error: missing spaces around ->
movie=* -> movie=@2; // Likely meant
```
```memelang
movie=* -> movie=@1; // Warning: wrong index - @1 retrieves -> pair
movie=* -> movie=@2; // Likely meant
```
```memelang
birthplace=* person=* -> actor=@birthplace; // Warning: joined dissimilar values - actor != birthplace
person=* -> actor=@person birthplace=*; // Likely meant
```
```memelang
movie=* -> actor=@director; // Semantic Error: undefined variable @director
director=* movie=* -> actor=@director; // Likely meant
```
```memelang
director=* movie=* -> actor=@director:2; // Semantic Error: undefined variable @director:2 - wrong index
director=* movie=* -> actor=@director; // Likely meant
```

## Credits
[Bri Holt](mailto:info@memelang.net) · [Patents Pending](https://patents.google.com/patent/US20250068615A1) · ©2025 HOLTWORK LLC