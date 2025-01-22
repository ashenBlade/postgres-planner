# EquivalenceClass

`EquivalenceClass` - множество равных друг другу элементов.

```c
/* src/include/nodes/pathnodes.h */
typedef struct EquivalenceClass
{
	pg_node_attr(custom_read_write, no_copy_equal, no_read, no_query_jumble)

	NodeTag		type;

	List	   *ec_opfamilies;	/* btree operator family OIDs */
	Oid			ec_collation;	/* collation, if datatypes are collatable */
	List	   *ec_members;		/* list of EquivalenceMembers */
	List	   *ec_sources;		/* list of generating RestrictInfos */
	List	   *ec_derives;		/* list of derived RestrictInfos */
	Relids		ec_relids;		/* all relids appearing in ec_members, except
								 * for child members (see below) */
	bool		ec_has_const;	/* any pseudoconstants in ec_members? */
	bool		ec_has_volatile;	/* the (sole) member is a volatile expr */
	bool		ec_broken;		/* failed to generate needed clauses? */
	Index		ec_sortref;		/* originating sortclause label, or 0 */
	Index		ec_min_security;	/* minimum security_level in ec_sources */
	Index		ec_max_security;	/* maximum security_level in ec_sources */
	struct EquivalenceClass *ec_merged; /* set if merged into another EC */
} EquivalenceClass;
```

Внутри себя он хранит не простые `Node *`, а структуры `EquivalenceMember`

Новый элемент добавляется, если находим новое `mergejoinable` выражение равенства `A = B`. При необходимости мы можем смержить 2 EC в один новый. Также, могут быть и вырожденные множества - из единственного элемента.

В множестве могут быть самые разные элементы: столбцы, константы и т.д.
Например, запрос `SELECT * FROM a JOIN b ON a.x = b.x WHERE a.x = 10` создает множество `{a.x, b.x, 10}`, а этот `SELECT * FROM a JOIN b ON a.x = b.x` - `{a.x, b.x}`.

EquivalenceClass'ы вычисляются (создаются) в функции `deconstruct_jointree` (`src/backend/optimizer/plan/initsplan.c`). Поставим точку останова на 804 строке (`return`). И запустим первый запрос:

Все классы эквивалентности создаются во время разбора дерева JOIN, внутри функции `deconstruct_jointree`. Ее логика разделена на 3 фазы (даны функции-лошадки каждой фазы):

1. `deconstruct_recurse` - проход по дереву и создание списка из `JoinTreeItem`. Это обрертка над каждым элементом из этого дерева, но с дополнительной информацией. Здесь дерево выражения условий превращается в неявный список условий AND

```c
typedef struct JoinTreeItem
{
	/* Fields filled during deconstruct_recurse: */
	Node	   *jtnode;			/* jointree node to examine */
	JoinDomain *jdomain;		/* join domain for its ON/WHERE clauses */
	struct JoinTreeItem *jti_parent;	/* JoinTreeItem for this node's
										 * parent, or NULL if it's the top */
	Relids		qualscope;		/* base+OJ Relids syntactically included in
								 * this jointree node */
	Relids		inner_join_rels;	/* base+OJ Relids syntactically included
									 * in inner joins appearing at or below
									 * this jointree node */
	Relids		left_rels;		/* if join node, Relids of the left side */
	Relids		right_rels;		/* if join node, Relids of the right side */
	Relids		nonnullable_rels;	/* if outer join, Relids of the
									 * non-nullable side */
	/* Fields filled during deconstruct_distribute: */
	SpecialJoinInfo *sjinfo;	/* if outer join, its SpecialJoinInfo */
	List	   *oj_joinclauses; /* outer join quals not yet distributed */
	List	   *lateral_clauses;	/* quals postponed from children due to
									 * lateral references */
} JoinTreeItem;
```

2. `deconstruct_distribute` - обрабатываем поочередно каждое выражение, сохраняем его для узла. Также если оно `mergejoinable`, то мы создаем новое, либо добавляем к существующему ec.

3. `deconstruct_distribute_oj_quals` - дополнительно обрабатываем условия для OUTER JOIN'ов.

