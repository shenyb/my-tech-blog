---
title: '单机 MySQL 扛住千万级台账任意翻页：宽覆盖索引、延迟关联、尾页反转，以及它们各自没解决的问题'
description: '《千万级业务存储架构与实战》系列第 2 篇：一个尾页查询从 600ms~4s 降到取数 7ms 的完整过程。拆清覆盖索引、延迟关联、索引全扫三个常被混用的概念，讲透主键 id 为什么要显式放进索引第 2 列、尾页为什么要从另一头数，以及最后剩下的 90ms COUNT 该怎么处理。'
pubDate: 2026-11-18T17:30:00+08:00
heroImage: ''
category: engineering
tags: ['MySQL', '索引优化', '深分页', '存储架构', '工程化']
---

# 单机 MySQL 扛住千万级台账任意翻页：宽覆盖索引、延迟关联、尾页反转，以及它们各自没解决的问题

> **TL;DR**
> [上一篇总纲](/blog/storage-architecture-overview-and-selection/)的结论是：20 人用的内部台账，单机 MySQL 就够了。这篇把「够了」拆开讲清楚。
>
> 起点是一个很普通的慢查询：只有单列时间索引，用户填了两个没建索引的区间条件，再点一下「尾页」，接口就要 600ms，数据量大一些直接 4 秒。最后的结果是列表取数 7ms，接口总耗时 97ms。
>
> 中间做了三件事，每件事解决一个问题，也各自留下一个问题：
> - 宽覆盖索引（主键 id 显式放第 2 列）：消灭回表和 filesort，但跳过时间条件时仍是从头扫索引；
> - 延迟关联：把 offset 次随机回表变成 offset 次索引项比较，但深分页依然随 offset 线性变慢；
> - 尾页反转：后半段从另一头数，扫描量最多只有一半，但需要先拿到总数，而 COUNT 最后成了最贵的那一步。

> 📚 **《千万级业务存储架构与实战》系列**
> - 第 1 篇：[存储架构全景与选型边界推演（总纲）](/blog/storage-architecture-overview-and-selection/)
> - 第 2 篇（本文）：单机 MySQL 千万级台账任意翻页实战
> - 第 3 篇（待更）：分布式 KV 与 Key-List 列表引擎深度拆解
> - 第 4 篇（待更）：突破单机极限——TiDB 全局索引与 ES 位图检索

---

## 📌 本文要点

- 覆盖索引、延迟关联、索引全扫是三个不同的东西，分别解决「回表」「回表次数」「能不能用上索引」三个问题，混在一起讲很容易得出错误结论。
- 联合索引里主键 id 的位置决定了叶子节点的排序。显式放在排序列后面，能让 `ORDER BY time, id` 直接顺着索引读出来，而且不额外占空间。
- 延迟关联没有消灭深分页，只是让每跳过一行的成本从「一次随机 IO」降到「一次内存比较」。
- 尾页反转把最坏情况从「最后一页」移到了「正中间那页」，扫描量减半，前提是先有 total，且 COUNT 和取数之间数据不变。
- 优化做完以后，最贵的是那个用户根本不需要精确值的 COUNT。T-1 只读的数据，COUNT 结果整天都不会变，这是下一步最该利用的特征。

---

## 一、问题现场：一次「尾页」为什么要 600ms 到 4 秒

先看表结构（已脱敏，只保留和本文相关的列）：

```sql
CREATE TABLE t_loan_iou (
  id             BIGINT PRIMARY KEY AUTO_INCREMENT,
  iou_id         VARCHAR(64)   NOT NULL,   -- 借据号，唯一
  user_id        BIGINT,                   -- 客户 ID
  lend_time      DATETIME      NOT NULL,   -- 放款时间，列表默认按它倒序
  loan_status    VARCHAR(16),              -- 借据状态
  customer_type  TINYINT,                  -- 客户类型
  loan_period_cnt INT,                     -- 分期期数
  annual_rate    DECIMAL(8,4),             -- 年化定价
  loan_amt       DECIMAL(16,2),            -- 放款金额
  residue_prin   DECIMAL(16,2),            -- 借据余额
  fund_name      VARCHAR(64),              -- 资方
  -- …… 另有约 20 个展示字段
  UNIQUE KEY uk_iou_id (iou_id),
  KEY idx_user_time (user_id, lend_time),
  KEY idx_lend_time (lend_time)
);
```

列表页有十来个可选条件。按单号、客户这类精确条件查，有专门的索引，一直很快。慢的是另一类：用户什么精确条件都不填，只填几个区间，比如「借据余额 100~60000、年化定价 10%~50%」，然后点「尾页」。

