> 跳表（Skip List，全称跳跃表）是用于有序元素序列快速搜索查找的一个数据结构，跳表是一个随机化的数据结构，实质就是一种**可以进行二分查找的有序链表**。跳表在原有的有序链表上面增加了多级索引，通过索引来实现快速查找。跳表不仅能提高搜索性能，同时也可以提高插入和删除操作的性能。**它在性能上和红黑树、AVL 树不相上下，但是跳表的原理非常简单，实现也比红黑树简单很多。**

基于以上特性，跳表是一种能在工程中替代平衡树的随机化数据结构

## 为什么需要跳表？
在工程中，我们经常需要一种数据结构，支持：
- 快速查找（find）
- 快速插入（insert）
- 快速删除（erase）
- 元素保持有序
最直观的方案是平衡二叉搜索树（如红黑树、AVL 树），但它们有两个明显问题：
- 实现复杂：旋转、颜色维护、平衡条件难以调试
- 并发场景不友好：复杂结构导致锁粒度难控制


## 跳表的核心思想：从链表一步步“进化”
#### 从有序链表开始
假设我们维护一个有序链表：
```
1 → 3 → 5 → 7 → 9 → 11
```
查找元素的时间复杂度是O(n)
###### 代码实现
```cpp
struct ListNode {
    int val;
    ListNode* next;         
    ListNode(int v, ListNode* node) : val(v), next(node) {}
    ListNode(int v) : ListNode(v, nullptr) {}
    ListNode(ListNode* node) : ListNode(0, node) {}
    ListNode() : ListNode(0, nullptr) {}
};

bool find(ListNode* head, int target) {
    ListNode* cur = head;
    while (cur) {
        if (cur->val == target) return true;
        if (cur->val > target) return false;
        cur = cur->next;
    }
    return false;
}
```
问题很明显：即使链表是有序的，查找效率依然很低。

#### 引入“跳跃”：给链表加一条捷径
既然查找慢的原因是走得太细碎，那一个自然的想法是：
能不能在链表中跳着走？
在原有链表的基础上，我们人为选出一部分节点，组成一条“快链表”
```
Level 2: 1 --------> 7 ----> 11
Level 1: 1 → 3 → 5 → 7 → 9 → 11
```
###### 更新代码
```cpp
struct ListNode {
    int val;
    ListNode* next;
    ListNode* nextfast; // 增加一个快速链表 
    ListNode(int v,ListNode* node) : val(v),next(node){};
    ListNode(int v) : ListNode(v,nullptr){};
    ListNode(ListNode* node) : ListNode(0,node) {};
    ListNode(): ListNode(0,nullptr){};
};

bool find(ListNode* head,int target){
    ListNode* cur = head;
    while(cur->next && cur->next->val <= target)
    cur = cur->nextfast; // 快速链表
    
    while(cur && cur->val < target)
    cur = cur->next; // 普通链表

    return cur && cur->val == target; // cur必须现存在才行
}
```

#### 多层结构：让“跳跃”系统化
多层链表的直观结构
```yaml
Level 3:   2           8
Level 2:   2   4       8
Level 1:   2   4   6   8   10
Level 0: 1 2 3 4 5 6 7 8 9 10
```

###### 再次修改代码
```cpp
struct ListNode {
    int val;
    std::vector<ListNode*> next_;   // 每层的后继
    explicit ListNode(int v, size_t level = 1)
        : val(v), next_(level, nullptr) {}

    // 给节点再拔高一层
    void addLevel() { next_.push_back(nullptr); }
};
```

#### 随机化：跳表真正成立的关键
到现在为止，还有一个核心问题没有解决：每一层到底选哪些节点？

跳表的解决方案极其优雅：**不设计规则，交给随机数，用概率代替规则。**

使用50%概率举例,两条规则：
- 每个节点有 50% 的概率晋升到上一层
- 晋升可以连续发生

