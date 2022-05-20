# Laravel 用法之 Database 模块使用

## 目录
- [连接](#连接)
- [运行原生 SQL 查询](#运行原生-SQL-查询)
- [数据库事务](#数据库事务)
- [查询日志](#查询日志)
- [查询构建器](#查询构建器)
- [获取结果集](#获取结果集)
- [查询](#查询)
- [连接](#连接)
- [联合](#联合)
- [where 子句](#where-子句)
- [排序 分组 限定](#排序-分组-限定)
- [插入](#插入) 
- [更新](#更新)
- [删除](#删除)
- [悲观锁](#悲观锁)

## 详细目录
- [连接](#连接)
- [运行原生 SQL 查询](#运行原生-SQL-查询)
   - [select 查询](#select-查询)
   - [update 查询](#update-查询)
   - [insert 查询](#insert-查询)
   - [delete 查询](#delete-查询)
   - [通用语句执行](#通用语句执行)
- [数据库事务](#数据库事务)
   - [手动使用事务](#手动使用事务)
- [查询日志](#查询日志)
- [查询构建器](#查询构建器)
- [获取结果集](#获取结果集)
   - [从一张表中取出所有行](#从一张表中取出所有行)
   - [从一张表中获取一行或一列](#从一张表中获取一行或一列)
   - [从一张表中获取组块结果集](#从一张表中获取组块结果集)
   - [获取数据列值列表](#获取数据列值列表)
   - [聚合函数](#聚合函数)
- [查询](#查询)
   - [指定查询子句](#指定查询子句)
   - [原生表达式](#原生表达式)
- [连接](#连接)
   - [内连接（等值连接）](#内连接-等值连接)
   - [左连接](#左连接)
   - [高级连接语句](#高级连接语句)
- [联合](#联合)
- [where 子句](#where-子句)
   - [简单 where 子句](#简单-where-子句)
   - [更多 where 子句](#更多-where-子句)
   - [高级 where 子句](#高级-where-子句)
- [排序 分组 限定](#排序-分组-限定)
- [插入](#插入) 
   - [自增ID](#自增ID)
- [更新](#更新)
- [删除](#删除)
- [悲观锁](#悲观锁)

- [参考](#参考)

## 连接

获取系统目前的 connection 配置

```php
DB::getConnections()
```

## 运行原生 SQL 查询

使用多个数据库连接的时候，可以使用 DB 门面的 connection 方法访问每个连接。传递给 connection 方法的连接名对应配置文件 config/database.php 中相应的连接：

```php
$users = DB::connection('foo')->select(...);
```
DB 门面为每种查询提供了相应方法：select, update, insert, delete, 和 statement。

### select 查询

```php
$users = DB::select('select * from users where active = ?', [1]);
```

传递给 select 方法的第一个参数是原生的 SQL 语句，第二个参数需要绑定到查询的参数绑定，通常，这些都是 where 字句约束中的值。参数绑定可以避免 SQL 注入攻击。

select方法以数组的形式返回结果集，数组中的每一个结果都是一个PHP StdClass 对象。

除了使用 ? 占位符来代表参数绑定外，还可以使用命名绑定来执行查询：

```php
$results = DB::select('select * from users where id = :id', ['id' => 1]);
```

### update 查询
update 方法用于更新数据库中已存在的记录，该方法返回受更新语句影响的行数：

```php
$affected = DB::update('update users set votes = 100 where name = ?', ['John']);
```

### insert 查询
使用 DB 门面的 insert 方法执行插入语句。和 select 一样，改方法将原生 SQL 语句作为第一个参数，将绑定作为第二个参数：

```php
DB::insert('insert into users (id, name) values (?, ?)', [1, 'Dayle']);
```

### delete 查询
delete 方法用于删除数据库中已存在的记录，和 update 一样，该语句返回被删除的行数：

```php
$deleted = DB::delete('delete from users');
```

### 通用语句执行
有些数据库语句不返回任何值，对于这种类型的操作，可以使用 DB 门面的 statement 方法：

```php
DB::statement('drop table users');
```

## 数据库事务
想要在一个数据库事务中运行一连串操作，可以使用 DB 门面的 transaction 方法，如果事务闭包中抛出异常，事务将会自动回滚。如果闭包执行成功，事务将会自动提交。使用 transaction 方法时不需要担心手动回滚或提交：

```php
DB::transaction(function () {
    DB::table('users')->update(['votes' => 1]);
    DB::table('posts')->delete();
});
```

### 手动使用事务

如果你想要手动开始事务从而对回滚和提交有一个完整的控制，可以使用 DB 门面的 beginTransaction 方法：

```php
DB::beginTransaction();
```

你可以通过 rollBack 方法回滚事务：

```php
DB::rollBack();
```

最后，你可以通过commit方法提交事务：

```php
DB::commit();
```
## 查询日志
Laravel 默认会为当前请求执行的的所有查询生成日志并保存在内存中。 因此， 在某些特殊的情况下， 比如一次性向数据库中插入大量数据， 就可能导致内存不足。 在这种情况下，你可以通过 disableQueryLog 方法来关闭查询日志:

```php
DB::connection()->disableQueryLog();
```

开启日志的方法为 enableQueryLog()； 清理日志 flushQueryLog；

调用 getQueryLog 方法可以同时获取多个查询执行后的日志:

```php
$queries = DB::getQueryLog();
```

## 查询构建器
数据库查询构建器提供了一个方便的、平滑的接口来创建和运行数据库查询。查询构建器可以用于执行应用中大部分数据库操作，并且能够在支持的所有数据库系统上工作。

## 获取结果集
### 从一张表中取出所有行
和原生查询一样，get 方法返回结果集的数据组，其中每一个结果都是 PHP 对象的StdClass实例。

```php
$users = DB::table('users')->get();
```

### 从一张表中获取一行或一列
如果你只是想要从数据表中获取一行数据，可以使用 first 方法，该方法将会返回单个 StdClass 对象：

```php
$user = DB::table('users')->where('name', 'John')->first();
```

### 从一张表中获取组块结果集
如果你需要处理成千上百条数据库记录，可以考虑使用 chunk 方法，该方法一次获取结果集的一小块，然后填充每一小块数据到要处理的闭包，该方法在编写处理大量数据库记录的Artisan命令的时候非常有用。比如，我们可以将处理全部users表数据处理成一次处理 100 记录的小组块：

```php
DB::table('users')->chunk(100, function($users) {
    foreach ($users as $user) {
        //
    }
});
```

你可以通过从闭包函数中返回 false 来中止组块的运行：
```php
DB::table('users')->chunk(100, function($users) {
    // 处理结果集...
    return false;
});
```

### 获取数据列值列表

如果想要获取包含单个列值的数组，可以使用 lists 方法，在本例中，我们获取所有 title 的数组：

```php
$titles = DB::table('roles')->lists('title');
```

在还可以在返回数组中为列值指定更多的自定义键（该自定义键必须是该表的其它字段列名，否则会报错）：

```php
$roles = DB::table('roles')->lists('title', 'name');

foreach ($roles as $name => $title) {
    echo $title;
}
```

### 聚合函数
队列构建器还提供了很多聚合方法，比如count, max, min, avg, 和 sum，你可以在构造查询之后调用这些方法：

```php
$users = DB::table('users')->count();
$price = DB::table('orders')->max('price');
```

当然，你可以联合其它查询字句和聚合函数来构建查询：

```php
$price = DB::table('orders')->where('finalized', 1)->avg('price');
```

## 查询

### 指定查询子句
当然，我们并不总是想要获取数据表的所有列，使用 select 方法，你可以为查询指定自定义的 select 子句：

```php
$users = DB::table('users')->select('name', 'email as user_email')->get();
```

distinct 方法允许你强制查询返回不重复的结果集：

```php
$users = DB::table('users')->distinct()->get();
```

如果你已经有了一个查询构建器实例并且希望添加一个查询列到已存在的select子句，可以使用addSelect方法：

```php
$query = DB::table('users')->select('name');
$users = $query->addSelect('age')->get();
```

### 原生表达式

有时候你希望在查询中使用原生表达式，这些表达式将会以字符串的形式注入到查询中，所以要格外小心避免被SQL 注入。想要创建一个原生表达式，可以使用 DB::raw 方法：

```php
$users = DB::table('users')
            ->select(DB::raw('count(*) as user_count, status'))
            ->where('status', '<>', 1)
            ->groupBy('status')
            ->get();
```

## 连接

### 内连接 等值连接
查询构建器还可以用于编写基本的 SQL “内连接”，你可以使用查询构建器实例上的 join 方法，传递给 join 方法的第一次参数是你需要连接到的表名，剩余的其它参数则是为连接指定的列约束，当然，正如你所看到的，你可以在单个查询中连接多张表：

```php
$users = DB::table('users')
            ->join('contacts', 'users.id', '=', 'contacts.user_id')
            ->join('orders', 'users.id', '=', 'orders.user_id')
            ->select('users.*', 'contacts.phone', 'orders.price')
            ->get();
```

### 左连接
如果你是想要执行“左连接”而不是“内连接”，可以使用 leftJoin 方法。该方法和 join 方法的使用方法一样：

```php
$users = DB::table('users')->leftJoin('posts', 'users.id', '=', 'posts.user_id')->get();
```

### 高级连接语句
你还可以指定更多的高级连接子句，传递一个闭包到 join 方法作为该方法的第2个参数，该闭包将会返回允许你指定join子句约束的JoinClause对象：

```php
DB::table('users')
        ->join('contacts', function ($join) {
            $join->on('users.id', '=', 'contacts.user_id')->orOn(...);
        })
        ->get();
```

如果你想要在连接中使用“where”风格的子句，可以在查询中使用 where 和 orWhere 方法。这些方法将会将列和值进行比较而不是列和列进行比较：

```php
DB::table('users')
        ->join('contacts', function ($join) {
            $join->on('users.id', '=', 'contacts.user_id')->where('contacts.user_id', '>', 5);
        })
        ->get();
```

## 联合

查询构建器还提供了一条“联合”两个查询的快捷方式，比如，你要创建一个独立的查询，然后使用union方法将其和第二个查询进行联合：

```php
$first = DB::table('users')->whereNull('first_name');

$users = DB::table('users')->whereNull('last_name')->union($first)->get();
```

unionAll 方法也是有效的，并且和 union 有同样的使用方法。

## where 子句

### 简单 where 子句 
And

```php
$users = DB::table('users')->where('votes', '=', 100)->get();
或
$users = DB::table('users')->where('votes', 100)->get();
```

Or

```php
$users = DB::table('users')->where('votes', '>', 100)->orWhere('name', 'John')->get();
```

### 更多 where 子句
whereBetween 方法验证列值是否在给定值之间：

```php
$users = DB::table('users')->whereBetween('votes', [1, 100])->get();
```

whereNotBetween 方法验证列值不在给定值之间：

```php
$users = DB::table('users')->whereNotBetween('votes', [1, 100])->get();
```

whereIn 方法验证给定列的值是否在给定数组中：

```php
$users = DB::table('users')->whereIn('id', [1, 2, 3])->get();
```

whereNotIn 方法验证给定列的值不在给定数组中：

```php
$users = DB::table('users')->whereNotIn('id', [1, 2, 3])->get();
```

whereNull 方法验证给定列的值为 NULL：

```php
$users = DB::table('users')->whereNull('updated_at')->get();
```

whereNotNull 方法验证给定列的值不是 NULL：

```php
$users = DB::table('users')->whereNotNull('updated_at')->get();
```

### 高级 where 子句

#### 参数分组
有时候你需要创建更加高级的 where 子句比如“ where exists ”或者嵌套的参数分组。Laravel查询构建器也可以处理这些。作为开始，让我们看一个在括号中进行分组约束的例子：

```php
DB::table('users')
            ->where('name', '=', 'John')
            ->orWhere(function ($query) {
                $query->where('votes', '>', 100)
                      ->where('title', '<>', 'Admin');
            })
            ->get();
```

正如你所看到的，传递闭包到 orWhere 方法构造查询构建器来开始一个约束分组，该闭包将会获取一个用于设置括号中包含的约束的查询构建器实例。上述语句等价于下面的 SQL：

```php
select * from users where name = 'John' or (votes > 100 and title <> 'Admin')
```

#### exists语句

whereExists 方法允许你编写 where existSQL 子句，whereExists 方法接收一个闭包参数，该闭包获取一个查询构建器实例从而允许你定义放置在“exists”子句中的查询：
```php
DB::table('users')
            ->whereExists(function ($query) {
                $query->select(DB::raw(1))
                      ->from('orders')
                      ->whereRaw('orders.user_id = users.id');
            })
            ->get();
```

上述查询等价于下面的 SQL 语句：

```sql
select * from users
where exists (
    select 1 from orders where orders.user_id = users.id
)
```

## 排序 分组 限定

orderBy 方法允许你通过给定列对结果集进行排序，orderBy 的第一个参数应该是你希望排序的列，第二个参数控制着排序的方向—— asc 或 desc：

```php
$users = DB::table('users')->orderBy('name', 'desc')->get();
```

groupBy 和 having 方法用于对结果集进行分组，having 方法和 where 方法的用法类似：

```php
$users = DB::table('users')->groupBy('account_id')->having('account_id', '>', 100)->get();
```

havingRaw 方法可以用于设置原生字符串作为 having 子句的值，例如，我们要找到所有售价大于 $2500 的部分：

```php
$users = DB::table('orders')
                ->select('department', DB::raw('SUM(price) as total_sales'))
                ->groupBy('department')
                ->havingRaw('SUM(price) > 2500')
                ->get();
```

想要限定查询返回的结果集的数目，或者在查询中跳过给定数目的结果，可以使用 skip 和 take 方法：

```php
$users = DB::table('users')->skip(10)->take(5)->get();
```

## 插入
查询构建器还提供了 insert 方法来插入记录到数据表。insert 方法接收数组形式的列名和值进行插入操作：

```php
DB::table('users')->insert(['email' => 'john@example.com', 'votes' => 0]);
```

你甚至可以一次性通过传入多个数组来插入多条记录，每个数组代表要插入数据表的记录：

```php
DB::table('users')->insert([
    ['email' => 'taylor@example.com', 'votes' => 0],
    ['email' => 'dayle@example.com', 'votes' => 0]
]);
```

### 自增ID

如果数据表有自增ID，使用insertGetId方法来插入记录将会返回ID值：

```php
$id = DB::table('users')->insertGetId(
    ['email' => 'john@example.com', 'votes' => 0]
);
```


## 更新
当然，除了插入记录到数据库，查询构建器还可以通过使用update方法更新已有记录。update方法和insert方法一样，接收列和值的键值对数组包含要更新的列，你可以通过where子句来对update查询进行约束：

```php
DB::table('users')->where('id', 1)->update(['votes' => 1]);
```

查询构建器还提供了方便增减给定列名数值的方法。相较于编写 update 语句，这是一条捷径，提供了更好的体验和测试接口。

```php
DB::table('users')->increment('votes');
DB::table('users')->increment('votes', 5);
DB::table('users')->decrement('votes');
DB::table('users')->decrement('votes', 5);
```

在操作过程中你还可以指定额外的列进行更新：

```php
DB::table('users')->increment('votes', 1, ['name' => 'John']);
```

## 删除
当然，查询构建器还可以通过 delete 方法从表中删除记录：

```php
DB::table('users')->delete();
```

在调用 delete 方法之前可以通过添加 where 子句对 delete 语句进行约束：

```php
DB::table('users')->where('votes', '<', 100)->delete();
```

如果你希望清除整张表，也就是删除所有列并将自增ID置为 0，可以使用 truncate 方法：

```php
DB::table('users')->truncate();
```

## 悲观锁
查询构建器还包含一些方法帮助你在select语句中实现”悲观锁“。可以在查询中使用 sharedLock 方法从而在运行语句时带一把“共享锁”。共享锁可以避免被选择的行被修改直到事务提交：

```php
DB::table('users')->where('votes', '>', 100)->sharedLock()->get();
```

此外你还可以使用 lockForUpdate 方法。“for update”锁避免选择行被其它共享锁修改或删除：

```php
DB::table('users')->where('votes', '>', 100)->lockForUpdate()->get();
```

## 参考
1. <a href="https://greenlightt.github.io/2017/12/20/laravel-database_1/" target="_blank">Laravel 用法之 Database 模块 原生 SQL 及 查询构建器</a>