```sql
SELECT * FROM t_loan_iou
WHERE residue_prin BETWEEN 100 AND 60000
  AND annual_rate  BETWEEN 10 AND 50
ORDER BY lend_time DESC, id DESC
LIMIT 85040, 10;
```

`EXPLAIN` 的结果是 `type = ALL`，`Extra = Using where; Using filesort`。这张表上只有 `idx_lend_time` 能和排序对上，为什么优化器不用？

因为用它的代价算下来更贵：沿着时间索引倒序扫，每拿到一个 id 都要回聚簇索引读整行，才能判断余额和定价满不满足；要跳过 8.5 万行，就是 8.5 万多次随机回表。优化器一比，不如直接把整张表顺序扫一遍，再在内存里排序。

于是每次翻到尾页，都是一次全表扫描加一次大排序。在测试环境 10 万行级别的表上是 600ms 左右，在 50 多万行的表上体感到了 4 秒。全量 1000 万行，结果可想而知。

---

## 二、先把三个概念拆开

网上讲深分页优化，经常一句话把「覆盖索引」「延迟关联」「索引下推」全带上。这几个东西解决的问题不同，混着讲容易误判效果。

| 概念 | 它是什么 | 解决什么 | 不解决什么 |
| :--- | :--- | :--- | :--- |
| 覆盖索引 | 查询用到的列全在某个二级索引里 | 不用回表 | 不保证能在索引上「定位」，可能要从头扫 |
| 延迟关联 | 先在索引里挑出主键，再用主键 JOIN 回原表取整行 | 把回表次数从 offset + limit 降到 limit | 跳过 offset 行的扫描本身 |
| 索引全扫（type=index） | 无法用索引定位起点，只能把索引从头到尾读一遍 | 比扫整张表读的数据少 | 扫描量与索引总行数成正比，与结果集大小无关 |

还有一个容易误会的点：联合索引「跳过前面的列就失效」这个说法不准确。更准确的说法是：

- 查询条件命中了最左列（比如带了 `lend_time` 范围），索引可以用二分找到起点，是 `range` 扫描；
- 查询条件没命中最左列，索引没法定位，但如果需要的列都在索引里，优化器仍可能选择把索引完整扫一遍（`type = index`），因为索引比表窄得多。

本文的方案在用户不填时间条件时，走的就是第二种。它快，是因为每一步便宜，而不是因为步数少。这一点决定了它的上限，第七节会再回来说。

---

## 三、第一步：设计宽覆盖索引，主键 id 放第 2 列

目标是让上面那条 SQL 的过滤和排序都在一个索引里完成。于是新增一个索引，把列表页常用、又没有专门索引的筛选列都挂上去：

```sql
ALTER TABLE t_loan_iou
  ADD INDEX idx_lend_time_cover (
    lend_time, id,
    loan_status, customer_type, loan_period_cnt,
    annual_rate, loan_amt, residue_prin, fund_name
  );
```

这里最关键的是第 2 列的 `id`。

![主键 id 的位置决定叶子节点排序](/storage-series/04-index-leaf-order.svg)

InnoDB 的二级索引，叶子节点里本来就会存主键，用来回表。如果建索引时没写 id，主键会被自动追加到所有列的最后面，叶子节点的排序就是 `(lend_time, loan_status, …, fund_name, id)`。同一个放款时间下，记录先按状态排、再按定价排，id 反而是乱的。

而列表的排序是 `ORDER BY lend_time DESC, id DESC`（加 id 是为了让同一时刻的多条记录有稳定的先后，否则翻页时会出现重复或漏行）。叶子顺序和排序要求对不上，MySQL 只能把候选结果全取出来再排一次，就是 `Using filesort`。filesort 要拿到全部候选行才能开始排，LIMIT 就没法提前结束。

把 id 显式写在 `lend_time` 后面，叶子节点的顺序就变成了 `(lend_time, id, …)`，正好等于排序要求。倒着读叶子链表，读出来的就是最终顺序，凑够 offset + 10 条就可以停。

至于空间：InnoDB 发现主键列已经出现在索引定义里，就不会在末尾再存一遍。调整位置，不多占一个字节。

### 哪些列该放进来，哪些不该

- 放进来的：列表页上没有专门索引的筛选列（状态、客户类型、期数、定价、金额、余额、资方）。这些列单独看区分度都不高，单独建索引没用，但挂在覆盖索引尾部，就能在索引项上直接判断，不用回表。
- 不放进来的：借据号、客户 ID、证件号、手机号这类精确条件。它们有自己的索引，一次能定位到几条记录，比扫覆盖索引快得多，优化器会优先选它们。
- 不删旧的 `idx_lend_time`：它更窄，只按时间范围查、不需要过滤其他列时更便宜，留给优化器选。

