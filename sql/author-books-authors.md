# Author-Books-Authors

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

### Approach 1: HAVING

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
GroupAggregate  (cost=7340.08..8336.11 rows=90 width=36) (actual time=112.514..227.972 rows=2626 loops=1)
  Group Key: b.book
  Filter: ('{M}'::text[] <@ array_agg(a.author))
  Rows Removed by Filter: 15275
  ->  Sort  (cost=7340.08..7510.77 rows=68276 width=6) (actual time=110.746..132.872 rows=68276 loops=1)
        Sort Key: b.book
        Sort Method: external merge  Disk: 1136kB
        ->  Hash Join  (cost=484.36..1857.83 rows=68276 width=6) (actual time=7.462..65.169 rows=68276 loops=1)
              Hash Cond: (ab.book = b.book)
              ->  Hash Join  (cost=1.58..1195.78 rows=68276 width=6) (actual time=0.068..35.293 rows=68276 loops=1)
                    Hash Cond: (ab.author = a.author)
                    ->  Seq Scan on ab  (cost=0.00..985.76 rows=68276 width=6) (actual time=0.027..12.892 rows=68276 loops=1)
                    ->  Hash  (cost=1.26..1.26 rows=26 width=2) (actual time=0.020..0.020 rows=26 loops=1)
                          Buckets: 1024  Batches: 1  Memory Usage: 9kB
                          ->  Seq Scan on authors a  (cost=0.00..1.26 rows=26 width=2) (actual time=0.006..0.011 rows=26 loops=1)
              ->  Hash  (cost=259.01..259.01 rows=17901 width=4) (actual time=6.769..6.769 rows=17901 loops=1)
                    Buckets: 32768  Batches: 1  Memory Usage: 900kB
                    ->  Seq Scan on books b  (cost=0.00..259.01 rows=17901 width=4) (actual time=0.024..2.346 rows=17901 loops=1)
Planning Time: 0.797 ms
Execution Time: 230.749 ms
```

</details>

<details><summary>Response sample</summary>

| book | authors   |
|------|-----------|
| ALM  | {A,L,M}   |
| ALMB | {A,L,M,B} |
| ...  | ...       |

</details>

Cost: 7340~8336

Timing: 213~285

Pros: selecting ordered data

Cons: aggregate filter in `having`

### Approach 2: GROUP BY subquery + WHERE

Almost the same as [Approach 1](#markdown-header-approach-1-having). Grouped data is closed under the subquery, and main query performs WHERE filtering, hoping that there is no need to produce the aggregate twice.

Kek. PostgreSQL is smart indeed: try to observe and compare query plans.

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
GroupAggregate  (cost=7340.08..8336.11 rows=90 width=36) (actual time=109.898..226.390 rows=2626 loops=1)
  Group Key: b.book
  Filter: ('{M}'::text[] <@ array_agg(a.author ORDER BY a.author))
  Rows Removed by Filter: 15275
  ->  Sort  (cost=7340.08..7510.77 rows=68276 width=6) (actual time=108.249..132.448 rows=68276 loops=1)
        Sort Key: b.book
        Sort Method: external merge  Disk: 1136kB
        ->  Hash Join  (cost=484.36..1857.83 rows=68276 width=6) (actual time=13.597..71.021 rows=68276 loops=1)
              Hash Cond: (ab.book = b.book)
              ->  Hash Join  (cost=1.58..1195.78 rows=68276 width=6) (actual time=0.060..34.309 rows=68276 loops=1)
                    Hash Cond: (ab.author = a.author)
                    ->  Seq Scan on ab  (cost=0.00..985.76 rows=68276 width=6) (actual time=0.017..10.462 rows=68276 loops=1)
                    ->  Hash  (cost=1.26..1.26 rows=26 width=2) (actual time=0.020..0.021 rows=26 loops=1)
                          Buckets: 1024  Batches: 1  Memory Usage: 9kB
                          ->  Seq Scan on authors a  (cost=0.00..1.26 rows=26 width=2) (actual time=0.007..0.011 rows=26 loops=1)
              ->  Hash  (cost=259.01..259.01 rows=17901 width=4) (actual time=13.469..13.469 rows=17901 loops=1)
                    Buckets: 32768  Batches: 1  Memory Usage: 900kB
                    ->  Seq Scan on books b  (cost=0.00..259.01 rows=17901 width=4) (actual time=0.015..5.302 rows=17901 loops=1)
Planning Time: 0.782 ms
Execution Time: 230.062 ms
```

</details>

<details><summary>Response sample</summary>

| book | authors   |
|------|-----------|
| ALM  | {A,L,M}   |
| ALMB | {A,L,M,B} |
| ...  | ...       |

</details>

Cost: 7340~8336

Timing: 215~324

Pros: `¯\_(ツ)_/¯`

Cons: drop of self-esteem.

### Approach 3: CTE

