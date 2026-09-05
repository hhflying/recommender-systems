# 推荐工程实践（1）：Cache 设计，支撑百毫秒级推荐的基石

全文章节结构如下，大家可以挑选自己感兴趣部分阅读：

1. 为什么要有 Cache ：开篇、概述。
2. Cache 的实现方案：对比 Redis 精确匹配和自研相似度匹配两种方案。
3. 自研 Cache 的设计：介绍相似度匹配原理（MinHash + LSH分桶）、实际工程设计（含伪代码）。
4. 实际案例：我实际参与的一个工程案例，以及当时的设计分析。
5. 总结

文中有部分公式的数学原理没有展开，大家如果感兴趣可以查阅相关资料，或者直接跳过公式部分，不影响阅读。

## 1. 为什么要有 Cache

推荐系统在线接口需要平均 100 毫秒返回，如果请求链路都经由业务模块实时计算，取画像/特征、召回、排序、重排，这些步骤叠加后很容易超时。所以在链路中需要增加缓存（Cache）模块，把短时间重复或者相似的请求由 Cache 模块承接。

> 注：本文中所述接口平均 100 毫秒返回只是示例，不同的系统对接口响应时间要求不完全一样，但基本都是百毫秒级别。

整个流程参考下图，该图首次出现在《推荐系统架构》的第10篇《[在线系统总览](../architecture/10-online.md)》中。下面流程描述有一些概念（例如 Recall Manager）也在这篇文章有介绍，如果之前没见过可以查阅。

![推荐系统在线流程|280](../../assets/architecture/10_engine_flow.png)

在该流程中，请求到达服务端之后，会首先尝试取缓存，此缓存即最终结果缓存。在推荐系统中，多个节点可以设置缓存，包括：

**结果缓存**

Key：用户 ID；Value：最终推荐计算结果 Content ID 列表。这类缓存一般部署在整条链路的最末端，适合用户高频刷新的场景。（上图中的 Cache Hit 就是指是否命中结果缓存）

**召回缓存**

Key：用户兴趣标签（兴趣向量）；Value：根据用户兴趣召回的 Content ID 列表。这一类缓存一般在召回（Recall）阶段，由 Recall Manager 管理，可以缓解召回源的高频查询压力。

除了上面说到两类，其他的还可能有：

- 排序缓存。Key：用户特征列表加上待打分的 Content ID 集合；Value：Content ID 集合和对应的打分分数。这一类的缓存主要是为了解决同一批内容对同一用户的重复打分问题。

- Feature 缓存。把热门的 User Feature、Content Feature 缓存起来加速读取，Key 为 User ID 或者 Content ID。这部分一般在 Feature Server 内部实现。

本文为了简化，挑选**结果缓存**和**召回缓存**为例，讲解具体的工程实现方案。其他部分的缓存实现方案和这两类大同小异。

## 2. Cache 的实现方案

对于结果缓存，召回缓存，最基础通用的方案是采用 Redis，开发成本低，适用于大部分的场景。另外一种是自研 Cache 方案，优点是命中率高、业务定制化能力强，可以适配很多兴趣变化场景。在业界工程化中，两种方案一般都是配套使用。
### 2.1 通用方案：Redis 缓存

Redis 是业界最成熟的缓存体系之一，在推荐系统中广泛被使用。结果缓存、召回缓存，都可以直接用 Redis 来实现，是中小团队实现缓存的最快落地方案。

结果缓存是直接面向用户的，为了解决用户短时间重复刷新问题设立。所以结果缓存的 Key 一般设计带上用户 ID 做标识，形如：`result:{scene_id}:{uid}`，这样可以通过用户 ID 直接定位到结果。Value 一般为用户最终推荐内容 ID 列表，可以采用 JSON 或者 Protobuf 等序列化方式存储。中小规模场景用 JSON 会更方便调试；大规模下用 Protobuf，和 JSON 相比能节省 30% 以上存储空间，同时也能减少编码解码的时间消耗。结果缓存一般会设置 5 ~ 10 分钟的过期时间。过期时间设置太短会让缓存命中率下降，性能优化效果不佳；过期时间太长不能响应用户实时兴趣，降低体验。

