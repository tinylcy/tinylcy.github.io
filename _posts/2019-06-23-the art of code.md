---
title: 码出高效码出质量
---

### 编程规约

#### 命名风格

* POJO 类中布尔类型变量都不要加 is 前缀，否则部分框架解析会引起序列化错误。注意，在 MySQL 规约中，表达是与否的值采用 is_xxx 的命名方式，所以，需要在 resultMap 设置从 is_xxx 到 xxx 的映射关系。
* 对于 Service 和 DAO 类，暴露出来的服务一定是接口，内部的实现类用 Impl 的后缀与接口区别。如果是形容能力的接口名称，取对于的形容词为接口名（通常是 able 的形容词）。
* 各层命名规约
  * Service/DAO 层方法命名规约
    * 获取单个对象的方法用 get 做前缀
    * 获取多个对象的方法用 list 做前缀，复数形式结尾如：listObjects
    * 获取统计值的方法用 count 做前缀
    * 插入的方法用 save/insert 做前缀
    * 删除的方法用 remove/delete 做前缀
    * 修改的方法用 update 做前缀
  * 领域模型命名规约
    * 数据对象：xxxDO
    * 数据传输对象：xxxDTO
    * 展示对象：xxxVO

#### OOP 规约

* 浮点数之间的等值判断，基本数据类型不能用 == 来比较，包装数据类型不能用 equals 来判断。

  ```java
  public class FloatEqualsDemo {
      public static void main(String []args) {
          /****************反例******************/
          float a = 1.0f - 0.9f;
          float b = 0.9f - 0.8f;
  
          if (a == b) {
              // 不会进入此代码块
              System.out.println("float a == float b");
          }
  
          Float x = Float.valueOf(a);
          Float y = Float.valueOf(b);
          if (x.equals(y)) {
              // 不会进入此代码块
              System.out.println("Float x == Float y");
          }
  
          /****************正例******************/
          float diff = 1e-6f;
          if (Math.abs(a - b) < diff) {
              System.out.println("float a == float b");
          }
  
          BigDecimal p = new BigDecimal("1.0");
          BigDecimal q = new BigDecimal("0.9");
          BigDecimal r = new BigDecimal("0.8");
  
          BigDecimal m = p.subtract(q);
          BigDecimal n = q.subtract(r);
          if (m.equals(n)) {
              System.out.println("BigDecimal m == BigDecimal n");
          }
      }
  }
  ```

  * 为了防止精度损失，禁止使用构造方法 BigDecimal(double) 的方式把 double 值转化为 BigDecimal 对象。优先推荐入参为 String 的构造方法，或使用 BigDecimal 的 valueOf 方法。
  * 序列化类新增属性时，不要修复 serialVersionUID 字段，避免反序列化失败。

#### 集合处理

  * ArrayList 的 subList 结果不可强制转换成 ArrayList，否则会抛出 ClassCastException 异常。subList 返回的是 ArrayList  的内部类 SubList，并不是 ArrayList 而是 ArrayList 的一个视图。对于 SubList 子列表的操作最终会反映到原列表上。
  * 使用 Map 的方法 keySet、values、entrySet 返回集合对象时，不可以对去进行添加元素的操作，否则会抛出 UnsupportedOperationException 异常。
  * Collections 类返回的对象，如 emptyList、singletonList 等都是 immutable list，不可对其进行添加或者删除元素的操作。
  * 使用集合转数组的方法，必须使用集合的 toArray(T[] array)，传入的参数是类型一致，长度为 0 的空数组。
  * 在使用 Collection 接口任何实现类的 addAll() 方法时，都要对输入的集合参数进行 NPE 判断。

#### 并发处理

* SImpleDataFormat 是线程不安全的类，一般不要定义 static 变量，如果定义为 static 变量，必须加锁，或者使用 DateUtils 工具类。如果是 JDK8 应用，建议使用 Instant 代替 Date，LocalDateTime 代替 Calendar，DateTimeFormatter 代替 SimpleDateFormat。
* 必须回收自定义的 ThreadLocal 变量，尤其是在线程池场景下。

```java
objectThreadLocal.set(userInfo);
try {
    // doSomething();
} finally {
    objectThreadLocal.remove();
}
```

