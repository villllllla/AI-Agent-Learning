**HNSW (Hierarchical Navigable Small World)** 是目前近似最近邻（ANN）检索领域综合性能（召回率 vs QPS）最强、应用最广的图基（Graph-based）算法。
#### 1. 核心理论基石
HNSW 的架构融合了两种经典数据结构的理念：
- **NSW (Navigable Small World)**：基于六度分隔理论的“小世界图”。在平面内构建贪婪路由网络，长边保证全局跨越跳跃，短边保证局部精确搜索。
- **Skip List (跳表)**：通过构建多层级的链表结构，上层稀疏，下层稠密，实现 $O(\log N)$ 复杂度的快速定位。
#### 2. 算法机制拆解
- **层级拓扑结构**：图被划分为第 $0$ 层到第 $L$ 层。第 0 层包含所有节点数据（最稠密），越往上节点越少（按指数衰减概率分布）。
- **路由机制 (Routing)**：给定查询点 $q$，从最高层的唯一入口节点开始，进行**贪婪搜索 (Greedy Search)**。在当前层找到距离 $q$ 最近的局部最优节点后，不直接结束，而是**以该节点作为入口，垂直降落到下一层**继续贪婪搜索，直到抵达第 0 层。
- **启发式连边 (Heuristic Edge Selection)**：在插入新节点时，HNSW 并不盲目连接空间上绝对最近的 $M$ 个邻居，而是采用启发式策略。它会检查候选邻居之间的距离，如果候选邻居相互之间距离比它们到查询点的距离还要近，说明空间冗余，HNSW 会放弃部分近邻，转而连接更远但能提供新方向的节点，从而有效避免路由陷入局部死胡同。
#### 3. HNSW 核心操作伪代码
以下伪代码高度概括了 HNSW 最核心的 **K-NN 搜索过程 (K-Nearest Neighbor Search)**：
```Plaintext
Algorithm: K-NN-Search(q, K, ep, ef)
Input:
  q:  查询向量 (Query point)
  K:  需要返回的最近邻数量 (Number of nearest neighbors to return)
  ep: 最高层的入口点 (Enter point of the top layer)
  ef: 动态候选池大小，控制搜索宽度 (Size of the dynamic candidate list)

Output: 距离 q 最近的 K 个节点

// 阶段 1: 从顶层 L 逐层快速降落至第 1 层 (找第 0 层的入口)
curr_node = ep
for layer_c = L down to 1:
    changed = true
    while changed:
        changed = false
        // 遍历当前节点在 layer_c 的所有邻居
        for neighbor in neighbors(curr_node, layer_c):
            if distance(neighbor, q) < distance(curr_node, q):
                curr_node = neighbor
                changed = true
    // 将找到的局部最优解作为下一层的入口
    enter_point = curr_node 

// 阶段 2: 在第 0 层进行精细化束搜索 (Beam Search)
candidate_pool = priority_queue_min() // 待扩展节点，按距离从小到大排
result_set = priority_queue_max()     // 当前找到的近邻，按距离从大到小排

candidate_pool.insert(enter_point)
result_set.insert(enter_point)

while candidate_pool is not empty:
    c = candidate_pool.pop()
    
    // 剪枝：如果当前要扩展的节点距离，比 result_set 里最远的节点还要远，终止扩展
    if distance(c, q) > result_set.max_distance():
        break 

    for neighbor in neighbors(c, layer=0):
        if neighbor not in visited_set:
            visited_set.add(neighbor)
            dist = distance(neighbor, q)
            
            if result_set.size() < ef or dist < result_set.max_distance():
                candidate_pool.insert(neighbor)
                result_set.insert(neighbor)
                
                // 保证候选池大小不超过 ef
                if result_set.size() > ef:
                    result_set.pop_max()

// 阶段 3: 返回最终结果
return result_set.get_top(K)
```