召回缓存是为了解决兴趣标签、向量检索的重复计算和查询压力，Key 一般包含用户兴趣/向量 Hash 之后的值，例如：`recall:{scene_id}:{interest_hash}`。Value 是多路召回聚合之后的候选内容 ID 列表。召回缓存因为 Key 是用户兴趣，过期时间可以设置比结果缓存长一些，一般设置在 10 ~ 15 分钟。

如果采用 Redis 做缓存，可以在推荐引擎中直接添加相应代码来读写。也可以把 Redis 做个简单的封装，把 Key、Value 组装这些逻辑集中起来，推荐引擎直接调用封装之后的服务接口就行。这两种方式 Key 都是精确匹配；在召回缓存场景中，如果兴趣标签有任意微小变化，Key 都不一样，从而导致缓存不命中，达不到性能优化目的。

### 2.2 自研 Cache 方案

自研 Cache 相对于通用 Redis 方案，最大的改进在于 Key 支持相似度匹配，同时也可以在存储序列化、淘汰策略、读写性能上做定制化优化。

首先自研 Cache 存储可以有本地缓存和分布式两层，本地缓存采用内存结构，分布式可以采用已有的 Redis 等方案。对于本地缓存可定制化各类淘汰和加载策略，例如频繁访问的热点内容（非常高频使用的高频用户的结果缓存、新用户冷启动的召回缓存等），存放在本地实现更好的加速。其次在 Value 的序列化上可以采用更为定制化的二进制，比通用的 Protobuf 更节省存储空间。

另一项也是最核心的改进在于支持相似度匹配，适配推荐场景中，相似兴趣的推荐结果也类似的情况。大概的做法就是通过算法把用户高维兴趣特征转换为可比对的缓存标识，然后通过相似度算法来判断两个 Key 是否类似。这样就算用户实时兴趣有了小变化，通过相似度匹配也能命中缓存；此外对于一些具有长尾兴趣的用户，在一定相似度阈值内也能利用缓存加速。

在业界大的推荐平台中，很多系统都会采用这样的自研 Cache 方案，同时支持精确 Key 匹配和相似度匹配。这样既能支持热点用户结果缓存的精确型匹配，保证原有的推荐体验；也能支持相似用户、新用户冷启动用户、具有长尾兴趣的用户，通过相似匹配来优化系统整体性能。

## 3. 自研 Cache 的设计

推荐系统和数据库最大的区别在于：推荐结果不存在唯一正确答案。兴趣高度相似的两个用户，推荐结果通常也高度相似。因此，我们不需要完全相同的 Key，而是寻找“足够相似”的缓存。

自研 Cache 的核心点是通过用户兴趣的相似度来判断是否命中，下面我们来看看这个核心功能的大概原理和工程实现。
### 3.1 相似度匹配

用户兴趣可以看成一个标签集合，例如：{科技, 娱乐, 体育}（每个元素也可以是带权重的标签）。两个用户兴趣集合的相似度，可以通过 Jaccard 相似度来计算，公式是
$$
J(A,B)=\frac{|A \bigcap B|}{|A \bigcup B|}
$$

也即是同时出现在两个集合中的元素个数和两个集合并集元素个数比值。例如，假设 A = {a, b, c, d, e}, B = {c, e, f, h}，Jaccard 相似度计算：
$$
J(A,B) = \frac{|A \bigcap B|}{|A \bigcup B|} = \frac{|\{c,e\}|}{|\{a, b, c, d, e, f, h\}|} = \frac{2}{7}
$$
所以这两个集合相似度可以表示 2/7 ≈ 0.2857。根据公式，Jaccard 相似度的值域为 0 ~ 1。

当 A、B 两个集合比较大时，计算 Jaccard 相似度非常耗资源，所以业界一般用 MinHash 来计算。

### 3.2 MinHash 算法

