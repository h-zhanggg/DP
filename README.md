
Snap + Noise
- ✔ 可以降低 re-id 风险
- ✖ 不等于 DP

Geo-indistinguishability
- ✔ 单点 DP
- ✖ 不解决 trajectory composition

User-level DP
- ✔ 真正保护
- ✖ 只能 synthetic 或 query release



# Snap + Noise
## 不是 formal DP，是 risk-reduction heuristic

1. 删除原 agent_id
2. 重新随机生成 ID
3. Snap + Noise，发布 （有noise的）lat/long
3.1 每个 user 限制最多 7 天数据?
4. 能推断出 用户在这个lat/long 这个时间段的 activity type
5. 整个方法耗时少


