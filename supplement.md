

**“publish every column is a must” + 防御 re-identification attack + lat/long 极敏感**，这已经是 mobility privacy 领域的 hardest case 之一。

我们系统性拆解一下。

---

# 一、为什么你现在的 record-level noise ≠ Global DP

你提到：

* 坐标加 Gaussian noise
* 时间 bucket
* Randomized response

问题是：

> 如果只是“对每条记录加噪声”，通常不 automatically 满足 (ε,δ)-DP

因为：

* 你的数据是 **trajectory data**
* 一个 agent_id 对应多个 record
* Global sensitivity 不是 per-row，而是 per-user

在 mobility 数据中：

> 单个用户的 entire trajectory 是一个 high-dimensional object

这就是为什么：

* Apple telemetry 用的是 **local DP**
* Google 的 **RAPPOR** 也是 local DP

Local DP 的优点是：不需要证明 global sensitivity
缺点是：utility 会非常差（尤其是连续变量 lat/long）

---

# 二、你这个数据的 re-identification 风险有多高？

Mobility 数据的经典结论：

* 4 个时空点可以 uniquely identify 95% 用户
* 就算 coarse-grain 到 zip level 也不安全

（这是 mobility privacy 领域的公认结果）

你现在的 dataset 具有：

* 精确 timestamp
* 精确 lat/long
* 持续时间
* activity_type
* trajectory linkage

这是 **极易重识别结构**

尤其：

```
home location + work location = unique
```

---

# 三、如果“必须保留每一列”，正确路线是什么？

⚠️ 关键现实：

> 你不能在保留精确 lat/long 的同时保证强 DP。

你只能：

1. 要么放弃 record-level truth
2. 要么发布 synthetic data
3. 要么使用 user-level DP 机制

下面是专业可行路线。

---

# 四、推荐方案：User-level DP + Synthetic Data

这是目前学术界 publish mobility data 的主流安全方案。

## Step 1：定义 adjacency

必须明确：

> 一个 dataset 与另一个 dataset 的 difference 是“一个 agent 的所有记录”

也就是：

```
D ~ D' differ by one entire trajectory
```

这是 user-level DP。

---

## Step 2：不要对原数据直接加噪

而是：

> 训练一个 DP generative model
> 然后发布 synthetic dataset

可以用：

* DP-GAN
* PATE-GAN
* DP-VAE
* 或 DP-HMM（对轨迹更合适）

---

## Step 3：在生成模型中加入 DP-SGD

使用：

```
DP-SGD (Abadi et al.)
```

控制：

* ε
* δ
* clipping norm
* noise multiplier

这样你可以：

✔ 保留所有 columns
✔ 保留结构
✔ 提供 (ε,δ)-DP 证明
✔ 防御 membership inference attack
✔ 防御 linkage attack

---

# 五、如果你一定要在原数据上处理（不做 synthetic）

那必须是：

## 方案 A：Geo-indistinguishability

参考：

Andrés et al., 2013

对每个 location 加：

```
2D Laplace noise
```

满足：

```
ε-geo-indistinguishability
```

优点：

* 有严格证明
* 专门为 location 设计

缺点：

* 连续 trajectory 下 ε 会快速累积
* 需要 composition accounting

---

## 方案 B：Spatial Cloaking + k-anonymity

例如：

* 把 lat/long 替换成 H3 level 7
* 删除原始坐标
* 保证每个 cell 至少 k users

但你说 “every column must be published”
那你可以：

```
lat_noisy
long_noisy
```

保留列名，但不保留原始值。

---

# 六、真正安全的架构（工业级）

如果我是 principal investigator，我会这么做：

### 1️⃣ 删除原始 agent_id

改成：

```
DP pseudonym
```

避免 deterministic hash（会被 dictionary attack）

---

### 2️⃣ 时间离散化

* 5–15 min bucket
* 随机 shift ±1 bucket

---

### 3️⃣ Geo 处理

选一个：

| 方法             | 安全性           | Utility |
| -------------- | ------------- | ------- |
| Gaussian noise | ❌ 无 formal DP | 高       |
| 2D Laplace     | ✔ 有 DP        | 中       |
| Grid snapping  | 中             | 中       |
| Synthetic      | ✔✔✔           | 高       |

---

### 4️⃣ 加 trajectory-level noise budget accounting

你必须控制：

```
Total ε per user
```

例如：

```
ε_total = 3
per event ε = 0.05
最多 60 events
```

否则 composition 会炸。

---

# 七、Local DP 是否适合你？

如果你真要 local DP：

每条记录都：

```
Perturb locally before aggregation
```

但：

> continuous lat/long + local DP = 几乎 unusable

Apple 之所以能用，是因为：

* 统计是 aggregated
* 不是发布 microdata

你是发布 microdata，这是完全不同场景。

---

# 八、真正推荐的 publish 方案

我给你一个 research-grade 方案：

### Option 1（最安全）

> Publish DP synthetic trajectories

* 训练 DP-HMM
* 控制 ε=2~5
* 发布 synthetic full table
* 附 privacy proof

这是顶会级做法。

---

### Option 2（中等安全）

* 删除真实 lat/long
* 发布：

  * H3 level 7
  * noisy centroid
* 时间 bucket
* user-level DP noise
* publish ε value

---

### Option 3（不推荐）

Per-record Gaussian noise
→ 无 formal DP guarantee
→ 无法 defense linkage attack

---

# 九、你现在最大误区

你现在在想：

> “我对每条 record 加噪声，是不是就 DP 了？”

答案：

❌ 不是
❌ trajectory data 不是 i.i.d. tabular data
❌ sensitivity 是 per-user，不是 per-row

---

# 十、我建议你做的下一步

如果你愿意，我可以帮你：

* 明确定义 adjacency
* 推导 sensitivity
* 设计 ε 分配
* 写一个 publishable DP guarantee section
* 或帮你选最适合 mobility 数据的机制

你是做 mobility research 吗？
如果是 academic publish，我可以直接给你一套可写进论文的方法框架。