选择一个均匀分布的随机 Hash 函数，对集合 A 中每个元素计算 Hash 值，然后取最小值，记为 hmin(A)，作为其 MinHash 签名；对集合 B 中每个元素做同样计算，得到 hmin(B)。基于 Hash 的均匀随机性，hmin(A) 和 hmin(B) 相等的概率近似等于 A、B 集合的 Jaccard 相似度。（具体的数学原理此处不过多展开）

如果只选取一个 Hash 函数，计算 hmin(A) 和 hmin(B)，结果是 0 或者 1，不能得到 0 ~ 1 区间的相似度。所以，我们可以选择 k 个不同的 Hash 函数，分别对 A、B 计算 MinHash 各获得一个 k 维向量。比较这两个向量中对应位置元素相等的个数 y 和 k 的比值（y/k），可以作为 A、B 两个集合 Jaccard 相似度的估计。这个估计是无偏的，并且可以通过增加 Hash 函数数量 k 来减少估算方差。

在实际的工程实现中，通常取 k = 128；但此处存在一个工程难点：很难找到 128 个不同的 Hash 函数。在实际的实现中，一般采用采用折中方案，用一个基础 Hash 函数（例如 MurmurHash），然后通过 k 次不同加盐计算获得 k 维 MinHash 向量。同一个基础 Hash 函数加不同盐值生成的 Hash 值可视为相互独立，满足前面说的无偏估计要求。 

解决了两个集合相似度快速计算问题，回到我们 Cache 的主题，是为了从缓存中找到相似的记录。若没有任何优化加速，每一个用户请求进来，都需要把当前画像和缓存中所有记录全部比较一遍，扫描的时间复杂度是 O(N)（N 为缓存的记录数）。这个时间复杂度在线上不能接受，所以需要其他策略来解决这个问题，LSH 分桶就是其中最常用的一种。

### 3.3 LSH 分桶

LSH （Locality Sensitive Hashing，局部敏感哈希）分桶的算法，是把 k 维 MinHash 向量分成不同的组（Band，也可以称作桶，），每一个 Band 内所有 MinHash 签名连接起来构成一个 Key。查询时，比较所有 Band Key，只要有任意一个 Key 命中，则表示两个 k 维 MinHash 向量有可能相似，然后再进行这个桶内实际 MinHash 向量比较。用这种分桶方式，整个 O(N) 的扫描，变成了 k\*O(m) 的复杂度，其中 m 是一个桶内最大 MinHash 向量个数。在工程实现中，k 是个固定的常数数值（例如 16），所以时间复杂度可简化为 O(m)。

Band 的数量划分一般要考虑 Band 内部 MinHash 签名的个数。如果单 Band 内 MinHash 签名数量过多，可能存在微小变化从而导致 Band Key 变化的问题，最后导致漏命中、缓存命中率下降。如果单 Band 内 MinHash 签名数量过少，那样 MinHash 签名更容易相似，从而导致一个 Band 内列表过长，也即是前面所说的 m 值过大，计算过程变得更复杂。对于 k = 128 维的 MinHash 向量，可以选取 16 个 Band，每个 Band 内 8 个 MinHash 签名。

如果 b 表示 Band 数量，r 表示一个 Band 内部 MinHash 签名的数量，s 表示两个集合的 Jaccard 相似度。那么这两个向量至少有一个 Band 相同的概率为：
$$
P = 1-(1-s^r) ^b
$$
（具体数学原理可以查询相关资料：LSH 标准概率公式，此处不过多展开）。当 b=16、r=8、s=0.9 时，P=99.96%，代表这样两个向量肯定能被某一个分桶检索到。如果 s=0.8，P=94.7%，也在 95% 的黄金拐点附近。这也是这组参数目前很多系统使用的原因之一。

### 3.4 Cache 流程

根据前面的 MinHash 和 LSH 分桶原理讲解，我们可以设计 Cache 的结构如下：

- Band Key 索引：Key 为 Band Key，Value 为 128 维 MinHash 向量 列表，因为多个不同 MinHash 向量可能某个 Band 都一样。
- 业务数据缓存：Key 为 MinHash 向量，Value 为 MinHash 向量本身加上召回列表。

带 Cache 的执行流程大概如下：

