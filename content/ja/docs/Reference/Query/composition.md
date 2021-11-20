---
title: "Composition"
linkTitle: "Composition"
weight: 110
description: >
  クエリの合成
---

## 概要 {#overview}

クエリとその構成要素である宣言は合成をサポートします。

## クエリ {#query}

Komapperにおけるクエリは以下のクラスのいずれかもしくは両方で表現されます。

`org.komapper.core.dsl.query.Query<T>`
: `Database`インスタンスを介して実行するとデータベースにアクセスし`T`型の値を返すクエリ。合成可能です。

`org.komapper.core.dsl.query.FlowQuery<T>`
: `Database`インスタンスを介して実行すると`kotlinx.coroutines.flow.Flow<T>`型の値を返すクエリ。データベースアクセスは`Flow`が`collect`されたときに初めて行われます。合成はサポートされていません。

{{< alert title="Note" >}}
`FlowQuery<T>`は`R2dbcDatabase`インスタンスによってのみ実行可能です。
{{< /alert >}}

### andThen {#query-composition-andthen}

`andThen`関数を使うと、まとめて実行して最後の結果を返すクエリを構築できます。

```kotlin
val q1: Query<Address> = QueryDsl.insert(a).single(Address(16, "STREET 16", 0))
val q2: Query<Address> = QueryDsl.insert(a).single(Address(17, "STREET 17", 0))
val q3: Query<List<Address>> = QueryDsl.from(a).where { a.addressId inList listOf(16, 17) }
val query: Query<List<Address>> = q1.andThen(q2).andThen(q3)
val list: List<Address> = db.runQuery { query }
/*
insert into ADDRESS (ADDRESS_ID, STREET, VERSION) values (?, ?, ?)
insert into ADDRESS (ADDRESS_ID, STREET, VERSION) values (?, ?, ?)
select t0_.ADDRESS_ID, t0_.STREET, t0_.VERSION from ADDRESS as t0_ where t0_.ADDRESS_ID in (?, ?)
*/
```

### map {#query-composition-map}

`map`関数を使うと、クエリ結果に変更を加えるクエリを構築できます。

```kotlin
val query: Query<List<Address>> = QueryDsl.from(a).map { 
  it.map { address -> address.copy(version = 100) }
}
/*
select t0_.ADDRESS_ID, t0_.STREET, t0_.VERSION from ADDRESS as t0_ 
*/
```

### zip {#query-composition-zip}

`map`関数を使うと、2つのクエリ結果を`Pair`型で返すクエリを構築できます。

```kotlin
val q1 = QueryDsl.insert(a).single(Address(16, "STREET 16", 0))
val q2 = QueryDsl.from(a)
val query: Query<Pair<Address, List<Address>>> = q1.zip(q2)
/*
insert into ADDRESS (ADDRESS_ID, STREET, VERSION) values (?, ?, ?)
select t0_.ADDRESS_ID, t0_.STREET, t0_.VERSION from ADDRESS as t0_
*/
```

### flatMap {#query-composition-flatmap}

`flatMap`関数を使うと、1番目のクエリの実行結果を受け取って2番目のクエリを実行し2番目の結果を返すクエリを構築できます。

```kotlin
val q1: Query<Address> = QueryDsl.insert(a).single(Address(16, "STREET 16", 0)) // 1st query
val query: Query<List<Employee>> = q1.flatMap { newAddress ->
    QueryDsl.from(e).where { e.addressId less newAddress.addressId } // 2nd query
}
val list: List<Employee> = db.runQuery { query }
/*
insert into ADDRESS (ADDRESS_ID, STREET, VERSION) values (?, ?, ?)
select t0_.EMPLOYEE_ID, t0_.EMPLOYEE_NO, t0_.EMPLOYEE_NAME, t0_.MANAGER_ID, t0_.HIREDATE, t0_.SALARY, t0_.DEPARTMENT_ID, t0_.ADDRESS_ID, t0_.VERSION from EMPLOYEE as t0_ where t0_.ADDRESS_ID < ?
*/
```

