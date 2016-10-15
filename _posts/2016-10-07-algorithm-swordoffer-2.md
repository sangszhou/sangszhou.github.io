---
layout: post
title: algorithm 3
categories: [algorithm]
keywords: algorithm
---

### 第一个只出现一次的字符

ebay 第一轮电面的题目, 跪了

题目: 在字符串中找到第一次只出现一次的字符和下标

例如, 输入 abaccdeff, 则输出 b

**解:**

创建一个 HashMap, Key 是字符, value 是出现的次数和下标。如果便利到一个字符, 且该字符
重来没出现过, 那么把这个字符和字符对应的 index 放到 Map 中, 此时他出现的次数是 1,
如果一个字符经过查询表, 发现已经存在了, 那么就把 value 中出现的次数置为负数, 表示他已经
不再考虑范围之内了, 经过一遍 pass, map 中包含了所有的字符

再来一遍 pass, 这遍 pass 中, 寻找那个下标最小的, 且出现次数为正的字符, 返回下标和字符

两边 pass, 第一遍 pass 的时间复杂度为 o(N), 第二遍的常数, 因为字符个数为常数




### 数组中的逆序对

```java
 * 题目：在数组中的两个数字如果前面一个数字大于后面的数字，则这两个数字组成一个逆序对。
 * 输入一个数组，求出这个数组中的逆序对的总数。
 * 例如在数组｛7, 5, 6, 4}中，
 * 一共存在 5 个逆序对，分别是（7, 6）、（7，5），(7, 4）、（6, 4）和（5, 4）。
 *
 * 【解】：归并排序的思想
 *  在把数组分成1个1个的数字之后，开始合并的过程中：
 *  - 两个{1个数的数组}开始合：
 *          只要第1个数比第2个数大，cnt++;
 *  - 两个{2个数的数组}开始合：
 *      p1：指向第一个数组的末尾
 *      p2：指向第二个数组的末尾
 *      p3：指向合并的{4个数的}大数组的末尾
 *          p1>p2： cnt+=2，aux[p3--] = aux[p1--],
 *          p1<=p2：cnt不变, aux[p3--] = aux[p2--],
 *  - 两个{4个数的数组}开始合：
 
    private static int inversePairs(int[] arr) {
        if (arr==null || arr.length<=0){
            throw new IllegalArgumentException("args should not be null or empty");
        }
        int[] aux = new int[arr.length];
        System.arraycopy(arr, 0, aux, 0, arr.length);
        return inversePairsCore(arr, aux, 0, arr.length-1);
    }

    private static int inversePairsCore(int[] arr, int[] aux, int lo, int hi) {
        if (lo == hi){
            aux[lo] = arr[lo];
            return 0;
        }
        int len = (hi-lo)/2;
        int left = inversePairsCore(aux, arr, lo, lo+len);
        int right = inversePairsCore(aux, arr, lo+len+1, hi);

        int p1 = lo+len;
        int p2 = hi;
        int p3 = hi;
        int cnt = 0;
        while (p1>=lo && p2>=lo+len+1){
            if (arr[p1] > arr[p2]){
                cnt += p2-lo-len;
                aux[p3--] = arr[p1--];
            } else {
                aux[p3--] = arr[p2--];
            }
        }

        while (p1 >= lo){
            aux[p3--] = arr[p1--];
        }
        while (p2 >= lo+len+1){
            aux[p3--] = arr[p2--];
        }

        return cnt+left+right;
    }
```

### 两个单向链表的第一个公共结点

```java
     1. 两个链表：一个有环，一个无环
     则两个链表不可能相交
     return null;
     2. 两个链表：都无环
     - 不相交 （"||"型）
     - 相交   （"Y " 型）（不可能是"X "型的，因为一个结点只能有一个next元素）
         法一：
         HashMap 保存第一个链表结点之间的关系
         然后比较第二个链表，
         法二：
         先统计两条链表的长度
         然后让长链表先把多出来的步数走掉
         然后两条链表一起走，相遇则“Y”，否则为“||”
     3. 两个链表：都有环
     - 先相交，再共同入环（两个链表的第一个入环结点相等）
     - 共享环，（在环外没遇上）：让环1的入环结点，往下走，如果遇到了环2的入环结点，√
     - 不相交： 两环的入环的第一个节点没遇上
 *
```

