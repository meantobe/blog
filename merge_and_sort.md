---
title: 多数据源的高效归并分页排序
date: 2019-08-15 12:50:00
tags:
---

我们的用户持仓接口原本很简单，从一张单独的表A中做分页查询，按时间倒序排列，接口形式如下：

```
/user/holding/list?userId={}&pageNo={}&rows={}
```

查询数据库时需要几个连表操作，较为复杂，不过能够满足分页查询的效率。

## 多数据源归并分页，第一个实现（有bug）

产品提了一个需求，想要把另一类用户持仓放在一起展示。从表B中取出数据，排序规则相同，按时间倒序。

首先想到的方案是通过时间戳控制分页来归并数据。从A、B中各取N条数据，合并后取时间戳最大的前N条，核心代码如下

```java
List<Resp> queryPagedListByLimitTime(long userId, int rows, long limitTime) {
    List<Resp> totalList = new ArrayList<>();
    totalList.addAll(aService.getListByLimitTime(userId, rows, limitTime));
    totalList.addAll(bService.getListByLimitTime(userId, rows, limitTime));
    return totalList.stream()
                    .sorted(Comparator.comparing(Resp::getCreateTime).reversed())
                    .limit(rows)
                    .map(/* do business */)
                    .collect(Collectors.toList());
}
```

分别查询A、B服务时的SQL如下

```sql
SELECT xxx,xxx,xxx,xxx
FROM A
WHERE userId = #{userId} AND createTime < #{limitTime}
LIMIT #{rows}
```

翻页时，需要传递上一页中最后一条的时间戳，第一次请求时传递当前时间戳。因此接口设计变为

```
/user/holding/list?userId={}&rows={}&limitTime={}
```

这个实现简单而高效，但是上线后发现有丢数据的情况。因为系统有批量下单功能，导致许多持仓数据的createTime字段毫秒值都相同。而查询时传入当页的最后一条时间戳，因此下一页中按小于此时间戳查询，就丢失了跨页的数据。

## Redis归并排序分页，第二个实现

使用时间戳排序有分页的缺陷，因此接口API需要变为

```
/user/holding/list?userId={}&pageNo={}&rows={}
```

首先查询逻辑不变，但是现在需要尽可能查出所有数据完成排序。因此将用户数据缓存到Redis的ZSET中，score是用于排序的字段（时间戳）。
```java
void putUserDataIntoRedis(long userId) {
    String userSetKey = RedisConstants.HOLDING + "userId:" + userId;
    List<Resp> totalList = new ArrayList<>();
    totalList.add(aService.getAllList(userId));
    totalList.add(bService.getAllList(userId));
    if (!totalList.isEmpty()) {
          Set<ZSetOperations.TypedTuple<Resp>> sets = totalList.stream()
                  .map(holding -> (ZSetOperations.TypedTuple<Resp>) new DefaultTypedTuple<>(holding, (double) holding.getCreateTime()))
                  .collect(Collectors.toSet());
          BoundZSetOperations<String, String> boundZSetOperations = redisTemplate.boundZSetOps(userSetKey);
          boundZSetOperations.add(sets);
    }
}
```

获取数据时通过Redis的ZSET取数据，以实现翻页时的高效。
```java
List<Resp> queryPagedListFromRedis(long userId, int rows, int pageNo) {
    String userSetKey = RedisConstants.HOLDING + "userId:" + userId;
    BoundZSetOperations<String, Resp> boundZSetOperations = redisTemplate.boundZSetOps(userSetKey);
    Set<ZSetOperations.TypedTuple<Resp>> totalRemainSet = boundZSetOperations.reverseRange(pageNo * rows, (pageNo + 1) * rows);
    return totalRemainSet.stream()
              .map(ZSetOperations.TypedTuple::getValue)
              .collect(Collectors.toList());
}
```

