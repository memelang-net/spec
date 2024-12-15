# MEMELANG

Memelang is a notation for efficiently encoding and querying knowledge. In Memelang, the smallest unit of knowledge is a *meme*. A meme states some idea `A` has some relation `R` to some other idea `B`.

`A.R:B = Q;`

A breakdown of each expression:

| Meme Syntax | Explanation                                               |
|-------------|-----------------------------------------------------------|
| `A`         | The A starts the statement without punctuation.           |
| `.R`        | The relation is preceded by a dot.                        |
| `:B`        | The B ends the statement preceded by a colon.             |
| `=Q`        | Where Q=1 is true and Q=0 is false.                       |
| `;`         | The statement is closed with a semi-colon.                |

For example, "Alice's uncle is Bob," where `A=Alice`, `R=uncle`, and `B=Bob`. Memelang encodes this relation thusly:

`Alice.uncle:Bob = 1;`

## Relation Chains

Memelang supports chained relations, where the relation "uncle" can be expressed as "parent’s brother."

`Alice.uncle:Bob = Alice.parent.brother:Bob;`

## Relation Rules

To define general rules, use `==` to indicate equivalence between relations:

`.uncle == .parent.brother;`

## Inverse Relations

Given that Alice's uncle is Bob, then Bob's niece is Alice. This inverse relation is expressed by swapping A and B and using an apostrophe instead of a dot:

`Alice.uncle:Bob = Bob'uncle:Alice;`

Relation chains are inverted by inverting each relation *and* reversing their order:

`'uncle == 'brother'parent;`

All of this notation can be summarized as:

~~~
// Given
.uncle == .parent.brother;

// Then these statements are equivalent
Alice.uncle:Bob = 
  Bob'uncle:Alice = 
  Alice.parent.brother:Bob =  
  Bob'brother'parent:Alice;
~~~

## Range Quantities

`Q` can represent decimal quantities as well as true/false. When `Q` is a quantity, `R` is a *range* relation and `B` is a *unit*.

`element.range:unit = quantity;`

For example, "Alice's height is 1.6 meters" is encoded:

`Alice.height:meter = 1.6;`

## Inverse Ranges

The inverse of a range relation inverts the quantity to `1/Q`. Using the prior example, the inverted relation states that "one meter's height is 1/1.6 Alices."

~~~
Alice.height:meter = 1.6;
Alice'height:meter = 1/1.6 = 0.625;
meter'height:Alice = 1.6;
meter.height:Alice = 1/1.6 = 0.625;
~~~

## Query Operations

Known memes are stored in the database where they can be queried. A query consists of an incomplete Memelang statement and returns a set of memes. Try the [Memelang query demo](//demo.memelang.net/).

| Query        | Explanation                                   |
|--------------|-----------------------------------------------|
| `Alice`      | Finds all memes related to Alice.             |
| `Alice.uncle`| Finds all of Alice's uncles.                  |
| `.uncle:Bob` | Finds all nieces/nephews of Bob.              |
| `Alice:Bob`  | Finds all direct relationships between them.  |

## Quantity Comparisons

Comparison operators filter the preceding set according to the `Q` values. Herein, `M` and `N` denote statements such as `A.R:B`.

| Query    | Explanation                                              |
|----------|----------------------------------------------------------|
| `M`/`M=t`| Default true, finds memes where `Q!=0`.                  |
| `M=Qx`   | Finds memes where `Q` equals `Qx`.                       |
| `M>Qx`   | Finds memes where `Q` is greater than `Qx`.              |
| `M<Qx`   | Finds memes where `Q` is less than `Qx`.                 |
| `M>=Qx`  | Finds memes where `Q` is >= `Qx`.                        |
| `M<=Qx`  | Finds memes where `Q` is <= `Qx`.                        |
| `M!=Qx`  | Finds memes where `Q` does NOT equal `Qx`.               |
| `M=f`    | Finds "false" memes (`Q=0`).                             |

## AND Queries

To perform an AND query with multiple conditions, simply separate each condition with a space. This filters for all `A` that match every query condition. For example, to find individuals whose uncle is Bob and whose height is less than 1.6 meters:

`.uncle:Bob .height:meter<1.6`

To find individuals whose uncle is Bob and height is less than 1.6 meters, but excludes those whose aunt is Cindy or who attended college:

`.uncle:Bob .height:meter<1.6 .aunt:Cindy=f .college=f`

## GET Queries

By default, a query returns only the memes relevant to the search criteria. Adding `M=g` returns additional memes for the matching `A` set. For example, to also return weight:

`.uncle:Bob .height:meter<1.6 .aunt:Cindy=f .college=f .weight:kilogram=g`

To get all memes for the matching `A` set, use `qry.all`:

`.uncle:Bob .height:meter<1.6 .aunt:Cindy=f .college=f qry.all`

## OR Queries

Memelang allows OR grouping by setting statements equal to `t` followed by a number where the number groups OR conditions. For example, to find `A` whose uncle is Bob, and whose aunt is Cindy or Diana:

`.uncle:Bob .aunt:Cindy=t1 .aunt:Diana=t1`

## Development

Available development tools:

- [Memelang query demo](//demo.memelang.net/)
- [GitHub: PHP Script to convert Memelang queries to SQL](//github.com/memelang-net/meme-sql-php)

We need you to help us develop Memelang projects such as:

- Database plugins (MySQL, PostgreSQL, SQLite)
- Memelang-specific database systems
- Blockchain integration for immutable storage
- Visualizers and proof assistants

Changelog from [Version 01](/01/)

- Renamed the language from *Memetics* to *Memelang*.
- Shortened `true` `false` to `1` `0` or `t` `f` respectively.
- Replaced the term *measure* relation with *range* relation.
- Inverting a range relation inverts the quantity.
- Added GET query `M=g`.
- Replaced OR operator `|` with grouping `Q=t1`.
- Replaced query set junction `&` with whitespaces.
- Streamlined features for clarity.

## License

Memelang is free for public use under the [Memelicense](//memelicense.net/). Patents pending. Copyright 2024 HOLTWORK LLC. Contact [info@memelang.net](mailto:info@memelang.net).