### 代价

宽索引不是免费的，需要算清楚三笔账：

1. 空间：9 列加起来每个索引项几十字节，1000 万行约 1GB 量级，内存放得下。
2. 写入：每次插入和更新都要多维护一棵树。我们的数据是夜里 T-1 批量写入、白天只读，这笔成本可以忽略；如果是白天高频更新的交易表，就要慎重。
3. 维护：以后列表页加一个筛选条件，要么接受这个条件回表，要么改索引。改索引在千万级表上是一次需要评估的 DDL。

---

## 四、第二步：延迟关联，先挑 id 再回表

有了覆盖索引，`SELECT *` 依然用不上它，因为 `*` 里的 30 多个字段大部分不在索引里。优化器还是会面临「回表 8.5 万次」和「全表扫」的选择。

延迟关联就是把查询拆成两步：子查询只要 id，可以完全在覆盖索引里完成；外层拿这 10 个 id 回表取整行。

![直接查与延迟关联的代价对比](/storage-series/05-deferred-join.svg)

MyBatis 里的写法（条件部分省略）：

```xml
<select id="selectPageByQuery" resultType="LoanIouEntity">
    SELECT a.*
    FROM t_loan_iou a
    INNER JOIN (
        SELECT id
        FROM t_loan_iou
        <where><include refid="conditions"/></where>
        ORDER BY lend_time DESC, id DESC
        LIMIT #{offset}, #{limit}
    ) b ON a.id = b.id
    ORDER BY a.lend_time DESC, a.id DESC
</select>
```

`EXPLAIN` 里应该看到三件事：

- 子查询：`key = idx_lend_time_cover`，`Extra = Using where; Using index`，没有 `Using filesort`；
- 外层：对 `a` 的访问是 `eq_ref`，`key = PRIMARY`；
- 外层的 `ORDER BY` 只对 10 行排序，开销可以忽略。之所以还要写，是因为 JOIN 之后结果顺序不保证和子查询一致。

这里要特别说清楚它省了什么：子查询依然要从头扫过 85040 个索引项才能拿到那 10 个 id。它省掉的是这 8.5 万次随机回表，把每跳过一行的成本，从「一次可能读磁盘的随机 IO」降成了「一次在内存索引页里的比较」。

所以加了延迟关联以后，深分页还是会随 offset 线性变慢，只是斜率小了很多。要让尾页也快，还得想办法把 offset 本身变小。

> 💡 小数据量下验证要注意：表只有几万行时，优化器可能觉得全表扫更省事，不选你的索引。我们的 EXPLAIN 核对脚本里专门备注了「以全量数据上的结果为准」，不要在小表上看到 `type = ALL` 就以为索引没建对。

---

## 五、第三步：后半段页码，从另一头数过来

用户点「尾页」，想要的是按时间倒序的最后 10 条，也就是按时间正序的最前 10 条。从最新的一头数过去要跳过几乎全部记录，从最老的一头数过来几乎不用跳。

![尾页反转示意](/storage-series/06-reverse-tail-page.svg)

做法是准备一条正序版本的 SQL，只是把子查询的排序反过来，外层依然按倒序返回：

```xml
<select id="selectPageByQueryAsc" resultType="LoanIouEntity">
    SELECT a.*
    FROM t_loan_iou a
    INNER JOIN (
        SELECT id
        FROM t_loan_iou
        <where><include refid="conditions"/></where>
        ORDER BY lend_time ASC, id ASC
        LIMIT #{offset}, #{limit}
    ) b ON a.id = b.id
    ORDER BY a.lend_time DESC, a.id DESC
</select>
```

InnoDB 的叶子链表是双向的，正着读和倒着读成本一样，同一个索引两个方向都能用上，不会出现 filesort。

Service 层负责判断走哪条，以及换算 offset：

```java
private static final int REVERSE_CHECK_OFFSET = 1000;

private List<LoanIouEntity> fetchPageRows(LoanIouQuery q, int offset, int pageSize) {
    // 前 1000 行以内直接查，不为浅翻页多付一次 COUNT
    if (offset < REVERSE_CHECK_OFFSET) {
        return mapper.selectPageByQuery(q, offset, pageSize);
    }
    long total = mapper.countByQuery(q);
    if (offset >= total) {
        return Collections.emptyList();          // 页码越界
    }
    if (offset > total / 2) {                    // 后半段：从另一头数
        int ascLimit  = (int) Math.min(pageSize, total - offset);
        int ascOffset = (int) (total - offset - ascLimit);
        return mapper.selectPageByQueryAsc(q, ascOffset, ascLimit);
    }
    return mapper.selectPageByQuery(q, offset, pageSize);
}
```