由于此时接口已经上线，因此接口设计无法变更。那么如何处理跨页的数据丢失问题？客户端表示可以每页返回超过请求的rows数量，那么我们可以考虑在一页中把下一页中相同时间戳的数据一并返回。考虑避免修改数据访问层，在中间层做一些处理。

```java
    // 获取小于入参limitTime的所有数据（需要过滤掉等于limitTime的数据）
    Set<ZSetOperations.TypedTuple<String>> totalRemainSet = boundZSetOperations.reverseRangeByScoreWithScores(0, limitTime-1);

    List<Resp> result = new ArrayList<>();
    double lastTimeStamp = 0;
    for (ZSetOperations.TypedTuple<Resp> val : totalRemainSet) {
        if (Math.abs(val.getScore() - lastTimeStamp) >= 1 && result.size() >= rows) {
            // 时间戳不同，且超过每页条数，退出并返回当前数据
            break;
        }
        // 时间戳相同 或 没有超过每页条数，则加入
        result.add(val.getValue());
        lastTimeStamp = val.getScore();
    }
```

这个方法存在几个问题：
1. 需要在用户首次进入时获取全量数据，效率无法保证。
2. 数据放在缓存中，需要更新维护，增加了系统复杂度。
3. （对于旧客户端的兼容）破坏了接口的约定，请求传入rows=15返回却可能是rows=200。

测试对于持仓较多的用户，这个方案性能过低，因此最终未能上线。

## 覆盖索引，第三个实现
第二个方案虽然未上线，但是思路有可取之处。总结上面两个方案可知：
1. 由于时间戳有重复，因此以limitTime做入参是不可行的，需要分页方式查询。
2. 由于数据源不同，因此需要以相同排序条件查出后归并。但是若通过标记id等辅助分页字段方式分页，则需要增加接口字段，增加复杂度。
3. 全量数据归并后排序就不需要辅助字段，可保持接口参数不变，但是需要高效的查询全量数据方式。

由于排序时仅需要根据createTime排序，因此获取全量数据可改为仅获取id和createTime两个字段，排序后再通过id查询信息。

增加查询这两个字段的方法，结果包装为RespIds对象。数据库建立userId和createTime两个字段的索引，使得该查询可以通过覆盖索引直接返回，无须回表。

```sql
SELECT id, createTime
FROM A
WHERE userId = #{userId}
```

改造查询列表接口

```java
List<Resp> queryPagedList(long userId, int rows, int pageNo) {
    List<RespIds> totalList = new ArrayList<>();
    totalList.addAll(aService.getAllIds(userId));
    totalList.addAll(bService.getAllIds(userId));

    List<RespIds> curPageIds = totalList.stream()
                        .sorted(Comparator.comparing(RespIds::getCreateDateLong).reversed())
                        .skip(pageNo * rows)
                        .limit(rows)
                        .collect(Collectors.toList());
    Set<Long> aIds = curPageIds.stream().filter(RespIds::isA)
                        .map(RespIds::getId)
                        .collect(Collectors.toSet());
    Set<Long> bIds = curPageIds.stream().filter(RespIds::isB)
                        .map(RespIds::getId)
                        .collect(Collectors.toSet());
    Map<Long, Resp> respSet = new HashMap<>();
    if (aIds.size() > 0) {
          respSet.putAll(aService.queryDetails(req, aIds));
    }
    if (bIds.size() > 0) {
          respSet.putAll(bService.queryDetails(req, bIds));
    }

    return curPageIds.stream()
                     .map(id -> respSet.getOrDefault(id.getId(), null))
                     .map(/* do business */)
                     .collect(Collectors.toList());
}
```

获取信息时queryDetails通过主键索引id查询，也可以保证效率。

对客户端接口参数中的limitTime改为pageNo，对于旧版本客户端limitTime稍作处理即可实现兼容。在此不赘述。

方案三上线后和方案一效率基本相同，但是避免了方案一的遗漏数据的问题。且对于单个用户具有大量数据的情况下，方案三表现优于方案一。
