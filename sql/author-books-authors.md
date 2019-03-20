# Author-Books-Authors

## Description

There are two tables: `authors` and `books`, bound as many-to-many via intermediate table `ab`.

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

<details><summary>Table for mtm bond: (hidden, obvious)</summary>

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

## Dataset

### Authors

Authors are `string.ascii_uppercase` letters from Python stdlib.

<details><summary>Example</summary>

```sql
select * from authors order by random() limit 5 ;
```

| author |
|--------|
| Z |
| X |
| V |
| P |
| L |

</details>
