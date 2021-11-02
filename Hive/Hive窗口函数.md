# Hive 窗口函数

它和Group By不同，Group By对分组范围内的数据进行聚合统计，得到当前分组的一条结果，而窗口函数则是对每条数据进行处理时，都会展开一个窗口范围，分析后（聚合、筛选）得到一条对应结果。

所以Group By结果数等于分组数，而窗口函数结果数等于数据总数。

## 静态窗口（Partition By）

over可以使用partition by、rows between .. and ..、range between .. and ..子句进行数据范围划分，默认范围是全部数据，并且支持使用order by进行排序。。
```
over (partition by <cols> order by <cols> rows|range between … and … )
```
窗口划分之后，在对当前窗口范围数据进行处理时，可以搭配聚合函数、排名函数、自定义函数使用。


在窗口函数计算时，可以使用order by子句对窗口内的数据进行排序。
```
order by {cols} [asc|desc] [nulls first|nulls last]
--asc|desc：指定了排列顺序
--nulls first|nulls last：指定了包含空值的返回行应出现在有序序列中的第一个
```

### row_number、rank、dense_rank 区别
row_number ：1 2 3 4 
rank ： 1 2 2 4
dense_rank ： 1 2 2 3
三个函数区别就是 row_number 唯一，rank 断层的重复排名，dense_rank 重复排名但是不会断层

## 动态窗口（Rows、Range）

除了可以使用Partition By对数据进行分组，还可以使用between .. and ..对窗口范围进行动态指定。

可以使用rows和range关键字，但它们的粒度不同，rows直接对行数进行指定，而range则是对处理值的范围进行指定。

指定范围时，n preceding表示向前n个粒度（rows、range），n following表示向后n个粒度，当n为unbounded时表示边界值，current row表示当前行。

```
--当前行，+前后各3行，一共7行
rows between 3 preceding and 3 following
--当前value值为中心，前后各浮动3个数，范围是(value-3, value+3)
range between 3 preceding and 3 following
--从数据第一行（边界）到当前行
rows between unbounded preceding and current rows
--当前行到最后一行
rows between current rows and unbounded following
--所有行，不做限定
rows between unbounded preceding and unbounded following
--所有范围，不做限定
range between unbounded preceding and unbounded following
```

使用range between 会将第一行数据当做边界值，用边界值对第一行数据进行累加，计算到当前数据为止的累加数据时会有不同。

但如果没有重复数据，则range between和rows between处理结果相同。