### 数字在排序数组中出现的次数

```java
 * 题目：统计一个数字：在排序数组中出现的次数。
 * 例如输入排序数组｛ 1, 2, 3, 3, 3, 3, 4, 5｝和数字 3 ，
 *      由于 3 在这个数组中出现了 4 次，因此输出 4 。
 *
 * 【解】：
 *      既然是排好序的数组,
 *      所以我们可以直接使用两次二分查找，得到数字第一次出现和最后一次出现的下标，
 *      时间复杂度：2 * O(logN)

    private static int getLastK(int[] arr, int k, int lo, int hi) {
        if (lo > hi) {
            return -1;
        }
        int mid = (lo+hi)>>1;
        int len = arr.length;
        if (k > arr[mid]){
            lo = mid + 1;
        } else if (k == arr[mid]){
            if ((mid<len-1 && arr[mid+1]>k) || mid==len-1 ){    // mid还有可能等于0
                return mid;
            } else {
                lo = mid + 1;
            }
        } else {
            hi = mid - 1;
        }
        return getLastK(arr, k, lo, hi);
    }

    private static int getFirstK(int[] arr, int k, int lo, int hi) {
        if (lo > hi) {
            return -1;
        }
        int mid = (lo+hi)>>1;
        if (k > arr[mid]){
            lo = mid + 1;
        } else if (k == arr[mid]){
            if ((mid>0 && arr[mid-1]<k) || mid==0 ){    // mid还有可能等于0
                return mid;
            } else {
                hi = mid-1;
            }
        } else {
            hi = mid - 1;
        }
        return getFirstK(arr, k, lo, hi);
    }
}
```

### 二叉树的深度

```java
    public static int TreeDepth(BinaryTreeNode head){
        if (head==null){
            return 0;
        }
        int left = TreeDepth(head.left);
        int right = TreeDepth(head.right);
        return left>right? left+1: right+1;
    }
```

### 判断二叉树是否平衡

题目里使用 depth 0 是因为简单类型作为参数是被 copy 的, depth 作为引用类型

```java
 * 题目二：输入一棵二叉树的根结点，判断该树是不是平衡二叉树。
 * 如果某二叉树中任意结点的左右子树的深度相差不超过 1 ，那么它就是一棵平衡二叉树。
 *
 * 【法1】：
 *      可以根据上一题已求出的TreeDepth，
 *      判断此结点的左右子树的深度相差如果如果不超过1，则平衡
 *
 * 【法2】：
 *      由于法1，每次判断都会重复遍历最下面叶子结点，影响性能！
 *      所以此题最优解：
 *          应该采用后序遍历的方法
 *          因为后序遍历，每遍历到一个根结点，他的左右子树都已经走过一遍了，
 *          所以 我们可以没走过一个结点，标上他的深度。然后即可直接判定此结点下面的二叉树是否平衡了

    private static boolean isBalanced2(BinaryTreeNode head){
        int[] depth = new int[]{0};
        return isBalanced2(head, depth);
    }

    private static boolean isBalanced2(BinaryTreeNode head, int[] depth) {
        if (head == null){
            depth[0] = 0;
            return true;
        }
        int[] left = new int[]{0};
        int[] right = new int[]{0};
        if (isBalanced2(head.left, left) && isBalanced2(head.right, right)){
            int diff = left[0] - right[0];
            if (diff>=-1 && diff<=1){
                depth[0] = 1+ Math.max(left[0], right[0]);
                return true;
            }
        }
        return false;
    }
```

### 数组中只出现一次的数字

