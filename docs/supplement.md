# 补充模板手册

> 定位：只收录另一份已打印模板的补充内容，以及实际使用中发现不适配或容易出错的模板。代码合并前必须结合题目条件重新确认。

## 目录

- [网络流](#网络流)
- [`__int128` 输入输出](#__int128-输入输出)
- [最小路径覆盖](#最小路径覆盖)
- [最小费用最大流](#最小费用最大流)
- [矩阵方程 `AB=C` 的高斯-约旦消元](#矩阵方程-abc-的高斯-约旦消元)
- [数位 DP：统计区间最优数字频次](#数位-dp统计区间最优数字频次)
- [线段树优化建图](#线段树优化建图)

## 网络流

### 数组版 Dinic

**适用条件**：有向容量网络，节点编号为 `0..n`，边容量为非负整数。调用 `init(n)` 后再加边。

**复杂度**：一般为 `O(V^2 E)`；二分图等常见场景通常更快。数组大小由 `MAXN`、`MAXM` 决定，必须按实际点数和有向边（含反边）估算。

**边界条件**：`s == t`、容量溢出、节点编号超过 `MAXN-1` 都不应直接调用；`MAXM` 统计正反边总数。递归 DFS 可能触及系统栈限制。

**直属依赖**：`<bits/stdc++.h>`、`queue`、`min`；需要在外部提供 `MAXN/MAXM` 足够大的数组空间。

**注意事项**：模板中 `visL/visR` 与 Dinic 无关，可删除；`add` 每调用一次占用两个边槽。下面代码是原数组版实现，适合作为轻量补充，不替代队伍已有封装版 Flow。

```cpp
const int MAXN = 200010, MAXM = 800010, INF = 1e9;
int head[MAXN], to[MAXM], nxt[MAXM], cap[MAXM], cnt;
int d[MAXN], cur[MAXN];
void init(int n) { cnt = 0; for (int i = 0; i <= n; ++i) head[i] = -1; }
void add(int u, int v, int c) {
    to[cnt] = v; cap[cnt] = c; nxt[cnt] = head[u]; head[u] = cnt++;
    to[cnt] = u; cap[cnt] = 0; nxt[cnt] = head[v]; head[v] = cnt++;
}
bool bfs(int s, int t, int n) {
    for (int i = 0; i <= n; ++i) d[i] = -1;
    queue<int> q; d[s] = 0; q.push(s);
    while (!q.empty()) {
        int u = q.front(); q.pop();
        for (int i = head[u]; i != -1; i = nxt[i])
            if (cap[i] > 0 && d[to[i]] == -1) d[to[i]] = d[u] + 1, q.push(to[i]);
    }
    return d[t] != -1;
}
int dfs(int u, int t, int flow) {
    if (u == t) return flow;
    int res = 0;
    for (int& i = cur[u]; i != -1 && res < flow; i = nxt[i]) {
        int v = to[i];
        if (cap[i] > 0 && d[v] == d[u] + 1) {
            int f = dfs(v, t, min(flow - res, cap[i]));
            cap[i] -= f; cap[i ^ 1] += f; res += f;
        }
    }
    return res;
}
int dinic(int s, int t, int n) {
    int ans = 0;
    while (bfs(s, t, n)) { for (int i = 0; i <= n; ++i) cur[i] = head[i]; ans += dfs(s, t, INF); }
    return ans;
}
```

## `__int128` 输入输出

**适用条件**：需要超过 `long long` 范围、但仍在 `__int128_t` 范围内的整数计算。

**复杂度**：单次读写 `O(位数)`，空间 `O(位数)`。

**边界条件**：最小值取相反数时可能超出 `__int128`；不要把它当作任意精度整数。输出 `0` 需要确保写函数覆盖该情况。

**直属依赖**：`string`、`reverse`、`istream`、`ostream`、`function`；GCC/Clang 扩展，不适用于 MSVC 原生编译器。

```cpp
namespace my128 {
using i128 = __int128_t;
i128 abs(i128 x) { return x >= 0 ? x : -x; }
istream& operator>>(istream& in, i128& x) {
    string s; in >> s; bool neg = !s.empty() && s[0] == '-';
    x = 0; for (int i = neg; i < (int)s.size(); ++i) x = x * 10 + s[i] - '0';
    if (neg) x = -x; return in;
}
ostream& operator<<(ostream& out, i128 x) {
    if (x == 0) return out << '0';
    if (x < 0) out << '-', x = -x;
    string s; while (x) s += char('0' + x % 10), x /= 10;
    reverse(s.begin(), s.end()); return out << s;
}
}
```

## 最小路径覆盖

**适用条件**：有向无环图；将原图拆为左右两份并做二分图最大匹配。此实现依赖队伍已有 `Flow`，且顶点编号为 `1..n`。

**复杂度**：取决于 `Flow`；建图 `O(n+m)`，恢复路径 `O(n+m)`。

**边界条件**：图必须是 DAG；若输入含环，恢复路径可能无限循环。`Flow` 的边结构必须能通过 `flow.h[u]`、`flow.ver[eid].to/w` 访问。

**直属依赖**：队伍 `Flow` 模板（直接依赖）；`vector`。没有该定义时本节代码不可独立编译。

```cpp
// 依赖：Flow flow; 其 add(u,v,cap)、work(S,T)、h、ver[eid].to/w 接口。
struct MinPathCover {
    int n, S, T; Flow flow;
    MinPathCover(int n) : n(n), S(0), T(2 * n + 1), flow(2 * n + 5) {
        for (int i = 1; i <= n; ++i) { flow.add(S, i, 1); flow.add(i + n, T, 1); }
    }
    void add(int u, int v) { flow.add(u, v + n, 1); }
    vector<vector<int>> work() {
        flow.work(S, T); vector<int> nxt(n + 1);
        for (int u = 1; u <= n; ++u) for (int id : flow.h[u]) {
            auto& e = flow.ver[id]; int v = e.to - n;
            if (1 <= v && v <= n && e.w == 0) nxt[u] = v;
        }
        vector<bool> start(n + 1, true); for (int i = 1; i <= n; ++i) if (nxt[i]) start[nxt[i]] = false;
        vector<vector<int>> paths;
        for (int i = 1; i <= n; ++i) if (start[i]) {
            vector<int> path; for (int u = i; u; u = nxt[u]) path.push_back(u); paths.push_back(path);
        }
        return paths;
    }
};
```

## 最小费用最大流

**适用条件**：费用可为负但不存在从源可达的负环；需要同时求流量和费用。该版本是队伍内部数组接口的草稿，使用前应与现有 `Flow`/常量定义核对。

**复杂度**：势能 + Dijkstra 的复杂度约为 `O(F E log V)`；首次 SPFA 可能较慢。

**边界条件**：必须正确初始化 `dot`（所有参与点）；`N`、`M`、`INF`、`PII` 必须由外部提供。费用或总费用可能需要 `long long`。

**直属依赖**：`queue`、`priority_queue`、`vector`、`PII`、`N/M/INF` 常量；无这些定义时不能独立编译。

> 原始版本包含未使用变量和依赖外部符号，暂作为“待核对补充”保留，不标记为稳定模板。合并前请维护者确认势能更新和负费用终止条件。

## 矩阵方程 `AB=C` 的高斯-约旦消元

**适用条件**：实数矩阵 `A(n*m)`、`C(n*p)`，求一个特解 `B(m*p)`；允许欠定系统。

**复杂度**：`O(n*m*(m+p))`，空间 `O(n*(m+p))`。

**边界条件**：浮点误差由 `EPS` 控制；空矩阵、维度不一致或无解返回 `false`。自由变量被置为 `0`，返回的是特解而非通解。

**直属依赖**：`vector<vector<double>>`、`fabs`、`swap`；需要 `<cmath>` 和 `<vector>`。

## 数位 DP：统计区间最优数字频次

**适用条件**：统计 `0..x` 中每个数位出现次数的最大值之和；代码使用全局 `num` 和记忆化 map。

**复杂度**：状态数受 `pos` 与计数向量影响，当前 `map<vector<int>, int>` 常数较大；不适合无上界的大范围重复调用。

**边界条件**：当前 `get(x <= 0)` 直接返回 `x`，只适用于题目明确允许该约定的场景；前导零由 `zero` 区分。计数结果可能超过 `int`。

**直属依赖**：`vector`、`map`、`max_element`；全局 `num`、`mp` 和 `dfs` 必须保持一致。

## 线段树优化建图

**适用条件**：需要点到区间、区间到点或区间到区间的批量连边；节点编号为 `1..n`，边权为 `long long`。

**复杂度**：建树 `O(n)`；每次点/区间操作 `O(log n)` 个线段树节点，区间到区间使用一个虚拟点。

**边界条件**：`n` 必须为正；数组预估 `n * 5 + q` 和 `q * 40` 不是严格证明，操作很多或区间拆分密集时需增大容量。这里是单向边图，不自动添加反边。

**直属依赖**：`vector`、`long long`；调用方需自行接入最短路/最小割等图算法。

> 这是本手册中依赖最复杂的模板之一。它直接依赖“物理点”，再依赖入树、出树和区间操作产生的虚拟点；不要把线段树内部编号当作原图点编号。