- 用户请求进来，根据用户 ID 获取用户画像，经过处理形成兴趣向量。
- 兴趣向量经过 MinHash 计算，获得 128 维 MinHash 向量。然后根据 LSH 分桶，获得 16 个 LSH Band Key。
- 从缓存 Band Key 索引并行查询 16 个 Band Key，如果有任一命中，进行当前 MinHash 向量和索引中 MinHash 向量进行 Jaccard 相似度计算，找到符合阈值且最优的那个。然后根据符合条件的 MinHash 从业务数据缓存中获取数据返回。如果没有任何命中或相似度不满足阈值，继续后续流程。
- 进行实时的召回计算，获取召回列表。
- 写入缓存操作：把 MinHash 向量作为 Key，MinHash + 召回列表作为 Value 写入业务数据缓存；然后往 16 个 Band Key 对应 Value 列表末尾附加当前 MinHash 向量。
- 结果返回。

### 3.5 核心伪代码


```java

private static final double SIMILARITY_THRESHOLD = 0.9;
private static final int MINHASH_SIGN_LEN = 128;
private static final int LSH_BAND_COUNT = 16;
private static final int LSH_UNIT_PER_BAND = MINHASH_SIGN_LEN / LSH_BAND_COUNT;
private static final int CACHE_TTL_SECONDS = 600;

public List<Long> queryRecommendWithLshCache(Long uid, Set<String> userInterest) {
    // 1. 构建用户实时128维MinHash签名、16个LSH Band Key
    long[] currentSign = buildUserMinHashSign(userInterest);
    Set<String> bandKeys = buildLshBandKeys(currentSign);

    // 2. 批量查询所有关联索引桶，获取候选数据Key
    List<IndexEntity> candidateKeys = cacheCluster.batchGetIndexEntity(bandKeys);
    
    // 3. 计算 Jaccard 相似度
    IndexEntity bestMatchEntity = null;
    double maxSimilarity = 0.0;
    
    for (IndexEntity entity : candidateKeys) {
        double similarity = calcJaccardSimilarity(currentSign, entity.getMinHashSign());
        if (similarity >= SIMILARITY_THRESHOLD && similarity > maxSimilarity) {
            maxSimilarity = similarity;
            bestMatchEntity = entity;
        }
    }
    
    if (bestMatchEntity != null) {
	    CacheEntity cacheEntity = cacheCluster.getCacheEntity(bestMatchEntity.getMinHashSign());
	    if (cacheEntity != null) {
		    return cacheEntity.getItemIdList();
	    }
    }
    
    // 未命中缓存
    List<Long> itemList = doRealTimeRecommend(uid, userInterest);
    
    // 写入缓存
    CacheEntity cacheEntity = new CacheEntity(currentSign, itemList);
    cacheCluster.setDataCache(currentSign, cacheEntity, CACHE_TTL_SECONDS);
    
    for (String bandKey : bandKeys) {
	    cacheCluster.appendIndexEntity(bandKey, currentSign);
    }
}

/**
 * 生成 MinHash 向量
 */
public long[] buildUserMinHashSign(Set<String> interestTags) {
    long[] minHashSign = new long[MINHASH_SIGN_LEN];
    Arrays.fill(minHashSign, Long.MAX_VALUE);
    // 多组加盐哈希函数，模拟独立哈希，保证签名离散度
    for (String tag : interestTags) {
        for (int idx = 0; idx < MINHASH_SIGN_LEN; idx++) {
            long currHash = murmurHash(tag + "_salt_" + idx);
            if (currHash < minHashSign[idx]) {
                minHashSign[idx] = currHash;
            }
        }
    }
    return minHashSign;
}

/**
 * LSH分桶：根据MinHash签名计算归属桶Key
 */
public Set<String> buildLshBandKeys(long[] minHashSign) {
    Set<String> bandKeySet = new HashSet<>(LSH_BAND_COUNT);
    for (int bandIdx = 0; bandIdx < LSH_BAND_COUNT; bandIdx++) {
        int start = bandIdx * LSH_UNIT_PER_BAND;
        int end = start + LSH_UNIT_PER_BAND;
        long[] bandSubSign = Arrays.copyOfRange(minHashSign, start, end);
        String bandKey = LSH_BUCKET_INDEX_PREFIX + bandIdx + "_" + Arrays.hashCode(bandSubSign);
        bandKeySet.add(bandKey);
    }
    return bandKeySet;
}

```