```java
 * 题目：一个整型数组里除了两个数字之外，其他的数字都出现了两次，请写程序找出这两个只出现一次的数字。
 * 要求时间复杂度是 O(n)，空间复杂度是 O(1)。
 * 例如输入数组｛2, 4, 3, 6, 3, 2, 5 }，
 *     因为只有 4 、6 这两个数字只出现一次，其他数字都出现了两次，
 *     所以输出 4 和 6 。

 *  1. 假如数组中只有1个数字只出现过一次，其他的都出现了两次，怎么找到它？
 *      我们可以用异或，因为异或：相同为0，不同为1
 *      所以我们可以从头到尾依次异或数组中的每一个数字，最终的异或结果刚好是只出现一次的那个数字。（那些成对出现的两个数字刚好在异或中抵消了）
 *  2. 那数组中又两个只出现一次的数字呢？
 *      我们可以把大数组分成两个子数组啊，每个子数组中都只有1个只出现一次的数字。
 *      这样分：
 *          - 还是从头到位依次异或数组中的每一个数字，最终得到的结果 也是这两个只出现一次的数字异或的结果。
 *          - 假设最终得到的异或结果为 0010，也就是说： 这两个数字的二进制表示时的第3位，一定是一个为1，一个为0
 *          - 因此 我们可以将数组中的所有数字按第3位是1的分成一组，第3位是0的分成一组。
 *          - 然后，再分别求出两个子数组中的那个只出现过一次的数字吧。
```

### 和为s的两个数字

```java
 * 题目一：输入一个递增排序的数组和一个数字 s，在数组中查找两个数，得它们的和正好是 s。如果有多对数字的和等于 s，输出任意一对即可。
 * 例如输入数组｛1 、2 、4、7 、11 、15 ｝和数字 15。
 *  由于 4+11 = 15 ，因此输出 4 和 11 。
 * 【解】：双指针
 *    两个指针：p1 = arr[0], p2 = arr[length-1]
 *    do{
 *       p1+p2 > s , p2--
 *       p1+p2 == s , return p1、p2
 *       p1+p2 < s , p1++
 *    }while(p1<p2);
```

### 和为s的连续正数序列

```java
 * 题目二：输入一个正数 s，打印出所有和为 s 的连续正数序列（至少两个数）。
 * 例如输入 15，
 *   由于 1+2+3+4+5 = 4+5+6 ＝ 7+8 = 15，
 *   所以结果打出 3 个连续序列 1～5、4～6 和 7～8。
 *
 * 【解】：还是双指针
 *    p1 = 1，指向序列的第一个字符
 *    p2 = 2；指向序列的末尾
 *    由于序列至少要有两个数字，所以p1不可能>(1+s)/2。
 *    while(p1 < (1+s)/2){
 *        p1+...+p2 == s, 保留当前序列，
 *        while( curSum > s && p1<(1+s)/2){
 *             p1++;
 *             if(p1+...+p2 == s)
 *                 保留当前序列，
 *        }
 *        (p1+...+p2 < s)  p2++;
 *    }

    private static List<List<Integer>> findContinuousSequence(int s) {
        List<List<Integer>> result = new ArrayList<>();
        if (s < 3){
            return result;
        }
        int p1 = 1;
        int p2 = 2;
        int mid = (1+s) / 2;    //p1如果>mid, p1+p2一定大于s,
        int curSum = p1+p2;
        while (p1 < mid){
            if (curSum == s){
                addListToResult(p1, p2, result);
            }
            while (curSum > s && p1<mid){
                curSum -= p1;
                p1++;
                if (curSum == s){
                    addListToResult(p1, p2, result);
                }
            }
            //curSum < s
            p2++;
            curSum += p2;
        }
        return result;
    }

    private static void addListToResult(int p1, int p2, List<List<Integer>> result) {
        List<Integer> list = new ArrayList<>();
        for (int i = p1; i <= p2; i++) {
            list.add(i);
        }
        result.add(list);
    }
```

### 翻转单词顺序

某一年的 408 考题