###### 随机层数生成代码
```cpp
int randomLevel() {
    int level = 1;
    while (rand() % 2 == 0 && level < MAX_LEVEL) {
        level++;
    }
    return level;
}
```
但是更一般的，我们使用随机数引擎来实现：
```cpp
std::mt19937 rng;
std::uniform_real_distribution<double> dist;
```
## 完整代码加注释
```cpp
#include<vector>
#include<random>
#include<climits>
#include<iostream>

/**
跳表节点定义
每个节点包含：
key：存储的值
next_：多层 forward 指针，next_[i] 表示第 i 层的下一个节点
 */
class SkipList{
    struct Node{
        int key_;
        std::vector<Node*> next_;
        explicit Node(int k,int level) : key_(k),next_(level,nullptr) {};
        explicit Node() : Node(INT_MIN,MAX_LEVEL) {};
    };

    // 最大层数
    static constexpr int MAX_LEVEL = 16;
    // 节点晋升概率
    static constexpr double P = 0.5;
    // 头节点（哨兵节点，不存储真实数据）
    Node* head_;
    // 当前跳表实际使用的最高层数
    int currentLevel_;

    // 随机数引擎
    std::mt19937 rng_;
    std::uniform_real_distribution<double> dist_;

    /**
    随机生成节点层数
    通过“抛硬币”方式决定是否晋升到更高层
     */
    int randomLevel() {
        int level = 1;
        while (dist_(rng_) < P && level < MAX_LEVEL) {
            ++level;
        }
        return level;
    }

public:
    SkipList():
    currentLevel_(1),
    rng_(std::random_device{}()),
    dist_(0,1){
        head_ = new Node();
    }

    ~SkipList(){
        const Node* cur = head_;
        while(cur != nullptr){
            const Node* next = cur->next_[0];
            delete cur;
            cur = next;
        }
    }

    /**
    查找 key 是否存在
    时间复杂度：O(log n)（期望）
     */
    [[nodiscard]] bool find(const int value) const{ /* const: 不能通过find函数来修改成员变量*/
        const Node* cur = head_;
        // 从最高层开始向下查找
        for(int i=currentLevel_-1;i>=0;i--){
            while(cur->next_[i] && cur->next_[i]->key_ < value){ //find
                cur = cur->next_[i];
            }
        }
        cur = cur->next_[0];
        return (cur && cur->key_ == value);
    }

    /*
    插入一个 key
    如果 key 已存在，则忽略
     */
    void insert(const int key){
        // update[i] 记录第 i 层中，待插入位置的前驱节点
        std::vector<Node*> update(MAX_LEVEL,nullptr);
        Node* cur = head_;
        for(int i=currentLevel_-1;i>=0;i--){
            while(cur->next_[i] && cur->next_[i]->key_ < key){
                cur = cur->next_[i];
            }
            update[i] = cur;
        }

        // 如果 key 已存在，直接返回
        if(cur->next_[0] && cur->next_[0]->key_ == key) return;

        //随机生成节点应该有的层数
        const int nodeLevel = randomLevel();
        std::cout<<"new node layer:"<<nodeLevel<<std::endl;

        /*
        如果新节点层数高于当前层数，需要更新头节点
        (头节点的高度等于所有节点中最高的那个节点)
        如原本是3层索引，现在随机数生成一个4层，就需要动头节点
        */
        if(nodeLevel<currentLevel_){
            for(int i=currentLevel_;i>nodeLevel;i--){
                update[i] = head_;
            }
            currentLevel_ = nodeLevel;
        }

        //链表的插入
        const auto newNode = new Node(key,nodeLevel);
        /*
         * const auto newNode = ... → Node* const（指针不可重指向）
         * const auto* newNode = ... → const Node*（内容不可改)
         */
        for(int i=0;i<nodeLevel;i++){
            newNode->next_[i] = update[i]->next_[i];
            update[i]->next_[i] = newNode;
        }
    }

    /**
    删除一个 key
    成功返回1,
    不存在就是失败，返回0
     */
    bool erase(const int value){
        //update 存储前区节点
        std::vector<Node*> update(MAX_LEVEL,nullptr);
        Node* cur = head_;

        //与前面一样，现维护update这个vector
        for(int i=currentLevel_-1;i>=0;i--){
            while(cur->next_[i] && cur->next_[i]->key_ < value){
                cur = cur->next_[i];
            }
            update[i] = cur;
        }

        const Node* target = cur->next_[0];
        //没找到，返回false
        if(target == nullptr || target->key_ != value) return false;

        // 逐层断开节点，如果发现节点的next_不是target了，就说明target已经到达最大层了，就break
        for(int i=0;i<currentLevel_;i++){
            if(update[i]->next_[i] != target) break;
            update[i]->next_[i] = target->next_[i];
        }
        delete target;

        //更新头节点
        while(currentLevel_ > 1 && head_->next_[currentLevel_-1] == nullptr){
            currentLevel_ --;
        }
        return true;
    }
};



```

