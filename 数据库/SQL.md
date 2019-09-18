# inner Join 和 outer join  

# LEFT OUTER JOIN 和 RIGHT OUTER JOIN 的区别  


# DROP TRUNCATE DELETE 的区别
DROP 直接删除掉表   
truncate 删除的是表中的数据，再插入数据时自增长的数据id又重新从1开始  
delete 删除表中数据，可以在后面添加where字句  

delete是数据库操作语言，这个操作会被放到rollback segement中，事务提交之后才生效，如果有相应的trigger，执行的时候将被触发  
truncate、drop是数据库定义语言，操作立即生效，原数据不放到rollback segment中，不能回滚，操作不触发trigger   




# having 与 where 的区别  
select price, name from goods where price > 100 
select price, name from goods having price > 100 正确，和where等价  
select name from goods having price > 100 错误，having是从前筛选的字段再筛选，而where是从数据表中的字段直接进行筛选  
  
where 是一个约束声明，使用where来约束来自数据库的数据，where是在结果返回之前起作用，不能使用聚集函数  
having 是一个过滤声明，是在查询返回结果集以后对查询结果进行的过滤操作，可以使用聚合函数  
having 可以让我们筛选成组后的各组数据，where在聚合前先筛选记录  

# 四类  
查询：select   
操作：update、delete、insert    
定义：create、drop、truncate  
控制：commit、rollback、grant（分配权限）， revoke（收回权限）  