* 对多个资源、数据库表、对象同时加锁时，需要保持一致的加锁顺序，否则可能会造成死锁。
* 在使用阻塞等待获取锁的方式中，必须在 try 代码块之外获取锁，并且在加锁方法与 try 代码块之间没有任何可能抛出异常的方法调用，避免加锁成功后，在 finally 中无法解锁。
  * 如果在 lock 方法与 try 代码块之间的方法调用抛出异常，那么无法解锁，造成其他线程无法成功获取锁。
  * 如果 lock 方法在 try 代码块之间，可能由于其他方法抛出异常，导致在 finally 代码块中，unlock 对未加锁的对象进行解锁，抛出 IllegalMonitorStateException 异常。

```java
/****************正例******************/
Lock lock = new XxxLock();
doSomething();
lock.lock();
// lock() 与 try 之间不能抛出异常
try {
    doOthers();
} finally {
    lock.unlock();
}

/****************反例******************/
Lock lock = new XxxLock();
try {
    // 如果 doSomething 方法抛出异常，则会直接执行 finally 代码块
    doSomething();
    // 无论加锁是否成功，finally 代码块都会执行
    lock.lock();
    doOthers();
} finally {
    lock.unlock();
}
```

* 多线程并行处理定时任务时，Timer 运行多个 TimeTask 时，只要其中之一没有捕获抛出的异常，其他任务便会自动终止运行，如果在处理定时任务时使用 ScheduledExecutorService 则没有这个问题。

#### 控制语句

* 当 switch 括号内的变量类型为 String 并且此变量为外部参数时，必须先进行 null 判断。
* 在高并发场景中，避免使用“等于”作为中断或者退出的条件。如果并发没有控制好，容易产生等值判断被“击穿”的情况，使用大于或者小于的区间判断条件来代替。

### MySQL 数据库

#### 建表规约

* 表达是与否概念的字段，必须使用 is_xxx 的方式命名，数据类型是 unsigned tinyint。
* 小数类型为 decimal，禁止使用 float 和 double。float 和 double 在存储时存在精度损失的问题，可能导致在比较值的时候，得到不正确的结果。如果存储的数据范围超过 decimal 的范围，建议将数据拆成整数和小数并分开存储。
* 如果存储的字符串长度几乎相等，使用 char 定长字符串类型。
* varchar 是可变长字符串，不预先分配存储空间，长度不要超过 5000，如果存储长度大于此值，定义字段类型为 text，独立出来一张表，用主键来对应，避免影响其他字段索引效率。

#### 索引规约

* 业务上具有唯一特性的字段，即使是多个字段的组合，也必须建成唯一索引。不要以为 insert 会影响 insert 的速度，这个速度损耗可以忽略不计，但是对提高查询速度是明显的。另外，即使在应用层做了非常完善的校验机制，只要灭有唯一索引，根据墨菲定律，必然会有脏数据产生。
* 在 varchar 字段上建立索引时，必须要指定索引长度，没有必要对全字段建立索引，根据实际文本区分度决定索引长度即可。可以使用 count(distinct left(列名, 索引长度))/count(*) 的来确定区分度。

#### SQL 语句

* 不要使用 count(列名) 或 count(常亮) 来替代 count(*)。
* 当某一列的值全为 NULL 时，count(col) 的返回结果为 0，但是 sum(col) 的返回结果为 NULL，因此使用 sum 时需要注意 NPE 问题，建议使用 SELECT IFNULL(SUM(column), 0) FROM table。
* 代码中写分页查询逻辑是，若 count 为 0 应直接返回，避免执行后面的分页语句。
* IN 操作能避免则避免，若实在避免不了，需要仔细评估 IN 后边的集合元素数量，控制在 1000 个之内。

#### ORM 映射

* 不要写一个大而全的数据更新接口。传入为 POJO 类，不管是不是自己的目标更新字段，都进行 update tabel set c1=value1,c2=value2；这是不对的。执行 SQL 时，不要更新无改动的字段，一是易出错；二是效率低；三是增加 binlog 存储。
* @Transactional 事务不要滥用，事务会影响数据库的 QPS，另外使用事务的地方需要考虑各方面的回滚方案，包括缓存回滚、搜索引擎回滚、消息补偿、统计修正等。