## 数据结构分析
在前面的小节中，我们从有序链表一步步演化到了跳表。本节将从**时间复杂度、空间复杂度以及工程应用**三个角度，对跳表进行系统分析。

#### 时间复杂度分析
   跳表支持三种核心操作：
   * 查找（Search）
   * 插入（Insert）
   * 删除（Delete）
###### 查找复杂度
在跳表中，查找过程遵循如下策略：
1. 从最高层开始向右查找
2. 无法继续向右时，向下移动一层
3. 重复上述过程直到第 0 层
由于：
* 每一层都是下一层的“随机抽样”
* 层数的期望值为 `O(log n)`
因此：
> **查找操作的期望时间复杂度为 `O(log n)`**
   
需要注意的是，跳表并不保证严格平衡，因此：
* **最坏时间复杂度：O(n)**（概率极低）
* **期望时间复杂度：O(log n)**（工程中可视为稳定）
   
###### 插入复杂度
插入操作包含两个步骤：
1. 查找插入位置（与查找操作等价）
2. 生成随机层数并建立指针连接
因此：
* 查找部分：`O(log n)`
* 指针调整：与节点层数成正比（期望为常数）

> **插入操作的期望时间复杂度：`O(log n)`**
   
###### 删除复杂度
删除操作同样需要：
1. 找到目标节点在各层的前驱
2. 断开对应层的指针
与插入对称
> **删除操作的期望时间复杂度：`O(log n)`**
   

#### 空间复杂度分析

跳表的空间开销主要来自：
* 节点本身
* 每个节点维护的多层指针

###### 单个节点的空间期望

每个节点的层数由随机过程决定：
* 每晋升一层的概率为 1/2
* 节点高度的期望值为常数（约 2）
因此：
> **单个节点的期望指针数是 O(1)**
   
###### 整体空间复杂度
设元素数量为 `n`：
* 节点数：`n`
* 总指针数：期望为 `O(n)`
   > **跳表的空间复杂度为 `O(n)`**
   

#### 跳表 vs 红黑树（std::map）
   
| 维度     | 跳表   | 红黑树        |
| ------ | ---- | ---------- |
| 平衡方式   | 随机化  | 严格规则       |
| 最坏复杂度  | O(n) | O(log n)   |
| 实现难度   | 中等   | 高          |
| 并发扩展性  | 高    | 较低         |
| STL 支持 | 无    | `std::map` |

   > **当你不想与旋转和颜色打交道时，跳表是更友好的选择。**

#### 工程应用场景
###### Redis
   * Redis 的 **有序集合（Sorted Set）**
   * 使用 **跳表 + 哈希表**
   
###### LevelDB / RocksDB（思想相通）
   * 内存结构中使用跳表维护有序数据
   * 顺序写入 + 范围扫描效率高
   * 非常适合 KV 存储系统 

###### 并发系统
   跳表具有以下优势：
   * 结构简单
   * 层级独立，易于分段加锁
   * 更适合无锁或细粒度锁设计
