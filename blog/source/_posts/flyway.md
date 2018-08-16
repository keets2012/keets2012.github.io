---
title: 数据库版本管理工具Flyway应用
date: 2018-8-15
categories: Utils
tags:
- Flyway
- 数据库管理
---

## Flyway介绍
[Flyway](https://flywaydb.org/)是一款开源的数据库版本管理工具，它更倾向于规约优于配置的方式。Flyway可以独立于应用实现管理并跟踪数据库变更，支持数据库版本自动升级，并且有一套默认的规约，不需要复杂的配置，Migrations可以写成SQL脚本，也可以写在Java代码中，不仅支持Command Line和Java API，还支持Build构建工具和Spring Boot等，同时在分布式环境下能够安全可靠地升级数据库，同时也支持失败恢复等。


## Flyway用途
通常在项目开始时会针对数据库进行全局设计，但在开发产品新特性过程中，难免会遇到需要更新数据库Schema的情况，比如：添加新表，添加新字段和约束等，这种情况在实际项目中也经常发生。那么，当开发人员完成了对数据库更的SQL脚本后，如何快速地在其他开发者机器上同步？并且如何在测试服务器上快速同步？以及如何保证集成测试能够顺利执行并通过呢？

到各测试服务器上手动执行SQL脚本费时费神费力的，干嘛不自动化呢，当然，对于高级别和PROD环境，还是需要DBA手动执行的。最后，写一段自动化程序来自动执行更新，想法是很好的，那如果已经有了一些插件或库可以帮助你更好地实现这样的功能，为何不好好利用一下呢，当然，如果是为了学习目的，重复造轮子是无可厚非的。其实，以上可以通过Flyway工具来解决，Flyway可以实现自动化的数据库版本管理，并且能够记录数据库版本更新记录。

## Flyway命令
Flyway对数据库进行版本管理主要由Metadata表和6种命令完成，Metadata主要用于记录元数据，每种命令功能和解决的问题范围不一样，以下分别对metadata表和这些命令进行阐述，其中的示意图都来自Flyway的官方文档。

### Metadata Table
Flyway中最核心的就是用于记录所有版本演化和状态的Metadata表，在Flyway首次启动时会创建默认名为`flyway_schema_history`的元数据表，其表结构为(以MySQL为例)：

![](http://image.blueskykong.com/flyway-metadata.jpg)

### Migrate
Migrate是指把数据库Schema迁移到最新版本，是Flyway工作流的核心功能，Flyway在Migrate时会检查Metadata(元数据)表，如果不存在会创建Metadata表，Metadata表主要用于记录版本变更历史以及Checksum之类的。

![](http://image.blueskykong.com/flyway-1.jpg)

Migrate时会扫描指定文件系统或Classpath下的Migrations(可以理解为数据库的版本脚本)，并且会逐一比对Metadata表中的已存在的版本记录，如果有未应用的Migrations，Flyway会获取这些Migrations并按次序Apply到数据库中，否则不需要做任何事情。另外，通常在应用程序启动时应默认执行Migrate操作，从而避免程序和数据库的不一致性。

### Clean
清除掉对应数据库Schema中的所有对象，包括表结构，视图，存储过程，函数以及所有的数据等都会被清除。Clean操作在开发和测试阶段是非常有用的，它能够帮助快速有效地更新和重新生成数据库表结构，但特别注意的是：不应在Production的数据库上使用！

### Info
Info用于打印所有Migrations的详细和状态信息，其实也是通过Metadata表和Migrations完成的，下图很好地示意了Info打印出来的信息。Info能够帮助快速定位当前的数据库版本，以及查看执行成功和失败的Migrations。

![](http://image.blueskykong.com/flyway-version-record.jpg)

### Validate
Validate是指验证已经Apply的Migrations是否有变更，Flyway是默认是开启验证的。Validate原理是对比Metadata表与本地Migrations的Checksum值，如果值相同则验证通过，否则验证失败，从而可以防止对已经Apply到数据库的本地Migrations的无意修改。

### Baseline
Baseline针对已经存在Schema结构的数据库的一种解决方案，即实现在非空数据库中新建Metadata表，并把Migrations应用到该数据库。Baseline可以应用到特定的版本，这样在已有表结构的数据库中也可以实现添加Metadata表，从而利用Flyway进行新Migrations的管理了。

### Repair
Repair操作能够修复Metadata表，该操作在Metadata表出现错误时是非常有用的。Repair会修复Metadata表的错误，通常有两种用途：

- 移除失败的Migration记录，该问题只是针对不支持DDL事务的数据库。
- 重新调整已经应用的Migratons的Checksums值，比如：某个Migratinon已经被应用，但本地进行了修改，又期望重新应用并调整Checksum值，不过尽量不要这样操作，否则可能造成其它环境失败。

## 支持的数据库
目前Flyway支持的数据库还是挺多的，包括：Oracle, SQL Server, SQL Azure, DB2, DB2 z/OS, MySQL(including Amazon RDS), MariaDB, Google Cloud SQL, PostgreSQL(including Amazon RDS and Heroku), Redshift, Vertica, H2, Hsql, Derby, SQLite, SAP HANA, solidDB, Sybase ASE and Phoenix。

## Flyway应用
Flyway可以通过命令行和插件（如maven）的方式运行相应的命令，具体可以参考`https://flywaydb.org/getstarted/firststeps/commandline`。

本文将会重点讲解在Spring Boot中应用Flyway。

### 引入依赖

```xml
<!-- https://mvnrepository.com/artifact/org.flywaydb/flyway-core -->
<dependency>
	<groupId>org.flywaydb</groupId>
	<artifactId>flyway-core</artifactId>
	<version>5.0.7</version>
</dependency>
```

### 配置Flyway

```yaml
flyway:
  locations: classpath:db/migration
  baseline-on-migrate: true
  url: jdbc:mysql://localhost:3306/test
  sql-migration-prefix: V
  sql-migration-suffix: .sql
```
我们在配置文件中指定了如上的属性。Flyway的配置属性意义如下：

- flyway.baseline-version：执行基线时用来标记已有Schema的版本（默认值：1）
- flyway.enabled：开启Flyway （默认为true）
- flyway.sql-migration-prefix：SQL迁移的文件名前缀
- flyway.sql-migration-suffix ：SQL迁移的文件名后缀
- flyway.baseline-on-migrate：在没有元数据表的情况下，针对非空Schema执行迁移时是否自动调用基线
- flyway.location：迁移脚本的位置（默认为db/migration）


#### 正确创建Migrations

Migrations是指Flyway在更新数据库时是使用的版本脚本，比如：一个基于Sql的Migration命名为V1__init_tables.sql，内容即是创建所有表的sql语句，另外，Flyway也支持基于Java的Migration。Flyway加载Migrations的默认Locations为classpath:db/migration，也可以指定filesystem:/project/folder，其加载是在Runtime自动递归地执行的。

除了需要指定Location外，Flyway对Migrations的扫描还必须遵从一定的命名模式，Migration主要分为两类：Versioned和Repeatable。

- Versioned migrations   
	一般常用的是Versioned类型，用于版本升级，每一个版本都有一个唯一的标识并且只能被应用一次，并且不能再修改已经加载过的Migrations，因为Metadata表会记录其Checksum值。其中的version标识版本号，由一个或多个数字构成，数字之间的分隔符可以采用点或下划线，在运行时下划线其实也是被替换成点了，每一部分的前导零会被自动忽略。

- Repeatable migrations   
	Repeatable是指可重复加载的Migrations，其每一次的更新会影响Checksum值，然后都会被重新加载，并不用于版本升级。对于管理不稳定的数据库对象的更新时非常有用。Repeatable的Migrations总是在Versioned之后按顺序执行，但开发者必须自己维护脚本并且确保可以重复执行，通常会在sql语句中使用CREATE OR REPLACE来保证可重复执行。

![](http://image.blueskykong.com/sql_migration_naming.png)

其中的文件名由以下部分组成，除了使用默认配置外，某些部分还可自定义规则。

- prefix: 可配置，前缀标识，默认值V表示Versioned，R表示Repeatable
- version: 标识版本号，由一个或多个数字构成，数字之间的分隔符可用点.或下划线_
- separator: 可配置，用于分隔版本标识与描述信息，默认为两个下划线__
- description: 描述信息，文字之间可以用下划线或空格分隔
- suffix: 可配置，后续标识，默认为.sql

![](http://image.blueskykong.com/sql_migration_base_dir%20(1).png)
### 创建sql脚本文件

![](http://image.blueskykong.com/flyway-scripts.jpg)

如上所示即为我们在服务中创建的sql脚本，启动服务之后会看到如下的日志信息：

![](http://image.blueskykong.com/flyway-boot.jpg)

说明当前数据库脚本是最新的，schema_version表中最新的版本为1.4。

## 总结
本文主要介绍了Flyway，包括其提供的6中命令和如何使用Flyway。Flyway可以有效改善数据库版本管理方式，通常在local和dev环境中简化了很多不必要的繁琐操作，当然在产线上应用Flyway需要非常慎重，可能会对线上的数据造成非常严重的后果。


### 参考
1. [Flyway](https://flywaydb.org/documentation/)
2. [快速掌握和使用Flyway](https://blog.waterstrong.me/flyway-in-practice/)
