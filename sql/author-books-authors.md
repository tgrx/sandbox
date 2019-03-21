# Author-Books-Authors

## Spoilers!

| Approach                                                                |    Costs 💰 |    Timing ⌛️ |
|-------------------------------------------------------------------------|------------:|-------------:|
| ["HAVING"](#approach-1-having)                                          | 7380 ~ 8376 | 256 _±51_ ms |
| ["GROUP BY" subquery + "WHERE"](#approach-2-group-by-subquery--where)   | 7380 ~ 8376 | 284 _±50_ ms |
| ["GROUP BY" CTE + "WHERE"](#approach-3-group-by-cte--where)             | 8116 ~ 8519 | 290 _±52_ ms |
| [Subquery + "LATERAL" Subquery](#approach-4-subquery--lateral-subquery) | 3337 ~ 3534 |  75 _±15_ ms |
| [CTE + CTE](#approach-5-cte--cte)                                       | 3038 ~ 3042 |  53 _±10_ ms |
| [CTE + CTE on mtm table](#approach-6-cte--cte-on-mtm-table)             | 2456 ~ 2460 |  50 _±19_ ms |
| [M2M JOIN LATERAL M2M](#approach-7-m2m-join-lateral-m2m)                | 2679 ~ 2784 |  38 _±11_ ms |
| [M2M JOIN M2M](#approach-8-m2m-join-m2m)                                | 2679 ~ 2784 |   38 _±8_ ms |

## Description

There are two tables: `authors` and `books`, bound as many-to-many via intermediate table `ab`.
Given the author, display its books and for each book display a set of its authors.

### Example

Author A has created books {A, AB, ABC}.

Author B has created books {B, BC, ABC, BCD}.

Author C has created books {C, BC, ABC, BCD}.

Author D has created books {D, BCD}.

For the given author name, i.e. "C", the query MUST return a table:

| book | authors |
|------|---------|
| C    | C       |
| BC   | {B,C}   |
| ABC  | {A,B,C} |
| BCD  | {B,C,D} |

## Dataset

### Tables

Table `authors`:

```sql
create table if not exists authors
(
  author text primary key
);
```

Table `books`:

```sql
create table if not exists books
(
  book text primary key
);
```

<details><summary>Table for mtm bond: (hidden because obvious)</summary>

```sql
create table if not exists ab
(
  book   text not null
         references books(book)
             on update cascade on delete cascade,
  author text not null
         references authors(author)
             on update cascade on delete cascade
);

create index ab_author_idx on ab(author);
create index ab_book_idx on ab(book);
```
</details>

### Data

#### Amounts

| Table   | Rows  |
|:--------|------:|
| authors |    26 |
| books   |  ~18k |
| ab      |  ~68k |
| CROSS   |  ~33b |

#### Authors

Authors are `string.ascii_uppercase` letters from Python stdlib.

<details><summary>Example</summary>

```sql
select * from authors order by random() limit 4;
```

| author |
|--------|
| A |
| S |
| M |
| R |

</details>

#### Books

Books are the union of all combinations of authors of size 1..4.

```python
from itertools import combinations
from string import ascii_uppercase

books = sorted(
    "".join(c)
    for n in range(5)
    for c in combinations(ascii_uppercase, n)
)
```

<details><summary>Example</summary>

```sql
select * from books order by random() limit 4;
```

| book |
|------|
| SAM  |
| A    |
| ASMR |
| RS   |

</details>

#### Many-to-many bond

Supposed to be obvious.

<details><summary>If not...</summary>

Each book contains names of its authors by definition, so mtm table is formed as:

```sql
with books_authors(author, book) as (
    select
        author,
        book
    from
        books,
        unnest(regexp_split_to_array(book, '')) as author
    )
insert
  into
    ab(author, book)
select
    author,
    book
  from
    books_authors
;
```

```sql
select * from ab where book = 'ASMR';
```

| book | author |
|------|--------|
| ASMR | A      |
| ASMR | S      |
| ASMR | M      |
| ASMR | R      |

</details>

## Solutions

### Approach NULL: naïve

Naïve query with `WHERE author = 'M'`. Does not work.

<details><summary>SQL</summary>

```sql
select
  b.book,
  array_agg(a.author order by a.author) as authors
  from
    authors a,
    books   b,
            ab
  where
    a.author = ab.author and
    ab.book = b.book and
    a.author = 'M'
  group by
    b.book
;
```

</details>

<details><summary>Response sample</summary>

| book | authors |
|------|---------|
| ALM  | {M}     |
| ALMB | {M}     |
| ...  | ...     |

</details>

### Approach 1: "HAVING"

Yep, just filtering of grouped data. 

<details><summary>SQL</summary>

```sql
select
  b.book,
  array_agg(a.author order by a.author) as authors
  from
    authors a,
    books   b,
            ab
  where
    a.author = ab.author and
    ab.book = b.book
  group by
    b.book
  having
    '{"M"}' <@ array_agg(a.author)
```

</details>

<details><summary>Query plan</summary>

```text
GroupAggregate  (cost=7380.76..8376.78 rows=90 width=36)
  Group Key: b.book
  Filter: ('{M}'::text[] <@ array_agg(a.author))
  ->  Sort  (cost=7380.76..7551.45 rows=68276 width=6)
        Sort Key: b.book
        ->  Hash Join  (cost=553.97..1898.50 rows=68276 width=6)
              Hash Cond: (ab.book = b.book)
              ->  Hash Join  (cost=71.20..1236.46 rows=68276 width=6)
                    Hash Cond: (ab.author = a.author)
                    ->  Seq Scan on ab  (cost=0.00..985.76 rows=68276 width=6)
                    ->  Hash  (cost=37.20..37.20 rows=2720 width=2)
                          ->  Seq Scan on authors a  (cost=0.00..37.20 rows=2720 width=2)
              ->  Hash  (cost=259.01..259.01 rows=17901 width=4)
                    ->  Seq Scan on books b  (cost=0.00..259.01 rows=17901 width=4)
```

</details>

<details><summary>Response sample</summary>

| book | authors   |
|------|-----------|
| ALM  | {A,L,M}   |
| ALMB | {A,L,M,B} |
| ...  | ...       |

</details>

💰 Cost: 7380 ~ 8376

⌛️ Timing: 256 _±51_ ms

✅ Pros: selecting ordered data (eliminated by GroupAggregate)

❌ Cons: aggregate filter in `having`

### Approach 2: "GROUP BY" subquery + "WHERE"

Almost the same as [Approach 1](#approach-1-having).
Grouped data is closed under the subquery, and main query performs WHERE filtering.

What one may expect: there is no need to produce the aggregate twice, for both `SELECT` and `HAVING`.

What the reality hits with: try to observe and compare query plans. PostgreSQL is smart indeed.

<details><summary>SQL</summary>

```sql
select
  book,
  authors
  from
    (
      select
        b.book                                as book,
        array_agg(a.author order by a.author) as authors
        from
          authors a,
          books   b,
                  ab
        where
          a.author = ab.author and
          ab.book = b.book
        group by
          b.book
      ) sq
  where
    '{"M"}' <@ authors
```

</details>

<details><summary>Query plan</summary>

```text
GroupAggregate  (cost=7380.76..8376.78 rows=90 width=36)
  Group Key: b.book
  Filter: ('{M}'::text[] <@ array_agg(a.author ORDER BY a.author))
  ->  Sort  (cost=7380.76..7551.45 rows=68276 width=6)
        Sort Key: b.book
        ->  Hash Join  (cost=553.97..1898.50 rows=68276 width=6)
              Hash Cond: (ab.book = b.book)
              ->  Hash Join  (cost=71.20..1236.46 rows=68276 width=6)
                    Hash Cond: (ab.author = a.author)
                    ->  Seq Scan on ab  (cost=0.00..985.76 rows=68276 width=6)
                    ->  Hash  (cost=37.20..37.20 rows=2720 width=2)
                          ->  Seq Scan on authors a  (cost=0.00..37.20 rows=2720 width=2)
              ->  Hash  (cost=259.01..259.01 rows=17901 width=4)
                    ->  Seq Scan on books b  (cost=0.00..259.01 rows=17901 width=4)
```

</details>

<details><summary>Response sample</summary>

| book | authors   |
|------|-----------|
| ALM  | {A,L,M}   |
| ALMB | {A,L,M,B} |
| ...  | ...       |

</details>

💰 Cost: 7380 ~ 8376

⌛ Timing: 284 _±50_ ms

✅ Pros: `¯\_(ツ)_/¯`

❌ Cons: wasted time.

### Approach 3: "GROUP BY" CTE + "WHERE"

Here we are wrapping subquery into CTE.
Expecting PostgreSQL to build a table-like object and do not perform filtering by aggregate.

<details><summary>SQL</summary>

```sql
with cte(book, authors) as (
  select
   b.book                                as book,
   array_agg(a.author order by a.author) as authors
   from
     authors a,
     books   b,
             ab
   where
     a.author = ab.author and
     ab.book = b.book
   group by
     b.book
  )
select
  book,
  authors
  from
    cte
  where
    '{"M"}' <@ authors
```

</details>

<details><summary>Query plan</summary>

```text
CTE Scan on cte  (cost=8116.59..8519.36 rows=90 width=64)
  Filter: ('{M}'::text[] <@ authors)
  CTE cte
    ->  GroupAggregate  (cost=7380.76..8116.59 rows=17901 width=36)
          Group Key: b.book
          ->  Sort  (cost=7380.76..7551.45 rows=68276 width=6)
                Sort Key: b.book
                ->  Hash Join  (cost=553.97..1898.50 rows=68276 width=6)
                      Hash Cond: (ab.book = b.book)
                      ->  Hash Join  (cost=71.20..1236.46 rows=68276 width=6)
                            Hash Cond: (ab.author = a.author)
                            ->  Seq Scan on ab  (cost=0.00..985.76 rows=68276 width=6)
                            ->  Hash  (cost=37.20..37.20 rows=2720 width=2)
                                  ->  Seq Scan on authors a  (cost=0.00..37.20 rows=2720 width=2)
                      ->  Hash  (cost=259.01..259.01 rows=17901 width=4)
                            ->  Seq Scan on books b  (cost=0.00..259.01 rows=17901 width=4)
```

</details>

<details><summary>Response sample</summary>

| book | authors   |
|------|-----------|
| ALM  | {A,L,M}   |
| ALMB | {A,L,M,B} |
| ...  | ...       |

</details>

💰 Cost: 8116 ~ 8519

⌛ Timing: 290 _±52_ ms

✅ Pros: flexibility that CTE provides.

❌ Cons: slight performance impact due to CTE node.

### Approach 4: Subquery + "LATERAL" Subquery

In few words, LATERAL allows one subquery to be referenced from another.
Please, [RTFM](https://www.postgresql.org/docs/current/queries-table-expressions.html#id-1.5.6.6.5.10.2).

<details><summary>SQL</summary>

```sql
select
  author_books.book,
  array_agg(book_authors.author) as authors
  from
    (
      select
        b.book as book
        from
          authors    as a
          join ab
               using (author)
          join books as b
               using (book)
        where
          a.author = 'M'
      )                                    as author_books
    join lateral
    (
      select
        a.author as author
        from
          authors    as a
          join ab
               using (author)
          join books as b
               using (book)
        where
          b.book = author_books.book
      )                                    as book_authors
    on true
  group by
    author_books.book
;
```

</details>

<details><summary>Query plan</summary>

```text
GroupAggregate  (cost=3337.68..3534.56 rows=9844 width=36)
  Group Key: b.book
  ->  Sort  (cost=3337.68..3362.29 rows=9844 width=6)
        Sort Key: b.book
        ->  Hash Join  (cost=950.81..2684.78 rows=9844 width=6)
              Hash Cond: (ab_1.author = a_1.author)
              ->  Nested Loop  (cost=879.61..2587.69 rows=9844 width=6)
                    ->  Nested Loop  (cost=879.32..1265.09 rows=2581 width=12)
                          ->  Index Only Scan using authors_pkey on authors a  (cost=0.15..8.17 rows=1 width=2)
                                Index Cond: (author = 'M'::text)
                          ->  Hash Join  (cost=879.16..1231.11 rows=2581 width=14)
                                Hash Cond: (b.book = b_1.book)
                                ->  Seq Scan on books b  (cost=0.00..259.01 rows=17901 width=4)
                                ->  Hash  (cost=846.90..846.90 rows=2581 width=10)
                                      ->  Hash Join  (cost=427.82..846.90 rows=2581 width=10)
                                            Hash Cond: (b_1.book = ab.book)
                                            ->  Seq Scan on books b_1  (cost=0.00..259.01 rows=17901 width=4)
                                            ->  Hash  (cost=395.56..395.56 rows=2581 width=6)
                                                  ->  Bitmap Heap Scan on ab  (cost=60.30..395.56 rows=2581 width=6)
                                                        Recheck Cond: (author = 'M'::text)
                                                        ->  Bitmap Index Scan on ab_author_idx  (cost=0.00..59.65 rows=2581 width=0)
                                                              Index Cond: (author = 'M'::text)
                    ->  Index Scan using ab_book_idx on ab ab_1  (cost=0.29..0.47 rows=4 width=6)
                          Index Cond: (book = b_1.book)
              ->  Hash  (cost=37.20..37.20 rows=2720 width=2)
                    ->  Seq Scan on authors a_1  (cost=0.00..37.20 rows=2720 width=2)
```

</details>

<details><summary>Response sample</summary>

| book | authors   |
|------|-----------|
| ALM  | {A,L,M}   |
| ALMB | {A,L,M,B} |
| ...  | ...       |

</details>

💰 Cost: 3337 ~ 3534

⌛ Timing: 75 _±15_ ms

✅ Pros: Fast and furious!

❌ Cons: low flexibility due to many subqueries, and `LATERAL`, of course.

### Approach 5: CTE + CTE

In this approach we build two CTEs.
The former produces books of given author.
The latter produces authors for books from the former.

This approach may be implemented in two ways:

1. select plain data in CTE 2, apply grouping in the top query;
1. apply grouping in CTE 2, select `*` in the top query;

Both ways are equivalent.

<details><summary>SQL</summary>

```sql
with author_books(book)          as (
                                      select
                                        b.book as book
                                        from
                                          authors    as a
                                          join ab
                                               using (author)
                                          join books as b
                                               using (book)
                                        where
                                          a.author = 'M'
                                      ),
     book_authors(book, authors) as (
                                      select
                                        b.book              as book,
                                        array_agg(a.author) as authors
                                        from
                                          authors           as a
                                          join ab
                                               using (author)
                                          join author_books as b
                                               using (book)
                                        group by
                                          b.book
                                      )
select
  book,
  authors
  from
    book_authors
;
```

</details>

<details><summary>Query plan</summary>

```text
CTE Scan on book_authors  (cost=3038.86..3042.86 rows=200 width=64)
  CTE author_books
    ->  Nested Loop  (cost=427.98..880.88 rows=2581 width=4)
          ->  Index Only Scan using authors_pkey on authors a  (cost=0.15..8.17 rows=1 width=2)
                Index Cond: (author = 'M'::text)
          ->  Hash Join  (cost=427.82..846.90 rows=2581 width=6)
                Hash Cond: (b.book = ab.book)
                ->  Seq Scan on books b  (cost=0.00..259.01 rows=17901 width=4)
                ->  Hash  (cost=395.56..395.56 rows=2581 width=6)
                      ->  Bitmap Heap Scan on ab  (cost=60.30..395.56 rows=2581 width=6)
                            Recheck Cond: (author = 'M'::text)
                            ->  Bitmap Index Scan on ab_author_idx  (cost=0.00..59.65 rows=2581 width=0)
                                  Index Cond: (author = 'M'::text)
  CTE book_authors
    ->  HashAggregate  (cost=2155.48..2157.98 rows=200 width=64)
          Group Key: b_1.book
          ->  Hash Join  (cost=1910.41..2106.10 rows=9875 width=34)
                Hash Cond: (ab_1.author = a_1.author)
                ->  Hash Join  (cost=1839.21..2008.94 rows=9875 width=34)
                      Hash Cond: (b_1.book = ab_1.book)
                      ->  CTE Scan on author_books b_1  (cost=0.00..51.62 rows=2581 width=32)
                      ->  Hash  (cost=985.76..985.76 rows=68276 width=6)
                            ->  Seq Scan on ab ab_1  (cost=0.00..985.76 rows=68276 width=6)
                ->  Hash  (cost=37.20..37.20 rows=2720 width=2)
                      ->  Seq Scan on authors a_1  (cost=0.00..37.20 rows=2720 width=2)
```

</details>

<details><summary>Response sample</summary>

| book | authors   |
|------|-----------|
| ALM  | {A,L,M}   |
| ALMB | {A,L,M,B} |
| ...  | ...       |

</details>

💰 Cost: 3038 ~ 3042

⌛ Timing: 53 _±10_ ms

✅ Pros: Fast and flexible!

❌ Cons: multiple CTEs

### Approach 6: CTE + CTE on mtm table

In this example (and maybe in your project) the whole information is contained in many-to-many table.
Just remove redundant JOINs from [Approach 5](#approach-5-cte-cte).

<details><summary>SQL</summary>

```sql
with author_books(book)          as (
                                      select
                                        ab.book as book
                                        from
                                          ab
                                        where
                                          ab.author = 'M'
                                      ),
     book_authors(book, authors) as (
                                      select
                                        b.book               as book,
                                        array_agg(ab.author) as authors
                                        from
                                          ab
                                          join author_books as b
                                               using (book)
                                        group by
                                          b.book
                                      )
select
  book,
  authors
  from
    book_authors
;
```

</details>

<details><summary>Query plan</summary>

```text
CTE Scan on book_authors  (cost=2456.37..2460.37 rows=200 width=64)
  CTE author_books
    ->  Bitmap Heap Scan on ab  (cost=60.30..395.56 rows=2581 width=4)
          Recheck Cond: (author = 'M'::text)
          ->  Bitmap Index Scan on ab_author_idx  (cost=0.00..59.65 rows=2581 width=0)
                Index Cond: (author = 'M'::text)
  CTE book_authors
    ->  HashAggregate  (cost=2058.31..2060.81 rows=200 width=64)
          Group Key: b.book
          ->  Hash Join  (cost=1839.21..2008.94 rows=9875 width=34)
                Hash Cond: (b.book = ab2.book)
                ->  CTE Scan on author_books b  (cost=0.00..51.62 rows=2581 width=32)
                ->  Hash  (cost=985.76..985.76 rows=68276 width=6)
                      ->  Seq Scan on ab ab2  (cost=0.00..985.76 rows=68276 width=6)
```

</details>

<details><summary>Response sample</summary>

| book | authors   |
|------|-----------|
| ALM  | {A,L,M}   |
| ALMB | {A,L,M,B} |
| ...  | ...       |

</details>

💰 Cost: 2456 ~ 2460

⌛ Timing: 50 _±19_ ms

✅ Pros: Fast

❌ Cons: inflexible for generic query generators

### Approach 7: M2M JOIN LATERAL M2M

Similar to the [Approach 4](#approach-4-subquery-lateral-subquery), with redundant JOINs removed.

<details><summary>SQL</summary>

```sql
select
  author_books.book,
  array_agg(book_authors.author) as authors
  from
    (
      select
        ab1.book as book
        from
          ab as ab1
        where
          ab1.author = 'M'
      )   as author_books
    join
      lateral (
        select
          ab2.author as author
          from
            ab as ab2
          where
            ab2.book = author_books.book
        ) as book_authors
      on true
  group by
    author_books.book
```

</details>

<details><summary>Query plan</summary>

```text
GroupAggregate  (cost=2679.59..2784.22 rows=2446 width=36)
  Group Key: ab1.book
  ->  Sort  (cost=2679.59..2704.27 rows=9875 width=6)
        Sort Key: ab1.book
        ->  Hash Join  (cost=427.82..2024.40 rows=9875 width=6)
              Hash Cond: (ab2.book = ab1.book)
              ->  Seq Scan on ab ab2  (cost=0.00..985.76 rows=68276 width=6)
              ->  Hash  (cost=395.56..395.56 rows=2581 width=4)
                    ->  Bitmap Heap Scan on ab ab1  (cost=60.30..395.56 rows=2581 width=4)
                          Recheck Cond: (author = 'M'::text)
                          ->  Bitmap Index Scan on ab_author_idx  (cost=0.00..59.65 rows=2581 width=0)
                                Index Cond: (author = 'M'::text)
```

</details>

<details><summary>Response sample</summary>

| book | authors   |
|------|-----------|
| ALM  | {A,L,M}   |
| ALMB | {A,L,M,B} |
| ...  | ...       |

</details>

💰 Cost: 2679 ~ 2784

⌛ Timing: 38 _±11_ ms

✅ Pros: Fast

❌ Cons: inflexible for generic query generators

### Approach 8: M2M JOIN M2M

Simply join M2M with itself and gather necessary information.

<details><summary>SQL</summary>

```sql
select
  author_books.book,
  array_agg(book_authors.author) as authors
  from
    ab      as author_books
    join ab as book_authors
         on author_books.author = 'M' and author_books.book = book_authors.book
  group by
    author_books.book
```

</details>

<details><summary>Query plan</summary>

```text
GroupAggregate  (cost=2679.59..2784.22 rows=2446 width=36)
  Group Key: author_books.book
  ->  Sort  (cost=2679.59..2704.27 rows=9875 width=6)
        Sort Key: author_books.book
        ->  Hash Join  (cost=427.82..2024.40 rows=9875 width=6)
              Hash Cond: (book_authors.book = author_books.book)
              ->  Seq Scan on ab book_authors  (cost=0.00..985.76 rows=68276 width=6)
              ->  Hash  (cost=395.56..395.56 rows=2581 width=4)
                    ->  Bitmap Heap Scan on ab author_books  (cost=60.30..395.56 rows=2581 width=4)
                          Recheck Cond: (author = 'M'::text)
                          ->  Bitmap Index Scan on ab_author_idx  (cost=0.00..59.65 rows=2581 width=0)
                                Index Cond: (author = 'M'::text)
```

</details>

<details><summary>Response sample</summary>

| book | authors   |
|------|-----------|
| ALM  | {A,L,M}   |
| ALMB | {A,L,M,B} |
| ...  | ...       |

</details>

💰 Cost: 2679 ~ 2784

⌛ Timing: 38 _±8_ ms

✅ Pros: Fast

❌ Cons: inflexible for generic query generators