## 4. 实际案例

以下是我之前经历过的一个实际案例相关数据，以结果缓存和召回列表缓存为例。

提前说明一下，以下案例来自我参与过的实际推荐系统项目。文中的系统架构、设计思路、上线结果都来自真实工程实践。但由于时间较久，部分原始测试记录已经缺失，文中少量容量估算、用户规模等数据根据当时保存的结果进行了反推和整理，数值可能存在小幅误差，但不影响整体设计思路和工程结论。

### 4.1 两级缓存方案

系统设置两级缓存，第一级为 L1 缓存，保存用户最终排序结果；第二级 L2 缓存，保存召回阶段列表。

- L1 缓存，Key 为 User ID，Value 为推荐计算出来的 200 个 Item ID 以及对应打分，分数归一化到 0 ~ 1 区间。
- L2 缓存，主数据 Key 为 MinHash 指纹，Value 为根据该画像计算出来的召回结果列表（1000条），字段：Item ID、召回源、召回源内部打分。此外还有 LSH Band Key 索引，Key 为 LSH Band Key，Value 是对应的 MinHash 列表。

整体流程大概为：

1. 用户请求进来，推荐引擎根据 User ID 查询 L1 缓存，如果查到缓存并且数据还没消耗完，从缓存直接返回；如果没命中缓存，往后继续常规推荐流程，并把返回结果写入缓存；
2. 根据 User ID 获取用户画像（用户兴趣列表）；
3. Recall Manager 根据用户画像指纹（MinHash）查询 L2 缓存，如果相似度超过阈值，则命中缓存，取缓存结果进行后续排序流程；如果没命中缓存，从多路召回中获取召回数据，写入缓存；
4. 召回候选集进行精排打分、重排等后续业务逻辑，结果返回给用户。

所以此处 L1 缓存是用户级缓存，L2 缓存是兴趣级缓存，多用户（有相似兴趣的人）共享。
### 4.2 离线预估

#### 4.2.1 L1 用户结果缓存

**TTL 和命中率** 

离线用过往数据进行实际测评，得到缓存 TTL 和命中率关系：

| TTL    | Hit Rate |
| ------ | -------- |
| 60 s   | 0.11     |
| 300 s  | 0.28     |
| 600 s  | 0.36     |
| 1200 s | 0.44     |

在设计时一般采用 TTL 300 秒（5 分钟），兼顾系统效率和用户体验。TTL 300 秒时，命中率大概为 0.28。

**容量估算**

一个用户结果缓存中有 200 个内容 ID 和相应的排序打分（0 ~ 1 区间），内容 ID 用 uint32 4个字节表示。排序分数原始为 0 ~ 1 区间，定点量化：`round(score x 65536)`，最小分辨率 ≈ 0.00001526。所以一个 Item 存储空间是 item_id (4B) + score (2B) = 6B，一条记录 Value 裸二进制存储空间为：6B x 200 = 1200B，加上 Key 和 Redis 本身元数据开销，单条记录占用内存大概为 1.25KB。

经过历史数据测算，高峰时期 5 分钟内活跃用户数量不超过 100万（日活 3000万，高峰时期一般为晚上 20:00 ~ 22:00），考虑突发热点，容量按双倍即 200 万估算，总的 Redis 内存消耗为 1.25KB x 200万 ≈ 2.39GB。算上内存碎片、主从复制缓冲，冗余系数取 1.25，总容量预估为 2.39GB x 1.25 ≈ 2.98GB。

#### 4.2.2 L2 召回列表缓存

**相似度和命中率**

L2 缓存采用兴趣相似来命中，根据历史数据计算，取 TTL = 10 分钟时，得到相似度阈值和命中率关系如下（其他 TTL 数据丢失，只找到上线实验时的这份数据）：

