---
layout:     post
title:      Greenplum表的种类、原理以及如何选择
subtitle:   Greenplum行存、列存、AO表的原理介绍和选择
date:       2019-02-22
author:     LANY
header-img: img/post-20190217-bg.png
catalog: true
tags:
    - Pivotal
    - Greenplum
    - 数据建模
---
# Greenplum行存、列存、AO表的原理和选择

## 行存和列存的原理

### 行存

以行为形式组织存储，一行是一个tuple，存在一起。当需要读取某列时，需要将这列前面的所有列都进行deform，所以访问第一列和访问最后一列的成本实际上是不一样的。

优势：

- 数据顺序写入block中，持续写入的情况下，一条记录命中在一个块中，IO开销相对较小，速度较快。

- 查询多个字段时，因为记录在一个块中命中，速度较快。

劣势：

- 查询少量字段时，也要访问整条记录，造成一定的IO浪费。

- 行存储的压缩比有限。

行存小结：

- 全表扫描要扫描更多的数据块。

- 压缩比较低。

- 读取任意列的成本不一样，越靠后的列，成本越高。

- 不适合向量计算、JIT架构。（简单来说，就是不适合批处理形式的计算）

- 需要REWRITE表时，需要对全表进行REWRITE，例如加字段有默认值。

### 列存

以列为形式组织存储，每列对应一个或一批文件。读取任一列的成本是一样的，但是如果要读取多列，需要访问多个文件，访问的列越多，开销越大。

优势：

- 数据按列存储，压缩比可以做到很高。

- 当查询少量字段时，扫描的块更少，可以节约IO还能提升效率。

劣势：

- 因为时按列存储的，当需要查询大量字段时，或者查询的记录偏少时，会造成离散IO较多。
例如查询一条记录的20个列，行存储可能只需要扫描一个块，而列存储至少要扫描20个块。

- 由于IO的放大，列存储不适合OLAP应用场景，按列做较大范围的聚合分析，或者JOIN分析。

列存小结：

- 压缩比高。

- 仅仅支持AO存储（后面会将）。

- 读取任意列的成本是一样的。

- 非常适合向量计算、JIT架构。对大批量数据的访问和统计，效率更高。

- 读取很多列时，由于需要访问更多的文件，成本更高。例如查询明细。

- 需要REWRITE表时，不需要对全表操作，例如加字段有默认值，只是添加字段对应的那个文件。

### 什么时候选择行存

如果OLTP的需求偏多，例如经常需要查询表的明细（输出很多列），需要更多的更新和删除操作时。可以考虑行存。

### 什么时候选择列存

如果OLAP的需求偏多，经常需要对数据进行统计时，选择列存。

需要比较高的压缩比时，选择列存。

如果用户有混合需求，可以采用分区表，例如按时间维度的需求分区，近期的数据明细查询多，那就使用行存，对历史的数据统计需求多那就使用列存。

## 堆表和AO表的原理

### 堆表
实际上就是PG的堆存储，堆表的所有变更都会产生REDO，可以实现时间点恢复。但是堆表不能实现逻辑增量备份（因为表的任意一个数据块都有可能变更，不方便通过堆存储来记录位点。）。

一个事务结束时，通过clog以及REDO来实现它的可靠性。同时支持通过REDO来构建MIRROR节点实现数据冗余。

### AO表
看名字就知道，只追加的存储，删除更新数据时，通过另一个BITMAP文件来标记被删除的行，通过bit以及偏移对齐来判定AO表上的某一行是否被删除。

事务结束时，需要调用FSYNC，记录最后一次写入对应的数据块的偏移。（并且这个数据块即使只有一条记录，下次再发起事务又会重新追加一个数据块）同时发送对应的数据块给MIRROR实现数据冗余。

因此AO表不适合小事务，因为每次事务结束都会FSYNC，同时事务结束后这个数据块即使有空余也不会被复用。（你可以测试一下，AO表单条提交的IO放大很严重）。

