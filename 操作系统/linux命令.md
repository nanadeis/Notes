# awk  
文本分析工具，它依次处理文件的每一行，并读取里面的每一个字段（默认空格隔开）。     
awk -F ‘：’ 指定分隔符为冒号  
awk ‘{[pattern] action}’ {filenames}  

## 变量
awk '{print $0}' demo.txt
变量$0代表当前行，所以这个命令会把每一行原样打印出来  
变量$1 $2 $3分别表示每一行的第一个、第二个、第三个字段  
变量NF表示当前行有多少个字段，因此$NF就代表最后一个字段  
$(NF-1)代表倒数第二个字段  
变量NR表示当前处理的是第几行   
print命令里面，如果原样输出字符，要放在双引号里  

## 函数  
toupper（）将字符转为大写  

## 条件  
允许指定输出条件，只输出符合条件的行  
awk -F ‘：’ ‘NR % 2 == 1 {print $0}' demo.txt  
只输出奇数行  

还提供了if else 语句  

## BEGIN END  


# sed  

# wc  
输出行数，字数，字节数   

# free  


# pmap  
查看进程虚拟地址空间的使用情况 pmap -d $pid  


# PS
-T 可以开启线程查看  

# TOP  
-H 可以实时显示各个线程情况



# traceroute  

# find 
find <path> <expression> <cmd>
path 搜索的目录及其所有子目录。默认为当前目录  
expression 所要搜索的文件的特征  
-name 按照文件名查找文件  
-perm 按照文件权限查找文件  
-type 查找某一类型的文件  
-user 按照文件属主来查找文件  
-mtime -n +n 按照文件的更改时间来查找文件， -n表示文件更改时间距现在n天以内，+n表示文件更改时间距现在n天以前  
-size n[c] 查找文件长度为n块的文件，带有c时表示文件长度以字节计  
cmd 对搜索结果进行特定的处理  

# grep  
grep [选项] pattern [文件名]  

