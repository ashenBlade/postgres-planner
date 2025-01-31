# Планировщик PostgreSQL

Описание и небольшой туториал по изучению работы планировщика PostgreSQL

Используется версия PG 16.4

## Путь изучения

### Базовые элементы

1. [EquivalenceClass](pages/equivalenceclass.md)
2. [JoinDomain](pages/joindomain.md)
3. PHV, Var, Param
4. PathKeys
5. JoinTree
6. Добавление очередного пути
7. Создание плана выполнения из путей
8. Вычисление стоимости пути
9. Представление отношений

### Оптимизации

1. Упрощение выражений
2. subquery pull up
3. Удаление ненужных join'ов

### Прочее

1. `LATERAL` запросы
2. constraint exclusion
