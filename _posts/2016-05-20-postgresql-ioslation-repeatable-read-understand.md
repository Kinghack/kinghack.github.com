---
layout: post
title: "对PostgreSQL 事务级别REPEATEDABLE READ的理解"
description: ""
category: 
tags: []
---
{% include JB/setup %}
这个开发迭代周期，主要是会将交易闭环做掉，即为Have完成线上交易的功能。正如我的前一篇[Blog](http://blog.qiuqiu.info/22/11/2015/have-server-step-by-step)为未来准备所说的，未来一定上交易，肯定还是会上关系型数据库，当初的数据库选择MongoDB主要是为对他的熟悉程度超过MySQL和PostgreSQL。但一旦涉及到交易相关的，必然还是需要上更稳妥的数据库选择。对于MySQL和PostgreSQL的选择就不说了，但后者对GIS和json的支持让我现在一点也不犹豫。因为如果这块摸熟了，会趁着现在规模少，把MongoDB里的数据迁移进来的。至于部署PostgreSQL，时间效率要求，不折腾配置了，直接使用AWS的RDS服务，目前测试阶段感觉不错，但真正线上稳定与否，需要上线后再评估。

这两天主要是确认了下PostgreSQL的事务设计。详细的可以看[这里](http://www.postgresql.org/docs/current/static/transaction-iso.html
)。
那当然的，为了更加严谨，我们会选择*repeaded read isolation*。于是我就写了些code来验证他的事务性质。测试的记录status原值为7。

code如下：

```go
	db, err := sql.Open("postgres", "postgres://user:pwd@localhost/have?sslmode=disable")
	defer db.Close()
	tran, _ := db.Begin()
	//开始事务
	log.Println("first transction begin")
	id := 1
	rows, _ := tran.Query("UPDATE orderb SET status = $1 WHERE id = $2", 6, id)
	//去更新status的值
	rows.Close()
	go func(){
		anothertrans, _ := db.Begin()
		//开第二个事务
		//time.Sleep(time.Second * 3)//here success    
		r, e := anothertrans.Query("UPDATE orderb SET status = status + 1 WHERE id = $1", id)
		//同样更新status这个值
		//time.Sleep(time.Second * 3)//here failed
		if e == nil {
			r.Close()
			log.Println("second transciton", anothertrans.Commit())
		} else {
			log.Println("second transciton", e)
			anothertrans.Rollback()
		}
	}()
	time.Sleep(time.Second)
	log.Println("after sleep")
		log.Println("first transciton", tran.Commit())
	log.Println("waiting")
	time.Sleep(time.Second * 20)
```

将success注释，fail加进来，则运行结果如下：

```shell
first transction begin
after sleep
first transciton <nil>
waiting
second transciton pq: could not serialize access due to concurrent update
```

可以看到，第二个事务失败了。
这个可以理解，原因是，第二个事务在去update的时候，第一个事务已经提交了，因此是去更改了一个在这个事务开始后更改的值。

而将success那行留着，而将fail注释掉，运行结果如下，可以看到，都update成功了；

```shell
first transction begin
after sleep
first transciton <nil>
waiting
second transciton <nil>
```

这里我就开始迷糊了。因为在groutine了sleep了3秒，理论上第一个事务也提交了，照道理应该与第一个情况类似，对这个事务等级，都应该报错才对。但竟然能成功。

于是我就继续翻文档，文档里对repeated read ioslation原话是：

> The Repeatable Read isolation level only sees data committed before the transaction began; it never sees either uncommitted data or changes committed during transaction execution by concurrent transactions. (However, the query does see the effects of previous updates executed within its own transaction, even though they are not yet committed.) 

换句话说，这个事务里应该只读得到事务开始前其他事务commit的更新；与我的理解还蛮符合的，这个时候，我在想，会不会是服务器虽然配置了事务隔离级别，但这个对话的session还是read commited这个级别，因为，事务级别是根据各家的驱动自己实现的，https://golang.org/pkg/database/sql/#DB.Begin
而pg默认就是read committed.

于是我更改了code，如下：

```go
	db, err := sql.Open("postgres", "postgres://user:pwd@localhost/have?sslmode=disable")
	defer db.Close()
	tran, _ := db.Begin()
	log.Println("first transction begin")
	id := 1
	rows, _ := tran.Query("UPDATE orderb SET status = $1 WHERE id = $2", 6, id)
	rows.Close()
	go func(){
			anothertrans, _ := db.Begin()
		log.Println("second transction begin")
		time.Sleep(time.Second * 2)//success    
		tt, ee := anothertrans.Query("SELECT status from orderb WHERE id = 1")
		log.Println(ee)
		for tt.Next() {
			var status int
			tt.Scan(&status)
			log.Println("status", status)
		}
		tt.Close()
		anothertrans, _ := db2.Begin()
		log.Println("second transction begin")
		tt, ee := anothertrans.Query("SELECT status from orderb WHERE id = 1")
		log.Println(ee)
		for tt.Next() {
			var status int
			tt.Scan(&status)
			log.Println("status", status)
		}
		//time.Sleep(time.Second * 3)//success    
		r, e := anothertrans.Query("UPDATE orderb SET status = status + 1 WHERE id = $1", id)
		//time.Sleep(time.Second * 3)//here failed
		if e == nil {
			r.Close()
			log.Println("second transciton", anothertrans.Commit())
		} else {
			log.Println("second transciton", e)
			anothertrans.Rollback()
		}
	}()
	time.Sleep(time.Second)
	log.Println("after sleep")
		log.Println("first transciton", tran.Commit())
	log.Println("waiting")
	time.Sleep(time.Second * 20)
```
	
	
运行结果变成了：

```shell
first transction begin
second transction begin
after sleep
first transciton <nil>
waiting
<nil>
status 6
second transciton pq: could not serialize access due to concurrent update
```

可以看到，是之前能update成功的代码比，只是在第二个事务更新前，去尝试读了下status的值，发现竟然读到了6，即第二个事务开始后，第一个事务commit的值，这显然与刚才的理解不同，那是什么状况呢？难道真的是事务级别被drivers的默认值改了？于是继续加了一段代码测试，如下：

```go
	db, err := sql.Open("postgres", "postgres://user:pwd@localhost/have?sslmode=disable")
	defer db.Close()
	tran, _ := db.Begin()
	log.Println("first transction begin")
	id := 1
	rows, _ := tran.Query("UPDATE orderb SET status = $1 WHERE id = $2", 6, id)
	rows.Close()
	go func(){
			anothertrans, _ := db.Begin()
				log.Println("second transction begin")
		tt, ee := anothertrans.Query("SELECT status from orderb WHERE id = 1")
		log.Println(ee)
		for tt.Next() {
			var status int
			tt.Scan(&status)
			log.Println("status", status)
		}
		tt.Close()
		time.Sleep(time.Second * 2)
		tt, ee = anothertrans.Query("SELECT status from orderb WHERE id = 1")
		log.Println(ee)
		for tt.Next() {
			var status int
			tt.Scan(&status)
			log.Println("status", status)
		}
		tt.Close()
		//time.Sleep(time.Second * 3)//success    
		r, e := anothertrans.Query("UPDATE orderb SET status = status + 1 WHERE id = $1", id)
		//time.Sleep(time.Second * 3)//here failed
		if e == nil {
			r.Close()
			log.Println("second transciton", anothertrans.Commit())
		} else {
			log.Println("second transciton", e)
			anothertrans.Rollback()
		}
	}()
	time.Sleep(time.Second)
	log.Println("after sleep")
		log.Println("first transciton", tran.Commit())
	log.Println("waiting")
	time.Sleep(time.Second * 20)
```
	
	
跟上面的代码区别是，不等第一个事务commit，先尝试读status，然后等第一个事务commit了，再尝试读status的值，如果真的是read comitted这个level的话，理论上两次读到的值应该不同，但，结果如下：

```shell
first transction begin
second transction begin
<nil>
status 7
after sleep
first transciton <nil>
waiting
<nil>
status 7
second transciton pq: could not serialize access due to concurrent update
```

发现两次读到的值是相同的，那显然也不是read committed的行为。很困惑的时候，偶然翻到了[这个](http://www.postgresql.org/docs/current/static/sql-set-transaction.html
)：

> REPEATABLE READ
> All statements of the current transaction can only see rows committed before the first query or data-modification statement was executed in this transaction.

这里我的理解就是，repeatable read这个level，*并不是*只能读到事务开始前commit的值，就像我在很多说明和文档以及postgresql官方文档里说的那样，而是能读到，在这个事务开始后，第一个select或任何改变rows的语句前commmit的值。这个也能解释最后一段代码的行为：第一次读到了7，因为第一个事务还没commit，第二次读，虽然commit了，但因为这个事务级别，前面select过了，所以只能读到这个select之前的commit，也就是也只能读到7，不能读到6，自然的，后面的更新就继续失败了。。。


*槽点*是：官方文档两处的按我的英语理解来看，解释不一致。。。。



