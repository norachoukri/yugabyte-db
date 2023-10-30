---
title: PL/pgSQL (a.k.a. "language plpgsql") subprograms [YSQL]
headerTitle: PL/pgSQL (a.k.a. "language plpgsql") subprograms
linkTitle: >
  "language plpgsql" subprograms
description: Describes PL/pgSQL functions and procedures. These are also known as "language plpgsql" subprograms.) [YSQL].
image: /images/section_icons/api/subsection.png
menu:
  preview:
    identifier: language-plpgsql-subprograms
    parent: user-defined-subprograms-and-anon-blocks
    weight: 30
type: indexpage
showRightNav: true
---

PL/pgSQL is a conventional, block-structured, imperative programming language designed to execute in the PostgreSQL server, and by extension in the YSQL server, for the specific purpose of executing SQL statements and dealing with the outcomes that they produce. It executes in the same process as SQL itself. And it uses the same underlying implementation primitives. This has these hugely valuable consequences:

- The identical set of data types, with identical semantics, is available in both top-level SQL and in PL/pgSQL.
- Expression syntax and semantics are identical in both top-level SQL and in PL/pgSQL.
- All of the SQL built-in functions are available, with the same semantics, in PL/pgSQL.

PL/pgSQL's basic syntax conventions and repertoire of simple and compound statements seem to be inspired by Ada. Here are some examples:

- _a := b + c;_
- _return a + d_;
- _declare... begin... exception... end;_
- _if... then... elsif... then... else... end if;_
- _case... when... then... else... end case;_

However, PL/pgSQL lacks very many of Ada's features. Here are a couple of notable missing Ada features:

- packages
- the ability to define functions and procedures within _declare_ sections

On the other hand, PL/pgSQL extends Ada with a wealth of language features that target its specific use for implementing user-defined subprograms that are stored in, and that execute within, a RDBMS. Here are some examples:

```output
if some_boolean then
  insert into s.t(v) values(some_local_variable);
end if;
```

and:

```output
foreach val in array values_array loop
  insert into s.t(v) values(val) returning k into new_k;
  new_ks_array := new_ks_array||new_k;
end loop;
```

You choose PL/pgSQL as the implementation language for a user-defined subprogram by including _language plpgsql_ in the subprogram's header. Its Ada-like features make _language plpgsql_ subprograms very much more expressive and generally useful than _language sql_ subprograms. See the [example that shows how to insert a master row together with its details rows](../language-sql-subprograms/#insert-master-and-details) in the section [SQL (a.k.a. "language sql") subprograms](../language-sql-subprograms/). A _language sql_ procedure cannot meet the requirement because it has no mechanism that allows the autogenerated new _masters_ row's primary key value to be used as the new _details_ rows' foreign key value. The _language plpgsql_ procedure manages the task trivially because you can populate a local variable when you _insert_ into the _masters_ table thus:

```output
insert into s.masters(mv) values(new_mv) returning mk into new_mk;
```

And then you can reference the local variable in the next _insert_ statement, into the _details_ table, thus:

```output
insert into s.details(mk, dv)
select new_mk, u.v
from unnest(dvs) as u(v);
```

Here's another example procedure that shows various ways to return the result set from a _select_ statement—either directly or by using a loop that allows you to intervene with arbitrary processing. Notice too the difference between so-called _static SQL_ where you write the statement as direct embedded constructs in PL/pgSQL that you fix when at subprogram creation time or as a _text_ value that you can assemble and submit at run time.

```plpgsql
create schema s;
create table s.t(k serial primary key, v int not null);
insert into s.t(v) select g.v from generate_series(11, 100, 11) as g(v);

create function s.f(v_min in int, mode in text = 'static qry')
  returns table(val int)
  set search_path = pg_catalog, pg_temp
  security definer
  language plpgsql
as $body$
begin
  case mode
    when 'static qry' then
      return query select v from s.t where v > v_min order by v;

    when 'static loop' then
      declare
        x s.t.v%type not null := 0;
      begin
        for x in (select v from s.t where v > v_min order by v) loop
          val := x + 1;
          return next;
        end loop;
      end;

    when 'dynamic qry' then
      return query execute format('select v from s.%I where v > $1 order by v', 't') using v_min;

    when 'dynamic loop' then
      declare
        x s.t.v%type not null := 0;
      begin
        for x in execute format('select v from s.%I where v > $1 order by v', 't') using v_min loop
          val :=
            case
              when x < 85 then x + 3
              else             x + 7
            end;
          return next;
        end loop;
     end;
  end case;
end;
$body$;
```

Test the _static_ and the _dynamic_ _qry_ variants first. Take advantage of the default value for the _mode_ formal argument just for the demonstration effect:

```plpgsql
select s.f(v_min=>40);
select s.f(v_min=>40, mode=>'dynamic qry');
```

Each produces the same result, thus:

```output
 44
 55
 66
 77
 88
 99
```

Next test the _static_ _loop_ variant:

```plpgsql
select s.f(v_min=>40, mode=>'static loop');
```

It produces this result:

```output
  45
  56
  67
  78
  89
 100
```

Finally, test the _dynamic_ _loop_ variant:

```plpgsql
select s.f(v_min=>40, mode=>'dynamic loop');
```

It produces this result:

```output
  47
  58
  69
  80
  95
 106
```

{{< tip title="Don't use PL/pgSQL to do procedurally what SQL can do declaratively." >}}
The purpose of the code shown above is to illustrate the syntax and semantics of some useful PL/pgSQL constructs. However, you should not use procedural code, in a loop, to achieve what SQL can achieve declaratively. The effect of the _dynamic loop_ variant is better expressed thus:

```plpgsql
create function s.f2(v_min in int)
  returns table(val int)
  set search_path = pg_catalog, pg_temp
  security definer
  language plpgsql
as $body$
begin
  return query execute format('
      select
        case
          when v < 85 then v + 3
          else             v + 7
        end
      from s.%I
      where v > $1
      order by v',
   't') using v_min;
end;
$body$;
```

This:

```plpgsql
select s.f2(v_min=>40);
```

produces the same result as does this:

```plpgsql
select s.f(v_min=>40, mode=>'dynamic loop');
```
{{< /tip >}}