几个细节：

1. 最后一页不满一页时，`ascLimit = total - offset` 会小于 pageSize，`ascOffset` 正好是 0。比如 total = 100005、offset = 100000，就是正序取前 5 条。
2. 1000 这个阈值是为了让前 100 页（每页 10 条）完全不付 COUNT 的成本。绝大多数翻页都发生在前几页。
3. 最坏情况从「最后一页」移到了「正中间那页」：offset 接近 total / 2 时，两头数过去都要跳一半。扫描量减半，不是消失。
4. COUNT 和取数是两次独立的查询，中间如果数据变了，换算出来的 offset 会错位。我们的数据只在夜里同步，白天不变，这个问题不存在；如果是实时写入的表，要么把两次查询放在同一个一致性快照里，要么接受偶尔的错位。

---

## 六、实测结果，以及结果说明了什么

测试环境，满足条件的结果约 10.9 万行，查第 10937 页（尾页）：

| 方案 | 执行计划 | 接口耗时 |
| :--- | :--- | :--- |
| 优化前：只有 `idx_lend_time` | 全表扫 + filesort | 约 600ms（50 多万行的表上约 4 秒） |
| 优化后：覆盖索引 + 延迟关联 + 尾页反转 | 子查询 Using index，无 filesort | 97ms：COUNT 约 90ms + 取数约 7ms |

97ms 看起来是个达标的数字，但拆开看，结论其实是另一句话：取数已经只要 7ms 了，剩下的 90ms 全部花在 COUNT 上。

COUNT 为什么这么贵？因为条件里没有时间范围，它只能把覆盖索引从头到尾扫一遍，逐项判断余额和定价，一条都不能跳。这正是第二节说的「索引全扫」：扫描量和索引总行数成正比，和结果有多少条无关。10.9 万行要 90ms，1000 万行按比例就是秒级。

关于这组数字的边界，也要说清楚：

- 测试集是十万级，不是千万级。7ms 这部分的扫描量由反转后的 offset 决定，在千万级上同样只扫很少的索引项，可以外推；COUNT 那部分随总行数线性增长，不能外推成 90ms。
- 千万级上真正需要担心的，是不填时间条件时的 COUNT，以及正中间页码的取数。

---

## 七、剩下的 COUNT 怎么办

几种做法，按代价从低到高：

| 做法 | 适用场景 | 代价 |
| :--- | :--- | :--- |
| 保留精确 COUNT | 低并发内部系统，能接受秒级 | 千万级不带时间条件时会慢 |
| 按查询条件缓存 COUNT | 数据在一段时间内不变 | 需要在数据更新时让缓存失效 |
| 用 `EXPLAIN` 的估算行数，显示「约 xx 万条」 | 用户只关心大概有多少 | 估算误差可能很大 |
| 不显示总数，只给「下一页」 | 公网高并发 | 产品交互要改，不能跳尾页 |

对我们这个场景，第二种几乎是量身定做的：数据每天夜里同步一次，白天一整天不会变，同样条件的 COUNT 结果整天都一样。把「条件 + 当天同步批次号」作为缓存 key，第一次查完存起来，后面同一个条件翻页、跳尾页都直接用缓存；夜里同步完成后批次号变了，缓存自然失效。

这一步我们还没有做，因为 20 个人用、90ms 的 COUNT 目前不构成问题。但如果后续数据量真的到了千万级、COUNT 到了秒级，这是成本最低的下一步，比引入 ES 便宜得多。

---

## 八、上线前的三道防线

优化只覆盖了预想到的查询。十几个条件可以自由组合，总有组合会落到索引外面。所以还需要几道兜底：

### 1. 给每条查询加执行时间上限

MySQL 5.7.8 以后支持在 SELECT 上加优化器提示，超时直接中止：

```xml
<sql id="timeoutHint">
    <if test="q.maxExecutionMs != null">/*+ MAX_EXECUTION_TIME(${q.maxExecutionMs}) */</if>
</sql>
```

这个提示只接受字面量，所以这里用了 `${}`。参数是后端配置的 Integer，不来自用户输入，不存在注入问题。超时异常在 Service 层统一转成「查询超时，请缩小查询条件」，而不是让一条慢 SQL 占着连接。

### 2. 慢查询日志记录条件组合

