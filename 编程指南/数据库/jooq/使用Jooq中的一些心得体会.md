# Jooq使用心得体会

## 事务写入
之前有同事使用SpringBoot的事务写入方式出现一些问题（使用方式导致的，非SprintBoot本身问题），所以还是建议项目中使用jooq来进行数据库操作，能避免一些误用 。

之前同事是这样使用springboot事务的：

```kotlin
@transaction
fun saveCommits()

@transaction
fun saveCommitRequirements()
```

后来程序发现出现数据不一致情况，有部分commitsRequirement存在，但是commits数据不存在，同事就这样修改了：

```kotlin
@transcation
fun saveCommitParseResult() {
  saveCommits()
  saveCommitRequirements()
}
```

这种事务套事务的场景，Sprintboot不一定支持的。所以我建议使用jooq的事务支持，以避免这种使用。

在jooq中使用事务很简单，只需要在dslContext上调用`transcation`即可，只要在这个代码块中使用对应对象写数据库，即可完成事务写入。当遇到异常时，事务会自动回滚。

**注意**：请不要创建特别大的事务，防止对数据库产生影响。

```kotlin

```

## 批量写入