```java
 * 题目一：输入一个英文句子，翻转句子中单词的顺序，但单词内字啊的顺序不变。为简单起见，标点符号和普通字母一样处理。
 * 例如输入字符串”I am a student. ”，
 *   则输出”student. a am I”。
 *
 *
 * 【解】：翻转两次字符串即可
 *  第一步，翻转句子中的所有字符
 *      比如翻转“I am a student. ”中所有的字符得到“.tneduts am a I”，
 *  第二步，再翻转每个单词中字符的顺序
 *      就得到了“student. a am I”。
```

### 左旋转字符串

```java
 * 题目二：字符串的左旋转操作是把字符串前面的若干个字符转移到字符串的尾部。请定义一个函数实现字符串左旋转操作的功能。
 * 比如输入字符串“abcdefg”和数字 2，
 * 该函数将返回左旋转 2 位得到的结果“cdefgab”。
 * 【解】：与上一题类似：

 *  第一步，翻转前半部分的字符：ba cdefg
 *  第二步，翻转后半部分的字符：ba gfedc
 *  第三部，翻转整个字符串：cdegfab
 *
```

### 单链表的公共后缀

loading 和 being 的公共后缀为 ing

2012 年 408 考题, 我做出来的, 13 分。现在还是得想个 2 分钟

第一遍 pass 得到两个单链表的长度, 长度的差值记为 diff, 然后设置两个指针, 分别指向两个单链表的头部,
较长的单链表先走 diff 步, 然后两个节点开始走, 每走一步比较指针是否重合

### n个骰子的点数 @todo

```java
 * 题目：把 n 个骰子扔在地上，所有骰子朝上一面的点数之和为 s。输入 n，打印出 s 的所有可能的值出现的概率。

```

### 扑克牌的顺子

```java
 * 题目：从扑克牌中随机抽 5 张牌，判断是不是一个顺子， 即这 5 张牌是不是连续的。
 * 2～10 为数字本身， A 为 1。 J 为 11、Q 为 12、 K 为 13。大、小王可以看成任意数字。

```

### 树中两个结点的最低公共祖先

```java
 * 题目：输入两个树结点，求他们的最低公共祖先
 *  1. 此树不一定是二叉树，（如果是二叉搜索树==> 二分查找 O（logN））
 *  2. 树中的结点没有指向父结点的指针（直接将此题转化为：求两个链表的第一个公共结点问题）
 *  【解】：
 *      即使是一颗多叉树，一样可以转化为 求两个链表的公共结点问题
 *      1. 将到达两个树结点的必经之路记录下来，
 *      2. 然后开始寻找两条链表的最后公共结点

    private static TreeNode getLastCommonParent(TreeNode root, TreeNode p1, TreeNode p2) {
        if (root==null || p1==null || p2==null){
            return null;
        }
        List<TreeNode> path1 = new LinkedList<>();
        getNodePath(root, p1, path1);
        List<TreeNode> path2 = new LinkedList<>();
        getNodePath(root, p2, path2);
        return getLastCommonNode(path1, path2);
    }

// 递归求解
    private static void getNodePath(TreeNode root, TreeNode p, List<TreeNode> path) {
        if (root == null){
            return;
        }
        path.add(root);
        List<TreeNode> children = root.children;
        //挨个递归处理各个子结点
        for(TreeNode node: children){
            if (node == p){
                path.add(node);
                return;
            } else {
                getNodePath(node, p, path);
            }
        }
        //此路不通，原路返回
        path.remove(path.size() - 1);
    }

// 转化为公共自学列问题
    private static TreeNode getLastCommonNode(List<TreeNode> path1, List<TreeNode> path2) {
        Iterator<TreeNode> it1 = path1.iterator();
        Iterator<TreeNode> it2 = path2.iterator();
        
        TreeNode last = null;
        while (it1.hasNext() && it2.hasNext()){
            TreeNode tmp = it1.next();
            if (tmp == it2.next()){
                last = tmp;
            }
        }
        return last;
    }

```

### 数组中重复的数字 

有些复杂