Here we are wrapping subquery into CTE, hoping our self-esteem will rise up.

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
CTE Scan on cte  (cost=8075.91..8478.68 rows=90 width=64) (actual time=98.535..222.530 rows=2626 loops=1)
  Filter: ('{M}'::text[] <@ authors)
  Rows Removed by Filter: 15275
  CTE cte
    ->  GroupAggregate  (cost=7340.08..8075.91 rows=17901 width=36) (actual time=96.565..211.477 rows=17901 loops=1)
          Group Key: b.book
          ->  Sort  (cost=7340.08..7510.77 rows=68276 width=6) (actual time=96.525..120.235 rows=68276 loops=1)
                Sort Key: b.book
                Sort Method: external merge  Disk: 1136kB
                ->  Hash Join  (cost=484.36..1857.83 rows=68276 width=6) (actual time=7.786..55.910 rows=68276 loops=1)
                      Hash Cond: (ab.book = b.book)
                      ->  Hash Join  (cost=1.58..1195.78 rows=68276 width=6) (actual time=0.121..28.798 rows=68276 loops=1)
                            Hash Cond: (ab.author = a.author)
                            ->  Seq Scan on ab  (cost=0.00..985.76 rows=68276 width=6) (actual time=0.062..8.473 rows=68276 loops=1)
                            ->  Hash  (cost=1.26..1.26 rows=26 width=2) (actual time=0.035..0.035 rows=26 loops=1)
                                  Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                  ->  Seq Scan on authors a  (cost=0.00..1.26 rows=26 width=2) (actual time=0.015..0.018 rows=26 loops=1)
                      ->  Hash  (cost=259.01..259.01 rows=17901 width=4) (actual time=7.488..7.489 rows=17901 loops=1)
                            Buckets: 32768  Batches: 1  Memory Usage: 900kB
                            ->  Seq Scan on books b  (cost=0.00..259.01 rows=17901 width=4) (actual time=0.152..2.492 rows=17901 loops=1)
Planning Time: 5.721 ms
Execution Time: 226.645 ms
```

</details>

<details><summary>Response sample</summary>

| book | authors   |
|------|-----------|
| ALM  | {A,L,M}   |
| ALMB | {A,L,M,B} |
| ...  | ...       |

</details>

Cost: 8075~8478

Timing: 222~338

Pros: `¯\_(ツ)_/¯`

Cons: drop of self-esteem, again.

### Approach 4: JOIN LATERAL

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
GroupAggregate  (cost=3301.42..3502.12 rows=10035 width=36) (actual time=70.447..74.045 rows=2626 loops=1)
  Group Key: b.book
  ->  Sort  (cost=3301.42..3326.50 rows=10035 width=6) (actual time=70.434..71.114 rows=10151 loops=1)
        Sort Key: b.book
        Sort Method: quicksort  Memory: 860kB
        ->  Hash Join  (cost=875.80..2634.45 rows=10035 width=6) (actual time=15.569..65.215 rows=10151 loops=1)
              Hash Cond: (ab_1.author = a_1.author)
              ->  Nested Loop  (cost=874.22..2602.23 rows=10035 width=6) (actual time=15.459..62.366 rows=10151 loops=1)
                    ->  Nested Loop  (cost=873.92..1254.01 rows=2631 width=12) (actual time=15.412..21.308 rows=2626 loops=1)
                          ->  Seq Scan on authors a  (cost=0.00..1.32 rows=1 width=2) (actual time=0.010..0.012 rows=1 loops=1)
                                Filter: (author = 'M'::text)
                                Rows Removed by Filter: 25
                          ->  Hash Join  (cost=873.92..1226.37 rows=2631 width=14) (actual time=15.401..20.913 rows=2626 loops=1)
                                Hash Cond: (b.book = b_1.book)
                                ->  Seq Scan on books b  (cost=0.00..259.01 rows=17901 width=4) (actual time=0.010..1.935 rows=17901 loops=1)
                                ->  Hash  (cost=841.04..841.04 rows=2631 width=10) (actual time=15.247..15.247 rows=2626 loops=1)
                                      Buckets: 4096  Batches: 1  Memory Usage: 145kB
                                      ->  Hash Join  (cost=421.46..841.04 rows=2631 width=10) (actual time=3.745..13.524 rows=2626 loops=1)
                                            Hash Cond: (b_1.book = ab.book)
                                            ->  Seq Scan on books b_1  (cost=0.00..259.01 rows=17901 width=4) (actual time=0.011..3.539 rows=17901 loops=1)
                                            ->  Hash  (cost=388.57..388.57 rows=2631 width=6) (actual time=3.313..3.313 rows=2626 loops=1)
                                                  Buckets: 4096  Batches: 1  Memory Usage: 132kB
                                                  ->  Bitmap Heap Scan on ab  (cost=52.68..388.57 rows=2631 width=6) (actual time=0.514..1.771 rows=2626 loops=1)
                                                        Recheck Cond: (author = 'M'::text)
                                                        Heap Blocks: exact=195
                                                        ->  Bitmap Index Scan on ab_author_idx  (cost=0.00..52.02 rows=2631 width=0) (actual time=0.488..0.488 rows=2626 loops=1)
                                                              Index Cond: (author = 'M'::text)
                    ->  Index Scan using ab_book_idx on ab ab_1  (cost=0.29..0.47 rows=4 width=6) (actual time=0.014..0.015 rows=4 loops=2626)
                          Index Cond: (book = b_1.book)
              ->  Hash  (cost=1.26..1.26 rows=26 width=2) (actual time=0.048..0.048 rows=26 loops=1)
                    Buckets: 1024  Batches: 1  Memory Usage: 9kB
                    ->  Seq Scan on authors a_1  (cost=0.00..1.26 rows=26 width=2) (actual time=0.011..0.015 rows=26 loops=1)
Planning Time: 2.524 ms
Execution Time: 74.299 ms
```

</details>

<details><summary>Response sample</summary>

| book | authors   |
|------|-----------|
| ALM  | {A,L,M}   |
| ALMB | {A,L,M,B} |
| ...  | ...       |

</details>

Cost: 3301~3502

Timing: 65~100

Pros: Fast and furious!

Cons: drop of self-esteem, again: one does not simply understand `LATERAL`.
