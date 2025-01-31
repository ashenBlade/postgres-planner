# JoinDomain

`JoinDomain` - это множество отношений, которые соединяются друг с другом INNER JOIN.

В комментарии дается другое определение - это область, в которой мы можем делать выводы с помощью EquivalenceClass'ов.

```c
typedef struct JoinDomain
{
	pg_node_attr(no_copy_equal, no_read, no_query_jumble)

	NodeTag		type;

	Relids		jd_relids;		/* all relids contained within the domain */
} JoinDomain;
```

Также как и EquivalenceClass создается в `deconstruct_jointree`.

Для всего запроса создается один верхнеуровневый JoinDomain (для всего запроса) - содержит все отношения из запроса.
Изначально он пуст, но заполняется последовательно, по мере обхода дерева JOIN'а функцией `deconstruct_recurse`.
На вход ей передается родительский JoinDomain и изначально он равен top_domain, т.е. верхнеуровневому.

При посещении очередного узла мы добавляем это отношение в родительский JoinDomain. Но все меняют не-INNER JOIN'ы - LEFT/RIGHT/ANTI/FULL OUTER. Такие в общем случае называют OUTER JOIN'ы, сокращенно OJ.

Рассмотрим как обрабатываются JOIN'ы в контексте JoinDomain'ов. Для начала, у каждого JOIN'а есть левая и правая части - как в самом запросе.
Почему мы их различаем станет понятно, если заметить, что не INNER JOIN может превратить одну часть в NULL. Как мы помним, при соединении таблиц мы создаем новый EquivalenceClass (из столбцов в выражении). Но вот если используется OUTER JOIN, то из этого выражения мы не можем заключить, что в результирующей выборке эти столбцы могут быть NULL, а сравнение с NULL всегда ложно.

Самый простой случай - INNER JOIN. В этом случае, обе части запроса используют один и тот же JD и мы всегда можем заключить, что значения столбцов после JOIN'а будут равны друг другу. То есть, имея родительский JD - `parent_jd` мы получим следующую картину (в узлах JoinTreeItem'ы).

```text
          INNER JOIN  <---- parent_jd
         /          \
        /            \
       /              \
      /                \
     /                  \
OTHER <- parent_jd       OTHER <- parent_jd
```

Другой случай - не-INNER JOIN. Тогда мы не можем обеим частям так просто назначить один и тот же JD. Поэтому в Postgres сделали такой ход - мы создаем отдельные JoinDomain'ы для каждой части, которые могут стать NULL. Но при этом сам JoinDomain узла не-INNER JOIN уже не может принадлежать parent_domain, т.к. в противном случае мы можем заключить, что все столбцы в условии не-INNER JOIN будут равны столбцам из левого JOIN DOMAIN.

Для LEFT JOIN картина будет следующей:

```text
          LEFT  JOIN  <---- child_jd
         /          \
        /            \
       /              \
      /                \
     /                  \
OTHER <- parent_jd       OTHER <- child_jd
```

Я задался вопросом почему самому JOIN'у задается child_jd, а не parent_jd? В коде я заменил child_jd на parent_jd и к моему удивлению все тесты прошли. Но затем мне стало понятно почему: мы не рассмотрели еще одну деталь реализации - поле nulling_relids у RelOptInfo. В нем хранятся relid тех OJ, которые могут превратить все столбцы этого отношения в NULL.

Рассмотрим запрос `SELECT * FROM a JOIN b USING (x) JOIN c USING (x) LEFT JOIN d USING (x)`.

```text
postgres=# explain select * from a join b on a.x = b.x left join c on a.x = c.x where a.x = 10;
                                       QUERY PLAN                                        
-----------------------------------------------------------------------------------------
 Nested Loop Left Join  (cost=8.51..101.42 rows=2197 width=12)
   ->  Nested Loop  (cost=8.51..32.05 rows=169 width=8)
         ->  Bitmap Heap Scan on a  (cost=4.26..14.95 rows=13 width=4)
               Recheck Cond: (x = 10)
               ->  Bitmap Index Scan on a_x_idx  (cost=0.00..4.25 rows=13 width=0)
                     Index Cond: (x = 10)
         ->  Materialize  (cost=4.26..15.02 rows=13 width=4)
               ->  Bitmap Heap Scan on b  (cost=4.26..14.95 rows=13 width=4)
                     Recheck Cond: (x = 10)
                     ->  Bitmap Index Scan on b_x_idx  (cost=0.00..4.25 rows=13 width=0)
                           Index Cond: (x = 10)
   ->  Materialize  (cost=0.00..41.94 rows=13 width=4)
         ->  Seq Scan on c  (cost=0.00..41.88 rows=13 width=4)
               Filter: (x = 10)
(14 rows)
```
