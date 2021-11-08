## linux过滤命令 grep -A / -B / -C 去固定行的上下几行
grep -A n ''  
-A 找出匹配及输出下n行  
-B 找出匹配及输出上n行  
-C 找出匹配及输出上下n行  
-i 不区分大小写

## sed
```
grep '***' | sed 's/^.*addr://g' | sed 's/Bcast.*$//g'  
```
^. 是前面数据  
.*$ 是后面面数据  
sed -i 's/\.$/\!/g'  
-i是修改当前文件 \$末尾是.的替换和^用法一致  
-n 搜索只打印包含模板  
-e 执行多个命令 sed -e '3,$d' -e''
## awk
-F, 和 -va=1 用过的的  
后面接行判断命令  
```
awk '$1>2 && $2=="Are" {print $1,$2,$3}' log.txt 
```

```
awk 'BEGIN{printf "%4s %4s %4s %4s %4s %4s %4s %4s %4s\n","FILENAME","ARGC","FNR","FS","NF","NR","OFS","ORS","RS";printf "---------------------------------------------\n"} {printf "%4s %4s %4s %4s %4s %4s %4s %4s %4s\n",FILENAME,ARGC,FNR,FS,NF,NR,OFS,ORS,RS}' 
```  
awk 'length>2' ls.log length长度函数