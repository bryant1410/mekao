# mekao

This library will help you to construct SQL queries. It have no other
facilities then this (no pools, or db specific code, just plain SQL).
Generated queries are not vendor specific.

Main assumption is that records are used to represent DB data.


# ToC
1.  [Status](#status)
2.  [Basic usage](#usage)
3.  [Records](doc/records.md#toc)
    * [#mekao_settings{}](doc/mekao_records.md#mekao_settings)
    * [#mekao_table{}](doc/mekao_records.md#mekao_table)
    * [#mekao_column{}](doc/mekao_records.md#mekao_column)
    * [#mekao_query{}](doc/mekao_records.md#mekao_query)

# Status

Alpha. It is under development, any feedback are appreciated!

[![Build Status](https://secure.travis-ci.org/ddosia/mekao.png?branch=master)](http://travis-ci.org/ddosia/mekao)

# Usage

Suppose that we have table `books` in our SQL db:

| Column    | Type    | Attributes                  |
|-----------|---------|-----------------------------|
| id        | int     | Primary key, read only      |
| isbn      | varchar |                             |
| title     | varchar |                             |
| author    | varchar |                             |
| created   | created | Read-only                   |

We could create a record, which will hold the data from this table:

```erlang
-record(book, {id, isbn, title, author, created}).
```

Normally we create a function to fetch the data from DB and transform it to
record:

```erlang
fetch_book_by_id(Id) ->
    Q = "SELECT * FROM books WHERE id = ?",
    db:fetch(Q, [int], [Id]).

fetch_book_by_isbn(ISBN) ->
    Q = "SELECT * FROM books WHERE isbn = ?",
    db:fetch(Q, [varchar], [ISBN]).
```
Now you see similarity in both queries and want somehow to generalize this.
Here comes *mekao*.

```erlang
fetch_book_by_id(Id) ->
    fetch_book(#book{id = Id, _ = '$skip'}).

fetch_book_by_isbn(ISBN) ->
    fetch_book(#book{isbn = ISBN, _ = '$skip'}).

fetch_book(Book) ->
    {ok, #mekao_query{
        body = Q, types = Types, values = Vals
    }} = mekao:select(Book, ?TABLE_BOOKS, ?S),
    db:fetch(Q, Types, Vals).
```

Probably you would ask, what is this `?TABLE_BOOKS` and `?S`
macro means?
First is our table definition and second is some general settings. It could be
defined like this:

```erlang
-include_lib("mekao/include/mekao.hrl").

-define(TABLE_BOOKS, #mekao_table{
    name    =  <<"books">>,
    columns = [
        #mekao_column{name = <<"id">>, type = int, key = true, ro = true},
        #mekao_column{name = <<"isbn">>, type = varchar},
        #mekao_column{name = <<"title">>, type = varchar},
        #mekao_column{name = <<"author">>, type = varchar},
        #mekao_column{name = <<"created">>, type = datetime, ro = true}
    ]
}).

-define(S, #mekao_settings{
    placeholder = fun (_, Pos, _) -> [$$ | integer_to_list(Pos)] end,
    is_null     = fun(V) -> V == undefined end
}).
```

Pay attention, when you describe a `#mekao_table{}` each field should be at
the same position as fields in corresponding record (in our case `#book{}`).
Second macros, `S`, does two things:
  * says how placeholders in our queries are looks like. Remember this
    `... WHERE id = ?"` question mark? this is a placeholder, which
    will be substituted by the underlying driver, or database to particular
    value. Not every db have this functionallity and moreover, they differs
    (in some cases this is questions symbols, in others "$1, $2, $3...");
  * it is application specific logic how nulls are looks like: `null`,
    `undefined` or something different.

Next thing you definitely notice is `'$skip'` atom. When you construct record
like this, every other field will have `'$skip'` as a value:
```erlang
1> #book{id = 123, _ = '$skip'}.
#book{id = 123,isbn = '$skip',title = '$skip',
      author = '$skip',created = '$skip'}
```
This instructs *mekao* that you don't want to include other fields in query
constructions.

Also you may wonder about `iolist_to_binary(MekaoQ#mekao_query.body)` part.
All queries which *mekao* produce is `iolist`. This means there could be mixed
strings, binaries, chars, nested lists of strings and so on. Some drivers
do accept `iolists`, others do not. It is up to application transform this to
any acceptable form.

For more examples please see [test/mekao_tests.erl](test/mekao_tests.erl).
