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
## 代码补录

以下代码来自初始整理文件，保留为待核对补充。它们对应前文的最小费用最大流、矩阵方程、数位 DP 和线段树优化建图章节。

最小费用最大流（dinic算法）

```cpp
struct MCMF {
    int h[N], e[M], f[M], w[M], ne[M], idx;
    int dis[N], cur[N]; // d表示最短距离,cur为当前弧
    int n, m, S, T;
    bool vis[N];
    int height[N]; // 势能函数
    vector<int> dot; // 点数
    MCMF(int n, int S, int T) {
        this->n = n;
        this->S = S;
        this->T = T;
        idx = 0;
        memset(h, -1, sizeof h);
        memset(height, 0, sizeof height);
        memset(vis, 0, sizeof vis);
    }
    void add(int a, int b, int c, int d) {
        e[idx] = b, f[idx] = c, w[idx] = d, ne[idx] = h[a], h[a] = idx++;
        e[idx] = a, f[idx] = 0, w[idx] = -d, ne[idx] = h[b], h[b] = idx++;
    }
    bool spfa() {
        for (auto& i : dot) height[i] = INF;
        height[S] = 0;
        queue<int> q;
        q.push(S);
        vis[S] = 1;
        while (!q.empty()) {
            int u = q.front();
            q.pop();
            vis[u] = 0;
            for (int i = h[u]; ~i; i = ne[i]) {
                int v = e[i];
                if (f[i] && height[v] > height[u] + w[i]) {
                    height[v] = height[u] + w[i];
                    if (!vis[v]) q.push(v), vis[v] = 1;
                }
            }
        }
        return height[T] != INF;
    }
    bool dijk() {
        for (auto& i : dot) dis[i] = INF, cur[i] = h[i];
        priority_queue<PII, vector<PII>, greater<>> pq;
        dis[S] = 0;
        pq.emplace(0, S);
        while (!pq.empty()) {
            auto [d, u] = pq.top();
            pq.pop();
            if (dis[u] != d) continue;
            for (int i = h[u]; ~i; i = ne[i]) {
                int v = e[i];
                if (f[i] && dis[v] > d + height[u] - height[v] + w[i]) {
                    dis[v] = d + height[u] - height[v] + w[i];
                    pq.emplace(dis[v], v);
                }
            }
        }
        return dis[T] != INF;
    }
    int find(int u, int limit) {
        if (!limit || u == T) return limit;
        int flow = 0;
        vis[u] = 1;
        for (int& i = cur[u]; ~i && flow < limit; i = ne[i]) {
            int v = e[i];
            if (dis[v] == dis[u] + height[u] - height[v] + w[i] && f[i] && !vis[v]) {
                int t = find(v, min(f[i], limit - flow));
                f[i] -= t, f[i ^ 1] += t, flow += t;
                if (flow == limit) break;
            }
        }
        vis[u] = 0;
        return flow;
    }
    void mcmf(int& flow, int& cost) {
        flow = cost = 0;
        int r = 0;
        spfa();
        while (dijk()) {
            if (height[T] >= 0) break;
            int r = find(S, INF);
            flow += r;
            for (auto& i : dot) {
                if (dis[i] != INF) height[i] += dis[i];
            }
            cost += height[T] * r;
        }
    }
};
```
高斯消元法求解矩阵方程`AB=C`，`A`,`C`已知求`B`。
```cpp
const double EPS = 1e-9;
// 求解实数矩阵方程 A * B = C
// A 为 n * m 矩阵, C 为 n * p 矩阵
// 成功找到特解返回 true 并存入 B (m * p)，无解返回 false
bool solveAB_C(const vector<vector<double>>& A, const vector<vector<double>>& C, vector<vector<double>>& B) {
    if (A.empty() || C.empty()) return false;
    int n = A.size(), m = A[0].size(), p = C[0].size();
    // 构造增广矩阵 M = [A | C]
    vector<vector<double>> M(n, vector<double>(m + p, 0.0));
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < m; j++) M[i][j] = A[i][j];
        for (int j = 0; j < p; j++) M[i][m + j] = C[i][j];
    }
    vector<int> pivot(m, -1); // 记录每列主元所在的行
    int r = 0;                // 当前行
    // 高斯-约旦消元 (列主元)
    for (int c = 0; c < m && r < n; c++) {
        // 寻找列主元 (绝对值最大，减小精度误差)
        int mx = r;
        for (int i = r + 1; i < n; i++) {
            if (fabs(M[i][c]) > fabs(M[mx][c])) mx = i;
        }
        if (fabs(M[mx][c]) < EPS) continue; // 自由未知量
        swap(M[r], M[mx]); // 交换行
        pivot[c] = r;
        // 主元所在行归一化
        double div = M[r][c];
        for (int j = c; j < m + p; j++) M[r][j] /= div;
        // 消去其他行的该列
        for (int i = 0; i < n; i++) {
            if (i != r && fabs(M[i][c]) > EPS) {
                double mul = M[i][c];
                for (int j = c; j < m + p; j++) {
                    M[i][j] -= mul * M[r][j];
                }
            }
        }
        r++;
    }
    // 判无解：A 部分全为 0，但 C 部分不为 0
    for (int i = r; i < n; i++) {
        for (int j = 0; j < p; j++) {
            if (fabs(M[i][m + j]) > EPS) return false;
        }
    }
    // 提取特解
    B.assign(m, vector<double>(p, 0.0));
    for (int c = 0; c < m; c++) {
        if (pivot[c] != -1) {
            for (int j = 0; j < p; j++) {
                B[c][j] = M[pivot[c]][m + j];
            }
        }
    }
    return true; // 求解成功
}
```
数位DP模板
下面是计算 $f(x)=x$中出现次数最多的数字的次数，求解$\sum_{i=0}^nf(i)$的程序。
这题的小技巧就是dp可以只记录本质不同的性质。
```cpp
int num[20];
vector<map<vector<int>, int>> mp(20);
int dfs(int pos, vector<int> cnt, bool lim, bool zero) {
    // cerr << pos << ' ' << lim << ' ' << zero << endl;
    int m = *max_element(cnt.begin(), cnt.end());
    if (!pos) return zero ? 0 : m;
    vector<int> p(20);
    for (int j : cnt) p[j]++;
    if (!lim && !zero && mp[pos].count(p)) return mp[pos][p];
    int mx = lim ? num[pos] : 9, res = 0;
    for (int i = 0; i <= mx; i++) {
        if (i == 0 && zero) {
            res += dfs(pos - 1, cnt, lim && i == mx, 1);
        } else {
            cnt[i]++;
            res += dfs(pos - 1, cnt, lim && i == mx, 0);
            cnt[i]--;
        }
    }
    if (!lim && !zero) mp[pos][p] = res;
    return res;
}
int get(int x) {
    if (x <= 0) return x;
    int len = 0;
    while (x) {
        num[++len] = x % 10;
        x /= 10;
    }
    return dfs(len, vector<int>(10, 0), 1, 1);
}
```
线段树优化建图
```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

const ll INF = 1e18;

struct SegGraph {
    int n, tot;
    vector<int> ls, rs;

    // 链式前向星存图 (极致缓存友好)
    int ecnt;
    vector<int> head, to, nxt;
    vector<ll> wt;

    int root_in, root_out;

    /**
     * @param n 真实节点数 (物理节点)
     * @param q 连边操作的总次数 (用于精确预估边数和虚拟节点数，极致优化常数)
     */
    SegGraph(int n, int q) : n(n), tot(n), ecnt(0) {
        // 节点预估：真实点 n + 入树 2n + 出树 2n + 虚拟中转点(每次区间到区间最多产生1个虚拟点) q
        int max_nodes = n * 5 + q + 5;
        // 边数预估：入树出树内部边 4n + q 次操作每次最多下放到 O(log N) 个节点，粗略给常数 40
        int max_edges = n * 4 + q * 40 + 5;

        ls.assign(max_nodes, 0);
        rs.assign(max_nodes, 0);

        head.assign(max_nodes, 0);
        to.assign(max_edges, 0);
        nxt.assign(max_edges, 0);
        wt.assign(max_edges, 0);

        root_in = build_in(1, n);
        root_out = build_out(1, n);
    }

    inline void add_edge(int u, int v, ll w) {
        to[++ecnt] = v;
        wt[ecnt] = w;
        nxt[ecnt] = head[u];
        head[u] = ecnt;
    }

    int build_in(int l, int r) {
        if (l == r) return l;
        int p = ++tot;
        int mid = (l + r) >> 1;
        ls[p] = build_in(l, mid);
        rs[p] = build_in(mid + 1, r);
        add_edge(p, ls[p], 0);
        add_edge(p, rs[p], 0);
        return p;
    }

    int build_out(int l, int r) {
        if (l == r) return l;
        int p = ++tot;
        int mid = (l + r) >> 1;
        ls[p] = build_out(l, mid);
        rs[p] = build_out(mid + 1, r);
        add_edge(ls[p], p, 0);
        add_edge(rs[p], p, 0);
        return p;
    }

    void _p2r(int p, int l, int r, int u, int L, int R, ll w) {
        if (L <= l && r <= R) {
            add_edge(u, p, w);
            return;
        }
        int mid = (l + r) >> 1;
        if (L <= mid) _p2r(ls[p], l, mid, u, L, R, w);
        if (R > mid)  _p2r(rs[p], mid + 1, r, u, L, R, w);
    }

    void _r2p(int p, int l, int r, int L, int R, int v, ll w) {
        if (L <= l && r <= R) {
            add_edge(p, v, w);
            return;
        }
        int mid = (l + r) >> 1;
        if (L <= mid) _r2p(ls[p], l, mid, L, R, v, w);
        if (R > mid)  _r2p(rs[p], mid + 1, r, L, R, v, w);
    }

    // 1. 点向点连边
    inline void add_point_to_point(int u, int v, ll w) { add_edge(u, v, w); }
    // 2. 点向区间连边
    inline void add_point_to_range(int u, int L, int R, ll w) { _p2r(root_in, 1, n, u, L, R, w); }
    // 3. 区间向点连边
    inline void add_range_to_point(int L, int R, int v, ll w) { _r2p(root_out, 1, n, L, R, v, w); }

    // 4. 区间向区间连边 (利用虚拟节点中转)
    inline void add_range_to_range(int L1, int R1, int L2, int R2, ll w) {
        int virtual_node = ++tot;
        add_range_to_point(L1, R1, virtual_node, w);
        add_point_to_range(virtual_node, L2, R2, 0);
    }

    int get_tot_nodes() const { return tot; }
};
```