```java
 * 题目：在一个长度为n的数组里的所有数字都在 0 到 n-1 的范围内。数组中某些数字是重复的，但不知道有几个数字重复了，
   也不知道每个数字重复了几次。
 * 请找出数组中任意一个重复的数字。
 * 例如，如果输入长度为 7 的数组 {2,3,1,0,2,5,3}，
 * 那么对应的输出是重复的数字 2 或者 3。
 *
 * 【法1】：一言不合先排序，排序之后双指针往后跑，直到找到相同的数字，return
 *      时间复杂度：O（N*logN）+ O（N）
 * 【法2】：用HashMap，遍历一下数组即可
 *      时间复杂度：O（N）+ N*O（1）
 *      空间复杂度：O（N）
 * 【法3】：依照题意，数组中的数字都在[0,n-1]的范围内，
 *      所以如果没有重复数字的话，每个数字就应该都在他对应的下标下面
 *      但是有重复的数字，数字还不是有序的：
 *          我们可以依次遍历整个数组，当遍历到第i个数字时，
 *
 *          arr[i] == i, i++
 *          while(arr[i] != i){ //直到i这个位置插上了i，或者找到了一对重复的数字
 *              arr[arr[i]] == arr[i], 找到了重复的数字
 *              arr[arr[i]] != i, 
                swap(arr[arr[i]], arr[i])
 *          }
 *
 *       时间复杂度：O（N），数组中的每个元素只需遍历或交换一遍，即可找到那个重复的数字
 *       空间复杂度：O（1）
```

### 删除链表中重复的结点

```java
 * 题目：在一个排序的链表中，如何删除重复的结点？
 * 例如，1 -> 2 -> 3 -> 3 -> 4 -> 4 -> 5 -> null 中重复结点被删除之后，链表变成了：
 *      1 -> 2 -> 5 -> null
 *
 * 【注意陷阱】：
 *      头结点也可能被删除，所以在链表头添加一个临时头结点 root。
 *      然后，最后返回 root.next即可
```

### 二叉树的下一个结点

```java
 * 题目：从上到下按层打印二叉树，同一层的结点按从左到右的顺序打印，每一层打印一行。
 *      为了把二叉树的每一行单独打印到一行里，我们需要两个变量：
 *       - current：表示在当前的层中还没有打印的结点数，
 *       - next：表示下一层结点的数目。

    private static void print(BinaryTreeNode root) {
        if (root == null) {
            return;
        }
        List<BinaryTreeNode> list = new LinkedList<>();
        BinaryTreeNode node;
        int current = 1;     // 当前层的结点个数
        int next = 0;   // 记录下一层的结点个数
        list.add(root);
        while (list.size() > 0) {
            node = list.remove(0);
            current--;
            System.out.printf("%-3d", node.value);
            if (node.left != null) {
                list.add(node.left);
                next++;
            }
            if (node.right != null) {
                list.add(node.right);
                next++;
            }
            if (current ==0) {
                System.out.println();
                current = next;
                next = 0;
            }}}}
```

### 二叉搜索树的第k个结点

```java
 * 题目：给定一棵二叉搜索树，请找出其中的第k大的结点。
 *
 * 【解】：
 *     因为按照中序遍历的顺序遍历一棵二叉搜索树，遍历序列的数值是递增排序的。
 *     所以只需要用中序遍历的方法遍历这棵二叉搜索树，就很容易找出它的第k大结点。
 *
```

### 题目：如何得到一个数据流中的中位数？

