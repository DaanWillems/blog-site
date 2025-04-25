### Schema and Query Language

The schema should define what kind of data can be written to the database. It enforces that data is structured a certain way. The query language allows us to insert, read, update and delete data. We'll take heavy ‘inspiration’ from SQL to define our query language, which will also be used to define the schema. 

I want the schema to support tables, so we can have multiple structures. Tables consist of columns, where each column has a key and a type. One of the columns contains the primary key, which is used to build the primary index. Columns can have the values: string, integer and boolean. The query language should allow simple operations on the data, inserting, reading, updating and deleting. Given these requirements, let's define a grammar for the query language we'll call it: SSQL (Simple SQL).

```
<int> ::= [0-9]+

<string> ::= “[a-Z0-9_-]+"

<bool> ::= true

<bool> ::= false

<type> ::= int

<type> ::= string

<type> ::= bool

<expr> ::= <int>

<expr> ::= <string>

<expr> ::= <bool>

<expr_list> ::= <expr_list>

<expr_list> ::= <expr_list>,  <expr_list>

<expr_list> ::= <expr>

<expr_list> ::= <select_expr>

<expr_list> ::= <select_expr>,  <select_expr>

<select_expr> ::= <string>

<key_marker> ::= <null>

<key_marker> ::= primary

<column_def> ::= <string> <type> <key_marker>,

<where_expr> ::= <where_expr> && <where_expr>

<where_expr> ::= <where_expr> || <where_expr>

<where_expr> ::= <string> == <expr>

<where_expr> ::= <string> != <expr>

create table <string> {

<column_def>

};

delete table <string>;

select <select_expr> from <string> where <where_expr>;

update <string> set <expr_list> where <where_expr>;

delete from <string> where <where_expr>;

insert into <string> values <expr_list>;
```

This grammar is fully LL(1) compatible, and should therefore be pretty simple to parse. Now that is out of the way, let's move on to the actual data storing part. 