На примере запроса `SELECT * FROM a JOIN b ON a.x = b.x WHERE a.x = 10`. Для создания этого множества будет пройден следующий путь.

1. После 1 этапа, мы получим следующий список из JoinTreeItem'ов (`item_list`)

![Узлы JoinTreeItem в массиве](../img/ec/item_list_1_phase.png)

2. Далее, мы будем проходить по каждому элементу этого массива и распределять все qual. Всего у нас 4 элемента массива:
   - RangeTblRef - `a`
   - RangeTblRef - `b`
   - JoinExpr   - `JOIN`
   - FromExpr   - весь запрос с условием WHERE

3. Обновление EC происходит в функции `deconstruct_distribute` и только при обработке тех JTI, которые имеют условия - RangeTblRef их не имеют, поэтому их пропускаем (первые 2 в обработке)

4. Далее, мы обрабатываем JoinExpr. У нас имеется 1 условие JOIN'а - `a.x = b.x`. Мы обрабатываем его внутри `distribute_quals_to_rels` - в нем мы пробегаем по всем условиям и для каждого вызываем функцию `distribute_qual_to_rel`.

Если посмотрим на переданные условия (все), то там окажется только 1 элемент - и это наше условие:

![Единственный элемент массива ограничений](../img/ec/clauses_distribute_elems.png)

5. Спускаемся на уровень `distribute_qual_to_rels`. В начале мы пропускаем проверки на некоторые случаи (lateral запрос, outer join, var-free выражения и т.д.). И создаем `RestrictInfo` - представление выражения ограничения, далее работа планировщика будет вестись с этой структурой - в ней находится наше выражение `a.x = b.x` (обе структуры Var - столбцы)

![Создание RestrictInfo](../img/ec/create_restrict_info.png)

6. После создания проверяются некоторые свойства - в частности `mergejoinable` (внутри `check_mergejoinable` функции). Но это не простой флаг - это список из OID семейств операторов (`get_mergejoin_opfamilies`)

![Определение mergejoinability](../img/ec/check_mergejoinable.png)

7. Мы нашли нужное семейство операторов. Так как в примере мы использовали `int` типы, то в этом списке будет OID 1976 - зарезервированный.

![Список OID семейства операндов](../img/ec/amopfamilies.png)

8. Доходим до главной функции - `process_equivalence`. Она просматривает `RestrictInfo` и *обновляет или создает EC*. Эта функция состоит из 2 частей: находим существующие EC и их обновление

9. Нахождение существующих EC - это 2 вложенных цикла: проходим по всем EC (`PlannerInfo->eq_classes`) и внутри каждого проходим по каждому члену множества и проверяем на равенство (`EquivalenceClass->ec_members`)

![Нахождение EC](../img/ec/find_ec.png)

10. Когда мы их нашли, то на выбор у нас 4 варианта: NULL/NULL, ec1/NULL, NULL/ec2, ec1/ec2. Так как мы просматриваем эти столбцы первый раз, то наткнемся на 1 случай. Тогда, мы создадим новый EC в который добавим эти 2 элемента.

![Созданный EC](../img/ec/created_ec_1_case.png)

На скриншоте можно заметить, что в этом множестве (список `ec_members`) содержатся 2 элемента - наши столбцы (`Var`).

Также для более удобной работы мы сохраняем EC и EM в самом RI

11. После мы переходим к следующему `JoinTreeItem`. На этот раз - это `FromExpr`. Здесь мы будем работать с условиями `WHERE` (он этот узел их хранит). В нем мы должны обработать условие `a.x = 10`.

![Обработка FromExpr](../img/ec/fromexpr_process.png)

На скриншоте показан список `quals` - условий в `WHERE`. В нем только 1 условие - столбец и константа

12. Далее, мы также проходим по всем шагам - так же как и для предыдущего условия. Эти части мы пропустим и остановимся в `process_equivalence` на месте, когда находим очередные EC для каждой части операндов.

В нашем списке уже есть 1 EC. Когда мы будем обходить его члены, то найдем равный столбец:

![Найденный равный столбец](../img/ec/find_equal_em.png)