每次查询结束记录耗时和「用到了哪些条件」（只记条件名，不记证件号、手机号的值），超过阈值再打一条 WARN。上线一段时间后按条件组合统计，就知道哪些组合常用又慢，该给谁补索引。索引应该由真实的查询分布决定，而不是靠猜。

### 3. 导出不用 offset，用游标

导出要把几万条结果全拉出来，如果按页循环 `LIMIT offset, 5000`，越往后越慢。导出改成按上一批最后一行的 `(lend_time, id)` 往后取：

```sql
SELECT * FROM t_loan_iou
WHERE <筛选条件>
  AND (lend_time < #{afterLendTime}
       OR (lend_time = #{afterLendTime} AND id < #{afterId}))
ORDER BY lend_time DESC, id DESC
LIMIT 5001;
```

每一批都从索引上直接定位起点，不需要跳过任何行，第 1 批和第 100 批一样快。多取 1 条用来判断还有没有下一批，省掉一次 COUNT。

游标分页没有「跳到第 N 页」的能力，所以它适合导出、下拉加载，不适合需要页码的列表。这也是为什么列表和导出用了两套写法。

---

## 九、这套方案什么时候会失效

把它的前提列清楚，比宣传它有多快更重要：

1. 并发上去就不行。它消灭的是随机 IO，不是 CPU 扫描。上一篇算过：一次 90ms 的索引扫描，每秒 2 次没问题，每秒 500 次就能把 16 核打满。
2. 筛选列不在索引里就退回原形。新加的条件如果没挂进覆盖索引，子查询就要回表判断，延迟关联的优势没了。
3. 索引列不能无限加。每加一列，索引项变宽、写入变慢、DDL 变重。超过十来列就该考虑是不是找错了工具。
4. 数据频繁变化时，尾页反转的前提不成立。COUNT 和取数之间的错位、宽索引的写入成本，都会冒出来。
5. 优化器可能选错索引。条件组合多了，统计信息不准时，优化器偶尔会选一个看起来便宜、实际很慢的索引。这种情况靠第八节的慢查询日志发现，再针对性处理。

---

## ✅ Checklist：给千万级列表做深分页优化前过一遍

- [ ] **排序是不是带了唯一列？** `ORDER BY time` 不带 id，同一时刻的多条记录顺序不稳定，翻页会重复或漏行。
- [ ] **索引里 id 的位置和排序一致吗？** 排序是 `(time, id)`，索引前两列就应该是 `(time, id)`，否则 EXPLAIN 里会出现 `Using filesort`。
- [ ] **覆盖索引覆盖了子查询用到的所有列吗？** 子查询 Extra 里要看到 `Using index`，看不到就说明在回表。
- [ ] **EXPLAIN 是在接近全量的数据上看的吗？** 小表上优化器的选择没有参考价值。
- [ ] **COUNT 是不是已经成了大头？** 优化完先拆开看耗时，再决定下一步优化什么。
- [ ] **COUNT 和取数之间数据会不会变？** 会变，就不能简单用 total 换算反转后的 offset。
- [ ] **有没有执行时间上限和慢查询日志？** 自由组合的条件，总有组合落在索引外面。
- [ ] **导出是不是还在用 offset 循环？** 导出改游标，别让最后一批拖垮数据库。

---

## 写在最后

回头看这次优化，三个手段各解决了一个具体的问题：

- 宽覆盖索引解决的是「要回表才能判断条件」；
- 延迟关联解决的是「为了跳过的行也去回表」；
- 尾页反转解决的是「为了最后几行从头数一遍」。

但每一步也都留下了一个没解决的问题，最后剩下的是那个扫完整个索引的 COUNT。优化做到这里，再往下就不是改 SQL 能解决的了：要么利用业务特征（T-1 数据一整天不变）去缓存它，要么从产品上把它拿掉，要么换一种能在内存里直接算交集的存储。

下一篇讲详情层：几十 KB 的决策大报文为什么从 MySQL 里搬出去，分布式 KV 底层的 LSM-Tree 怎么支撑 TB 级小对象的亚毫秒点查，以及 Key-List 这类列表模型适合什么、不适合什么。

---

## 系列目录

1. [第 1 篇 · 存储架构全景与选型边界推演（总纲）](/blog/storage-architecture-overview-and-selection/)
2. **第 2 篇 · 单机 MySQL 千万级台账任意翻页实战（本篇）**
3. 第 3 篇 · 分布式 KV 与 Key-List 列表引擎深度拆解（待更）
4. 第 4 篇 · 突破单机极限：TiDB 全局索引与 ES 位图检索实战（待更）