### flatZip {#query-composition-flatzip}

`flatZip`関数を使うと、1番目のクエリの実行結果を受け取って2番目のクエリを実行し1番目と2番目の結果を`Pair`型で返すクエリを構築できます。

```kotlin
val q1: Query<Address> = QueryDsl.insert(a).single(Address(16, "STREET 16", 0)) // 1st query
val query: Query<Pair<Address, List<Employee>>> = q1.flatZip { newAddress ->
    QueryDsl.from(e).where { e.addressId less newAddress.addressId } // 2nd query
}
val pair: Pair<Address, List<Employee>> = db.runQuery { query }
/*
insert into ADDRESS (ADDRESS_ID, STREET, VERSION) values (?, ?, ?)
select t0_.EMPLOYEE_ID, t0_.EMPLOYEE_NO, t0_.EMPLOYEE_NAME, t0_.MANAGER_ID, t0_.HIREDATE, t0_.SALARY, t0_.DEPARTMENT_ID, t0_.ADDRESS_ID, t0_.VERSION from EMPLOYEE as t0_ where t0_.ADDRESS_ID < ?
*/
```

## 宣言 {#declaration}

Query DSLでは、例えば`where`関数や`having`関数に検索条件を表すラムダ式を渡しますが、
Komapperではこれらのラムダ式のことを宣言と呼びます。

宣言には以下のものがあります。

- Having宣言 - `HavingDeclaration`
- On宣言 - `OnDeclaration`
- Set宣言 - `SetDeclaration`
- Values宣言 - `ValuesDeclaration`
- When宣言 - `WhenDeclaration`
- Where宣言 - `WhereDeclaration`

宣言は合成できます。

### plus {#declaration-composition-plus}

`+`演算子を使うと、被演算子の宣言内部に持つ式を順番に実行するような新たな宣言を構築できます。

```kotlin
val w1: WhereDeclaration = {
    a.addressId eq 1
}
val w2: WhereDeclaration = {
    a.version eq 1
}
val w3: WhereDeclaration = w1 + w2 // +演算子の利用
val query: Query<List<Address>> = QueryDsl.from(a).where(w3)
val list: List<Address> = db.runQuery { query }
/*
select t0_.ADDRESS_ID, t0_.STREET, t0_.VERSION from ADDRESS as t0_ where t0_.ADDRESS_ID = ? and t0_.VERSION = ?
*/
```

`+`演算子はすべての宣言で利用できます。

### and {#declaration-composition-and}

`and`関数を使うと、宣言を`and`演算子で連結する新たな宣言を構築できます。

```kotlin
val w1: WhereDeclaration = {
    a.addressId eq 1
}
val w2: WhereDeclaration = {
    a.version eq 1
    or { a.version eq 2 }
}
val w3: WhereDeclaration = w1.and(w2) // and関数の利用
val query: Query<List<Address>> = QueryDsl.from(a).where(w3)
val list: List<Address> = db.runQuery { query }
/*
select t0_.ADDRESS_ID, t0_.STREET, t0_.VERSION from ADDRESS as t0_ where t0_.ADDRESS_ID = ? and (t0_.VERSION = ? or (t0_.VERSION = ?))
*/
```

`and`関数は、Having宣言、When宣言、Where宣言に対して適用できます。

### or {#declaration-composition-or}

`or`関数を使うと、宣言を`or`演算子で連結する新たな宣言を構築できます。

```kotlin
val w1: WhereDeclaration = {
    a.addressId eq 1
}
val w2: WhereDeclaration = {
    a.version eq 1
    a.street eq "STREET 1"
}
val w3: WhereDeclaration = w1.or(w2) // or関数の利用
val query: Query<List<Address>> = QueryDsl.from(a).where(w3)
val list: List<Address> = db.runQuery { query }
/*
select t0_.ADDRESS_ID, t0_.STREET, t0_.VERSION from ADDRESS as t0_ where t0_.ADDRESS_ID = ? or (t0_.VERSION = ? and t0_.STREET = ?)
*/
```