# JetLinks Query DSL 规则

本文件用于 JetLinks CRUD 中的 `createQuery()`、`QueryParamEntity`、`toQuery()`、`toNestQuery()` 查询 DSL。适用于排序、嵌套条件、动态 QueryParam、AssetsHolder 查询注入、自定义 termType 与分页查询。

## 先选查询入口

1. 外部请求已有 `QueryParamEntity`
   - 优先 `createQuery().setParam(param)`，再补服务端固定条件。
   - 如果只需要在请求参数上追加条件或排序，可用 `param.toQuery()` / `param.toNestQuery(...)`。

2. 服务内部固定查询
   - 优先 `createQuery().where(...).and(...).fetch()`。
   - 不要为了几个固定条件构造控制器私有接口或手写 SQL。

3. 需要构造可复用请求参数
   - 使用 `QueryParamEntity.newQuery()...getParam()`。
   - 用于命令、订阅、远程调用、AssetsHolder 注入前的参数构造。

## 常用链路

### 基础条件

```java
return service
    .createQuery()
    .where(Entity::getState, state)
    .and(Entity::getType, type)
    .fetch();
```

- `where(field, value)` 常用于第一个条件。
- `and(field, value)` 追加 AND 条件。
- `in(field, values)` 处理集合匹配；集合为空时先判断，避免生成无意义查询。
- `not`、`notIn`、`gte`、`lte`、`between`、`like` 等只在当前模块已有用法或语义明确时使用。
- 字符串字段名只在时序表、动态列、跨实体 DTO 或没有方法引用时使用；普通实体优先方法引用。

### 合并 QueryParam 与服务端条件

```java
return service
    .createQuery()
    .setParam(param)
    .where(Entity::getType, type)
    .orderBy(SortOrder.desc(Entity::getId))
    .fetch();
```

- `setParam(param)` 承接前端筛选、分页、排序和 termType。
- 服务端强制条件继续追加在 DSL 上，例如类型、租户上下文、状态约束、业务边界。
- 追加条件前先判断是否会和前端同字段条件冲突；冲突时要明确是“收窄”还是“覆盖”。

### 排序

```java
query.orderBy(SortOrder.desc(Entity::getCreateTime),
              SortOrder.desc(Entity::getId));
```

- 分页查询必须有稳定排序；时间排序后建议补 `id`、`createTime`、`sequence` 等稳定 tie-breaker。
- 用户未传排序时，可在 `queryParam.getSorts().isEmpty()` 后用 `queryParam.toQuery().orderByAsc(...)` 或 `orderByDesc(...)` 补默认排序。
- 单字段排序可用 `orderByAsc(Entity::getSortIndex)` / `orderByDesc(...)`，多字段排序优先 `orderBy(SortOrder.asc/desc(...), ...)`。
- 时序、日志、动态列可用字符串列名，例如 `orderByDesc("timestamp")`；业务实体优先方法引用，避免字段名漂移。

### 条件化追加

```java
return service
    .createQuery()
    .where(Entity::getChannelId, channelId)
    .when(CollectionUtils.isNotEmpty(types),
          query -> query.in(Entity::getType, types))
    .fetch();
```

- 可选条件优先用 `when(condition, query -> ...)`，不要手写多套重复查询。
- 空集合、空字符串、零值时间窗口要先判断；不要把空条件交给数据库或 term builder 猜语义。

### 嵌套 OR / AND

```java
return service
    .createQuery()
    .where(Entity::getState, enabled)
    .nest(nest -> {
        nest.is(Entity::getOwnerId, userId)
            .or()
            .is(Entity::getCreatorId, userId)
            .or()
            .accept(Entity::getId, CustomTermBuilder.TERM_TYPE, terms);
    })
    .fetch();
```

- `nest(...)` 用于把一组 OR 条件包成括号，常见语义是 `state = enabled AND (owner = ? OR creator = ? OR exists(...))`。
- 使用链式 `nest()` / `orNest()` 时，记得在子条件结束后 `.end()` 回到父查询。
- 不要把 OR 条件平铺到主查询上，容易把权限、状态或租户边界放宽。
- 自定义关联条件优先通过 `accept(field, termType, value)` 或 `QueryParamEntity.newQuery().getParam().getTerms()` 接入，具体 termType 规则见 `dynamic-term-rules.md`。

### QueryParamEntity 构造和改写

```java
QueryParamEntity query = QueryParamEntity
    .newQuery()
    .where(Entity::getState, enabled)
    .in(Entity::getType, types)
    .getParam();
```

- 构造请求参数后交给仓储：`repository.createQuery().setParam(query).fetch()`。
- 对外部传入参数追加条件时优先 `param.toNestQuery(q -> q.and(...))`，保留原分页、排序和筛选。
- 只补默认排序时，直接 `param.toQuery().orderByAsc(...)`，不要重建整个 QueryParam。
- 需要取消分页或指定分页时，跟随当前模块已有 `noPaging()` / `doPaging(page, size)` 用法。

### AssetsHolder 查询注入

```java
return AssetsHolder
    .injectConditional(service.createQuery().where().in(Entity::getId, ids),
                       AssetType.xxx,
                       Entity::getId)
    .flatMapMany(query -> query.fetch());
```

- CRUD 查询、详情、导出、批量更新 / 删除涉及数据权限时，先路由到 `$jetlinks-assets-permission`。
- 对 `QueryParamEntity` 使用 `AssetsHolder.injectQueryParam(...)`；对 DSL 条件使用 `AssetsHolder.injectConditional(...)`。
- 权限注入后再 `fetch()` / `count()` / `execute()`，不要先查全量再内存过滤。

## 反模式

- 为一个筛选字段新增专用接口，而 `_query` / QueryParam 已能表达。
- 查询条件同时散落在 Controller、Service、前端私有字段和 SQL 字符串里。
- 分页查询没有稳定排序，导致翻页重复或漏数据。
- OR 条件没有用 `nest` 包住，意外放宽状态、权限或数据范围。
- 为了省事手写租户、部门、创建人过滤，而不走 AssetsHolder。
- 只 mock `fetch()` 返回值，不验证 DSL 是否包含关键条件、排序和权限注入。

## 测试目标

- 断言核心条件：状态、类型、所属对象、时间范围、关键 termType。
- 断言排序：默认排序、用户排序、稳定 tie-breaker。
- 断言嵌套：`A AND (B OR C)` 不应退化成 `(A AND B) OR C`。
- 断言权限：允许和拒绝的 AssetsHolder 绑定都要覆盖。
- 断言分页：有稳定顺序，边界页不重复、不漏数据。