虽然如此，AO表非常适合OLAP场景，批量的数据写入，高压缩比，逻辑备份支持增量备份，因此每次记录备份到的偏移量即可。加上每次备份全量的BITMAP删除标记（很小）。

### 什么时候选择堆表

当数据写入时，小事务偏多时选择堆表。

当需要时间点恢复时，选择堆表。

### 什么时候选择AO表

当需要列存时，选择AO表。

当数据批量写入时，选择AO表。

### 行存和列存性能对比

```SQL
-- 创建一个函数，用于创建400列的表（行存储表，ao行存表，ao列存表）
create or replace function f(name, int, text) returns void as $$  
declare  
  res text := '';  
begin  
  for i in 1..$2 loop  
    res := res||'c'||i||' int8,';  
  end loop;  
  res := rtrim(res, ',');  
  if $3 = 'ao_col' then  
    res := 'create table '||$1||'('||res||') with  (appendonly=true, blocksize=8192, compresstype=none, orientation=column)';  
  elsif $3 = 'ao_row' then  
    res := 'create table '||$1||'('||res||') with  (appendonly=true, blocksize=8192, orientation=row)';  
  elsif $3 = 'heap_row' then  
    res := 'create table '||$1||'('||res||') with  (appendonly=false)';  
  else  
    raise notice 'use ao_col, ao_row, heap_row as $3';  
    return;  
  end if;  
  execute res;  
end;  
$$ language plpgsql; 

-- 创建行表
select f('tbl_ao_col', 400, 'ao_col');
select f('tbl_ao_row', 400, 'ao_row'); 
select f('tbl_heap_row', 400, 'heap_row');


-- 创建1个函数，用于填充数据，其中第一个和最后3个字段为测试数据的字段，其他都填充1。
create or replace function f_ins1(name, int, int8) returns void as $$  
declare  
  res text := '';  
begin  
  for i in 1..($2-4) loop  
    res := res||'1,';  
  end loop;  
  res := 'id,'||res;  
  res := rtrim(res, ',');  
  res := 'insert into '||$1||' select '||res||'id,random()*10000,random()*100000 from generate_series(1,'||$3||') t(id)';  
  execute res;  
end;  
$$ language plpgsql; 

-- 填充数据
select f_ins1('tbl_ao_col',400,1000000);


-- 创建1个函数，用于填充数据，其中前4个字段为测试数据的字段，其他都填充1。
create or replace function f_ins2(name, int, int8) returns void as $$  
declare  
  res text := '';  
begin  
  for i in 1..($2-4) loop  
    res := res||'1,';  
  end loop;  
  res := 'id,id,random()*10000,random()*100000,'||res;  
  res := rtrim(res, ',');  
  res := 'insert into '||$1||' select '||res||' from generate_series(1,'||$3||') t(id)';  
  execute res;  
end;  
$$ language plpgsql; 

-- 插入数据
insert into tbl_ao_row select * from tbl_ao_col; 
insert into tbl_heap_row select * from tbl_ao_col;

-- 分析表
analyze tbl_heap_row;
analyze tbl_ao_row;
analyze tbl_ao_col;

-- 查看表占用的空间
select pg_size_pretty(pg_relation_size('tbl_heap_row'));
select pg_size_pretty(pg_relation_size('tbl_ao_row'));
select pg_size_pretty(pg_relation_size('tbl_ao_col'));


explain analyze select c2,count(*),sum(c3),avg(c3),min(c3),max(c3) from tbl_heap_row group by c2;
explain analyze select c398,count(*),sum(c399),avg(c399),min(c399),max(c399) from tbl_heap_row group by c398;  
explain analyze select c2,count(*),sum(c3),avg(c3),min(c3),max(c3) from tbl_ao_row group by c2;
explain analyze select c398,count(*),sum(c399),avg(c399),min(c399),max(c399) from tbl_ao_row group by c398; 
explain analyze select c2,count(*),sum(c3),avg(c3),min(c3),max(c3) from tbl_ao_col group by c2;
explain analyze select c398,count(*),sum(c399),avg(c399),min(c399),max(c399) from tbl_ao_col group by c398; 



```