| Sim. Threshold | Hit Rate |
| -------------- | -------- |
| Exact Match    | 0.31     |
| 0.9            | 0.38     |
| 0.8            | 0.40     |
| 0.7            | 0.41     |

新用户或者轻度使用的用户，很容易具备相同的用户画像，因此精确匹配时也能有 31% 的命中率。最后和业务团队一起确定，取相似度 0.9 作为线上实验阈值。

**容量估算**

TTL 取 10 分钟，每条数据保存 1000 Item ID，单条数据 7B (item_id 4B + source 1B + score 2B)，总的裸二进制占用内存：1000 x 7B = 7000B ≈ 6.84KB。额外附带 128 维 MinHash 签名、Redis 元数据开销，附加算 1.25KB。单条完整记录大小：6.84KB + 1.5KB ≈ 8.34KB。

L1 缓存按突发峰值的 200万用户预留，L2 缓存 TTL 为 10 分钟，对应峰值活跃用户按 400万估算。根据当时的一周离线统计结果，并结合历史容量数据估算，可得出其中 60% 为新用户或轻度使用用户（平均 3.5 个用户有相同 MinHash 签名）、30% 中度使用用户（平均 1.8 个用户有相同 MinHash 签名）、10% 重度使用用户（兴趣基本不重合）。独立的 MinHash 条目估算：400万 x 60% / 3.5 + 400万 x 30% / 1.8 + 400万 x 10% / 1 = 175.2 万。

基础内存占用：175.2万 x 8.34KB ≈ 13.93GB。叠加 LSH Band 索引、内存碎片、主从复制缓冲等，冗余系数取 1.3，一共消耗存储 = 13.93GB x 1.3 ≈ 18.1GB。

### 4.3 实际上线测试情况

**L1 结果缓存上线**

业务指标（点击率、 停留时长、用户留存等）基本没有变化，和上线之前持平。线上监测到的 L1 命中率在 31% 左右，内存使用量最高没有超过 2.5GB。整体请求响应时间，平均从  198ms 降到 141ms，降幅为 28.3%。

**L2 召回缓存上线**

点击率 CTR 有轻微下降（相对下降比例 3% 左右，绝对百分比下降数量在小数点后第三位），阅读时长、用户留存等指标基本不变。线上监测的 L2 缓存命中率也在 38% 左右，和线下估算基本一致。召回阶段的平均响应时间从 98ms 降到 68ms，降幅为 30.6%。叠加 L1 缓存，整体请求的响应时间，平均从 198ms 降到 108ms，降幅 45.4%。

此处有一个有意思的观察，召回阶段单次请求响应时间从 98ms 降为 68ms，降幅为 30.6%。如果按理论来说，单机吞吐量应该能上涨 44%（Q1:Q2=T2:T1）。但实际压测发现，单机的吞吐量只上涨了20%。分析数据发现，加了 L2 缓存之后，多了两次 Redis 请求，Redis value 的序列化、反序列化消耗了额外的 CPU 资源。然后新增了 MinHash 向量计算、Jaccard 相似度计算等 CPU 密集型操作。这些导致单次请求响应时间变少了，但整体的 CPU 使用量上升了，最后导致压测时，CPU 变成了瓶颈，最后单机的吞吐量只比之前上涨了 20% 左右。

## 5. 总结

本文主要介绍了缓存在推荐系统链路中的作用，以及两种缓存的工程落地方案。简要的总结以下几点：

1. 缓存分层设计，逐级优化链路响应时间。
2. 两类不同的缓存，精准满足不同阶段的业务和性能要求。
3. 缓存的参数选择需要在性能和推荐效果之间权衡。
4. 中小团队直接使用 Redis 方案，大团队自研满足进一步需求。
5. 缓存优化带来的响应时间下降，并不一定等比例转化为吞吐量提升。

整体而言，缓存是推荐系统（也是其他系统）的最基础，最具性价比的优化策略之一。分析好链路中的业务场景，不同场景使用不同优化方案，能达到性能和推荐效果的平衡。


