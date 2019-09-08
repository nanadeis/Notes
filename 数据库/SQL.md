# inner Join 和 outer join  

# LEFT OUTER JOIN 和 RIGHT OUTER JOIN 的区别  


# DROP TRUNCATE DELETE 的区别
DROP 直接删除掉表   
truncate 删除的是表中的数据，再插入数据时自增长的数据id又重新从1开始  
delete 删除表中数据，可以在后面添加where字句  

delete是数据库操作语言，这个操作会被放到rollback segement中，事务提交之后才生效，如果有相应的trigger，执行的时候将被触发  
truncate、drop是数据库定义语言，操作立即生效，原数据不放到rollback segment中，不能回滚，操作不触发trigger   




# having 与 where 的区别  
having子句中的谓词在形成分组后才起作用，可以使用聚集函数  
where中不能使用聚集函数  