### 排名问题函数
sum()over(partation by  order by )
raw_number()over(partition by $ order by $)
先用row对id排序，连续天数主要用date_sub(date_a, row) 日期相等则为连续登入
case when then
### 行列转换
lateral view explode  列转行 跟表后面
to_map(key, value) kv 列转行 kv['想要key'] as 名
### 日期函数
date_format(data,'yyyy-MM-dd HH')
### 解析json: json_tuple和get_json_object
- select get_json_object('{"movie":"594","rate":"4","timeStamp":"978302268","uid":"1"}','$.movie');
- select b.b_movie,b.b_rate,b.b_timeStamp,b.b_uid from json a lateral view json_tuple(a.data,'movie','rate','timeStamp','uid') b as b_movie,b_rate,b_timeStamp,b_uid;

### sort by 一般和 group by 合并使用
分组内排序
