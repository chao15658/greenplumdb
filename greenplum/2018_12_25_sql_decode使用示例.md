
sql

decode()函数
用于复杂语义，处理了if/else，可代替coalesce（）

select decode(a,b,c,d);
当a==b时，返回c，否则返回d

a,b,c,d可为值或者表达式。

rank over()用于局部排序