На скриншоте можно видеть, что item1 (левый операнд) и cur_em->em_expr (выражение текущего члена EC) указывают на один и тот же столбец (varno, varattno).

Но вот дальше, проверка не пройдет, т.к. правый операнд - константа (на скриншоте виден тип узла).

13. В тот раз мы наткнулись на 1 случай (NULL/NULL), но теперь мы нашли EC (как минимум 1 не NULL). Тогда мы добавляем другой элемент (константу) в найденный EC

![Результирующий EC](../img/ec/ec_final_members.png)

На скриншоте видно, что в этом множестве теперь находится 3 элемента: 2 столбца и 1 константа.

---

Теперь стоит рассмотреть, где это может быть использовано. Для примера можно использовать тот же самый запрос. Без индексов у нас будет NestedLoop с 2 SeqScan:

```text
postgres=# explain SELECT * FROM a JOIN b ON a.x = b.x WHERE a.x = 10;
                          QUERY PLAN                           
---------------------------------------------------------------
 Nested Loop  (cost=0.00..85.89 rows=169 width=8)
   ->  Seq Scan on a  (cost=0.00..41.88 rows=13 width=4)
         Filter: (x = 10)
   ->  Materialize  (cost=0.00..41.94 rows=13 width=4)
         ->  Seq Scan on b  (cost=0.00..41.88 rows=13 width=4)
               Filter: (x = 10)
(6 rows)
```

Но стоит нам добавить индексы на столбцы `x`, то план будет использовать индексы.

```text
postgres=# EXPLAIN SELECT * FROM a JOIN b ON a.x = b.x WHERE a.x = 10;
                                  QUERY PLAN                                  
------------------------------------------------------------------------------
 Nested Loop  (cost=0.31..511.05 rows=169 width=8)
   ->  Index Only Scan using a_x_idx on a  (cost=0.15..36.38 rows=13 width=4)
         Index Cond: (x = 10)
   ->  Index Only Scan using b_x_idx on b  (cost=0.15..36.38 rows=13 width=4)
         Index Cond: (x = 10)
(5 rows)
```

> Я выставил `enable_material = off` и `enable_bitmapscan = off`, т.к. в первоначальном плане использовался Bitmap Heap/Index Scan (не по плану) и Material узлы

Можно заметить, что в плане использовался фильтр `b.x = 10`, хотя мы его прямо не указывали. Рассмотрим, как мы вывели это.

Известный факт - планировщик выводит все возможные выражения условий из данных. В случае нашего запроса будет создано условие `b.x = 10`. Этим занимается функция `generate_base_implied_equalities`.

Это происходит практически сразу после `deconstruct_jointree`, когда мы обработали все элементы `jointree`.

1. В функции `generate_base_implied_equalities` мы проходим по всем EC и для каждого вызываем `generate_base_implied_equalities_const` (т.к. у в нашем EC была константа)

![Цикл с генерацией всех возможных условий](../img/ec/generate_equalities_loop.png)

2. Внутри же логика довольно тривиальна - находим константу среди всех EM, а далее проходим другим EM и создаем выражение равенства с этой константой

![Создание всех пар выражений](../img/ec/loop_over_em_create.png)

3. Само же выражение добавляется в функции `distribute_restrictinfo_to_rels`. Внутри нее проверяется 2 случая - `RestrictInfo->required_rels` имеет 1 или несколько элементов. Мы натыкаемся на 1 случай, т.к. имеется константа, а значит другой элемент должен быть столбцом (т.е. для вычисления требуется 1 отношение). Другой случай наступает когда в выражении имеется несколько отношений - выражение для JOIN'а.

![Добавление сгенерированного выражения в список](../img/ec/add_generated_qual_to_rel.png)

Теперь нам доступно это выражение в списке ограничений для этого отношения из любого участка кода.

Таким образом, внутреннее представление запроса при переводе в текстовый вид следующее:

```sql
SELECT * FROM a JOIN b ON a.x = b.x WHERE a.x = 10 AND b.x = 10
```

А из этого можно заключить, что мы можем использовать индекс.