[link](https://github.com/nibnait/algorithms/blob/master/src/SwordOffer/h64_%E6%95%B0%E6%8D%AE%E6%B5%81%E4%B8%AD%E7%9A%84%E4%B8%AD%E4%BD%8D%E6%95%B0.java)

大小根堆之间的元素会相互移动, 从大根堆向小根堆走, 从小根堆到大根堆中

```java
 *    如果从数据流中读出奇数个数值，那么中位数就是所有值排序之后位于中间的数值
 *    如果数据流中读出偶数个数值，那么中位数就是所有数值排序之后中间两个数的平均值

 *【法5】：建立一个大根堆和一个小根堆 √
 *      小根堆的堆顶(最小元素) 大于 大根堆的堆顶(最大元素)
 *      当数据总数目是偶数时，把新数据插入到小根堆中，否则插入大根堆中（保证两个堆中数据的数目之差不超过1）
 *      这样我们往堆中插入的时间效率为O（logN），
 *      由于只需O（1）的时间即可得到堆顶元素，因此得到中位数的时间效率就是O（1）

```

### 滑动窗口的最大值 @todo

这个和带 min 的堆栈反过来了

比较难

```java
 * 题目：给定一个数组和滑动窗口的大小，请找出所有滑动窗口里的最大值
 * 例如，如果输入数组 {2,3,4,2,6,2,5,1}及 滑动窗口的大小3
 * 那么一共存在 6 个滑动窗口，它们的最大值分别为{4,4,6,6,6,5}
 *
 思考如果是 4, 2, 3, 3 时的双端队列样子
 
 *【解】: 用一个双端队列
 *  队列里存的是数组元素的下标
 *  当前队列头永远放的是当前窗口的最大值
 移动时, 怎么添加元素? 把队列后后面的元素丢掉!!!

    private static int[] maxInWindows(int[] arr, int w) {

        if (arr.length < w){
            int max = arr[0];
            for(int i : arr){
                max = max>i? max : i;
            }
            
            return new int[]{max};
        }

        LinkedList<Integer> deque = new LinkedList<Integer>();
        
        int[] res = new int[arr.length - w + 1];
        int cnt = 0;
        
        for (int i = 0; i < arr.length; i++) {
            while (!deque.isEmpty() && arr[deque.peekFirst()] <= arr[i]) {
                deque.pollLast();
            }
            
            deque.addLast(i);
            if (deque.peekFirst() <= i - w) {
                deque.pollFirst();
            }
            if (i >= w - 1) {
                res[cnt++] = arr[deque.peekFirst()];
            }}
        return res;
    }}

```

### 矩阵中的路径

```java
 * 题目：请设计一个函数，用来判断在一个矩阵中是否存在一条包含某字符串所有字符的路径。
 *      路径可以从矩阵中任意一格开始，每一步可以在矩阵中间向左、右、上、下移动一格。
 *      如果一条路径经过了矩阵的某一格，那么该路径不能再次进入该格子。
 * 例如在下面的 3*4 的矩阵中包含一条字符串“bcced”的路径。
 *                 但矩阵中不包含字符串“abcb”的路径，因为字符串的第一个字符 b 占据了矩阵中的第一
 行第二格子之后，路径不能再次进入这个格子。
         a b c e
         s f c s
         a d e e

 *  这是一个可以用回朔法解决的典型题。首先，在矩阵中任选一个格子作为路径的起点。

```

### 袋鼠过河

 题目描述：

 一只袋鼠要从河这边跳到河对岸，河很宽，但是河中间打了很多桩子，每隔一米就有一个，每个桩子上都有一个弹簧，袋鼠跳到弹簧上就可以跳的更远。
 每个弹簧力量不同，用一个数字代表它的力量，如果弹簧力量为5，就代表袋鼠下一跳**最多**能够跳5米，如果为0，就会陷进去无法继续跳跃。
 河流一共N米宽，袋鼠初始位置就在第一个弹簧上面，要跳到最后一个弹簧之后就算过河了，给定每个弹簧的力量，求袋鼠最少需要多少跳能够到达对岸。
 如果无法到达输出-1

 输入分两行，第一行是数组长度N，第二行是每一项的值，用空格分隔

 输出最少的跳数，无法到达输出-1

 样例输入

 5
 2 0 1 1 1

 样例输出

 4

感觉像 DP 算法, 可以重当前节点往后跳

比如 a[0] = 2, 那么 dp[2] = 1. 然后可以在 dp[1] 处更新 dp[2] (没得更新了)

第一次到达 dp[length-1] 即为解

算不上 DP, 就是每次更新后面的值就好了吧
