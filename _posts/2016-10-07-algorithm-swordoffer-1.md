---
layout: post
title: algorithm sword upon offer
categories: [algorithm]
keywords: algorithm
---

[参考资料](https://github.com/nibnait/algorithms)

### b02_单例模式

```java
//4. 静态内部类
class Singleton_StaticInnerClass {
    private static class SingletonHolder{
        private static final Singleton_StaticInnerClass INSTANCE = new Singleton_StaticInnerClass();
    }
    private Singleton_StaticInnerClass(){}
    public static Singleton_StaticInnerClass getInstance(){
        return SingletonHolder.INSTANCE;    //只有在调用 getInstance() 时，对象才会被创建，同时没有性能缺点，也不依赖 Java 版本。
    }
}
```

### 二维数组中的二分查找

```java
/**
 * 在一个二维数组中，每一行都按照从左到右递增的顺序排序，每一列都按照从上到下递增的顺序排序。
 * 请完成一个函数，输入这样的一个二维数组和一个整数，判断数组中是否含有该整数。
 *
 * 【思路】
 * 从二维数组的右上角往左下角开始比较。
 */
    private static int findInPartiallySortedMatrix(int[][] matrix, int key) {
        int rows = matrix.length;
        int cols = matrix[0].length;
        if (matrix==null || rows<1 || cols<1){
            return 0;
        }
        int row = 0;
        int col = cols-1;
        while (row<rows && col>=0){
            if (matrix[row][col] == key){
                return 1;
            } else if (matrix[row][col] < key){
                row++;
            } else {
                col--;
            }
        }
        return 0;}}
```

### 替换空格

一个空格换成三个字符

```java
/**
 * 自然想法，遇到空格就替换成'20%'，长度是n的字符串，遇到一个空格，就需要向后移动n个字符，所以时间复杂度为O（N^2)
 * 【思路】
 * 双指针，从后往前遍历替换：
 *      p1：指向字符串末尾，
 *      p2：指向替换之后的字符串的末尾（需提前遍历一遍str，数一下一共有多少个空格）
 *  然后p1和p2一起向前跑，当p1遇到空格，则将' '变成'%20'，然后继续向前跑。
 */
public class b04_替换空格 {
    /**
     * @param str   需要替换的字符串数组
     * @param size  str中OriginalString的长度
     * @return
     */
    private static String replaceBlank(char[] str, int size) {
        if (str==null || size<=0){
            return null;
        }

        int cnt = 0;
        for (int i = 0; i < size; i++) {
            if (str[i] == ' '){
                cnt++;
            }
        }
        int p1 = size-1;
        int p2 = p1+(2*cnt);
        if (p2 > str.length){
            throw new RuntimeException("Invalid args: str有点小，不够装！");
        }
        while (p1 >= 0){
            if (str[p1] == ' '){
                str[p2--] = '0';
                str[p2--] = '2';
                str[p2--] = '%';
            } else {
                str[p2--] = str[p1];
            }
            p1--;
        }
        return new String(str);}}
```

### 从尾到头打印链表

曾经跪在这道题上...

```java
/**
 * 不改变原数据结构
 * --> 用 栈
 * -->改写成 递归√
 * Created by nibnait on 2016/9/20.
 */
public class b05_从尾到头打印链表 {

    /**
     * 递归
     * @param head
     */
    private static void printListInverselyUsingRecursion(ListNode head) {
        if (head == null){
            return;
        }
        printListInverselyUsingRecursion(head.next);
        System.out.print(head.value+ " ");
    }

    private static void printListInverselyUsingStack(ListNode head) {
        if (head == null){
            return;
        }
        Stack<Integer> stack = new Stack<>();
        while (head != null){
            stack.push(head.value);
            head = head.next;
        }
        while (!stack.isEmpty()){
            System.out.print(stack.pop()+ " ");
        }
    }
```

### 重建二叉树

```java
/**
 * 根据前序和中序序列（不含有重复的数字），构建一棵二叉树
 */
public class b06_重建二叉树 {
    private static BinaryTreeNode constructBinaryTree(int[] pre, int[] in) {

        if (pre==null || in==null || pre.length!=in.length || in.length<=0){
            return null;
        }
        HashMap<Integer, Integer> map = new HashMap<>();
        for (int i = 0; i < in.length; i++) {
            map.put(in[i], i);
        }
        return construct(pre, 0, pre.length-1, in, 0, in.length-1, map);
    }

    private static BinaryTreeNode construct(int[] pre, int ps, int pe, int[] in, int is, int ie, HashMap<Integer, Integer> map) {

        if (ps > pe){
            //开始位置大于结束位置，说明已经没有需要处理的元素了
            return null;
        }

        int value = pre[ps];     // 取前序遍历的第一个数字，就是当前的根结点
        int i = 0;
        try {
            i = map.get(pre[ps]);   // 在中序遍历的数组中找根结点的位置
        } catch (Exception e){
            throw new IllegalArgumentException("Invalid args: 前序/中序数组有问题");
        }
        //创建当前根节点
        BinaryTreeNode head = new BinaryTreeNode(value);
        //递归：
        head.left = construct(pre, ps+1, ps+i-is, in, is, i-1, map);
        head.right = construct(pre, ps+1+i-is, pe, in, i+1, ie, map);
        return head;
    }
```

### 用两个栈实现队列

@todo 两个优先队列可以做什么事来着? 跪在 baidu 的一道题

```java
public class b07_用两个栈实现队列 {
    private static class QueueWithTwoStacks {
        public Stack<Integer> stackPush;
        public Stack<Integer> stackPop;

        public QueueWithTwoStacks() {
            stackPush = new Stack<Integer>();
            stackPop = new Stack<Integer>();
        }

        public void add(int pushInt) {
            stackPush.push(pushInt);
        }

        public int poll() {
            if (stackPop.empty() && stackPush.empty()) {
                throw new RuntimeException("Queue is empty!");
            } else if (stackPop.empty()) {
                while (!stackPush.empty()) {
                    stackPop.push(stackPush.pop());
                }
            }
            return stackPop.pop();
        }

        public int peek() {
            if (stackPop.empty() && stackPush.empty()) {
                throw new RuntimeException("Queue is empty!");
            } else if (stackPop.empty()) {
                while (!stackPush.empty()) {
                    stackPop.push(stackPush.pop());
                }
            }
            return stackPop.peek();}}}
```

### 旋转数组中二分查找最小数字

```java
/**
 * 【题目】
     * 把一个数组最开始的若干个元素搬到数组的末尾， 我们称之数组的旋转。
     * 输入一个递增排序的数组的一个旋转， 输出旋转数组的最小元素。
     * 例如数组{ 3, 4, 5, 1, 2} 为{ l, 2, 3, 4, 5}的一个旋转，该数组的最小值为 1。
 */

public class b08_旋转数组中二分查找最小数字 {
    private static int MinNumberInRotatedArray(int[] arr){
        if (arr==null || arr.length==0){
            throw new IllegalArgumentException("args should not be null or empty");
        }

        int lo = 0;
        int hi = arr.length - 1;
        int mid = 0;
        
        while (arr[lo] >= arr[hi]){
            if (hi-lo == 1){
                return arr[hi]; //当lo和hi挨着的时候，则hi（后面那个数）一定就是最小值了
            }
            mid = (lo+hi)>>1;  //防止溢出
        
            if (arr[mid]==arr[lo] && arr[mid]==arr[hi]){
                //1,0,1,1,1
                //1,1,1,0,1
                //此时就要遍历整个arr[lo...hi]了，
                return MinInOrder(arr, lo, hi);
            }
        
            if (arr[mid] >= arr[lo]){
                lo = mid;
            } else if (arr[mid] < arr[hi]) {
                hi = mid;
            }
        }
        return arr[mid];
    }
```

### 斐波那契数列

大一上学期, 转专业跪在这道题上

```java
public class b09_斐波那契数列 {
    private static long Fibonacci(int n) {

        if (n <= 0){
            return 0;
        } else if (n == 1){
            return 1;
        } else {
            long fibNMinusOne = 0;
            long fibNMinusTwo = 1;
            long fibN = 0;
            for (int i = 2; i <= n; i++) {
                fibN = fibNMinusOne + fibNMinusTwo;
                fibNMinusOne = fibNMinusTwo;
                fibNMinusTwo = fibN;
            }
            return fibN;
        }
    }
```

### 二进制中1的个数

```java
    public static int NumberOf1InBinary(int n){
        int cnt = 0;
     
        while (n != 0){
            n = n & (n-1);
            cnt++;      //所以cnt就记录着 n一共被减了多少次1，即n的二进制表示中1的个数
        }
        return cnt;
    }
```

### 数值的整数次方

```java
    private static double powWithRecursion(double base, int exponent) {
        if (exponent == 0){
            return 1;
        }
        if (exponent == 1){
            return base;
        }
        double result = powWithRecursion(base, exponent>>1);
        result *= result;
        if (exponent % 2 != 0){     //如果exponent为奇数
            return result *= base;  //还得乘一次base
        }
        return result;
    }
```

### 顺序打印出从1最大的n位十进制数

```java
 * 输入数字n，按顺序打印出从1最大的n位十进制数。
 * 比如输入3，则打印出1、2、3、4、5、6...... 一直到最大的3位数即999。
 *      打印1 ~ n位的所有十进制数，其实就是从第1位开始设置0~9的全排列，直到递归将最后一个位置设置完毕，开始打印。

// 全排列
    
    private static void Print1ToMaxOfNDigits2(int i, int[] arr) {
        if (i == arr.length){
            pringNumber(arr);
            return;
        }
        
        for (int j = 0; j <= 9; j++) {
            arr[i] = j;
            Print1ToMaxOfNDigits2(i+1, arr);
        }
    }
```

### 删除链表结点

```java
/**
 * 给定单向链表的头指针和一个结点指针，定义一个函数在0(1)时间删除该结点,
 *
 * 【抖机灵】
 *      正常删除链表节点都得给个头指针和要删除的结点，然后从头开始遍历寻找
 *      但是要求时间复杂度是1，下面抖个机灵：
 *      我们可以直接复制这个结点的下一个结点，然后再将这个结点的下一个结点删除。over
 * 【注意】
 * - 要删除的结点是尾结点
 *     - 没办法，NULL为系统中的特定的那块区域！！并无法复制，所以只能从头遍历，得到该结点的前序结点，删除。
 *     - 如果链表只有一个结点，即这个要删除的结点也是头节点，则，将nodeToBeDeleted置为空之后，还需把头节点置为空；
 * - 由于Java子函数，只能是值传递，（所以不像C++，）必须返回新链表头节点，不然子函数就白调用了。

public class c13_删除链表结点 {
    private static ListNode deleteNodeInList(ListNode head, ListNode nodeToBeDeleted) {
        if (head==null || nodeToBeDeleted==null){
            return head;
        }

        if (nodeToBeDeleted.next != null){  //只要删除的不是尾结点
            ListNode tmp = nodeToBeDeleted.next;
            nodeToBeDeleted.value = tmp.value;
            nodeToBeDeleted.next = tmp.next;
            tmp = null;
        } else if (head == nodeToBeDeleted){    //链表中就这么一个结点
            nodeToBeDeleted = null;
        } else {    //多个结点的情况下，删除尾结点
            ListNode tmp = head;
            while (tmp.next != nodeToBeDeleted){
                tmp = tmp.next;
            }
            nodeToBeDeleted = null;
            tmp.next = null;
        }
        return head;}
```

### 调整数组顺序使奇数位于偶数前面

```java
 * 输入一个整数数组，实现一个函数来调整该数组中数字的顺序，使得所有奇数位于数组的前半部分，所有偶数位予数组的后半部分。
 *  O(N^2)的解法：
 *    插入排序的变形：将交换的判断条件改为（arr[j-1]为偶数 && arr[j]为奇数）
 *  O(N)的解法：
 *    两个指针：p1 = arr[0], p2 = arr[length-1]
 *      while(p1<=p2){
 *          p1往右走，遇到偶数即停
 *          p2往左走，遇到奇数即停
 *          swap(arr, p1, p2);
 *      }
 *   【注意】
 *      为了增加程序的鲁棒性：
 *          可将p1、p2的判断条件 写成一个独立的函数，
 *          这样前置的就不仅可以是奇数、也可以将非负数放在前面、将所有能被3整除的数放在前面...


    //O(N)
    public static void Reorder(int[] arr) {
        if (arr==null || arr.length<=1){
            return;
        }

        int p1 = -1;
        int p2 = arr.length;
        while (p1 < p2){
            while (p1 < p2 && flag(arr[++p1]));
            while (p1 < p2 && !flag(arr[--p2]));
            std.swap(arr, p1, p2);
        }
    }
```

### 链表中倒数第k个结点

```java
/**
 * 题目：输入一个链表，输出该链表中倒数第k 个结点．
 * 为了符合大多数人的习惯，本题从 1 开始计数，即链表的尾结点是倒数第 1 个结点．
 * 例如一个链表有 6 个结点，从头结点开始它们的值依次是 1 、2、3、4、5 、6。这个个链表的倒数第 3 个结点是值为 4 的结点。
 *
 * 【思路】
 *   两个指针:
 *      p1指向头节点
 *      p2指向第k-1个结点
 *      然后双指针一起往后跑，知道p2到达末尾
 *      则p1指向的就是倒数第k个结点
 *
 * 【注意】：特殊值的处理
 *  - 当输入的head为null时，直接return null;
 *  - 当输入k = 0时，由于我们是从1开始计数的，所以查找倒数第0个结点无意义，直接return null;
 *  - 当链表的结点数小于k时，则p2指不到第k-1个结点（NullPointerException），所以还要在for循环中加个判断！增加程序的鲁棒性
 *
 *  【相关题目】
 * - 求链表的中间节点
 * - 判断单向链表是否形成了环形结构？
 *  解：
 *      快慢指针，快指针一次走两步，慢指针一次走一步。
 *      若快指针走到头了，慢指针指向的就是中间节点，
 *      若快指针追上了慢指针，则是环形链表。

    public static ListNode findKthToTail(ListNode head, int k) {
        if (head==null || k<=0){
           return null;
        }
        ListNode p1 = head;
        ListNode p2 = head;
        for (int i = 1; i < k; i++) {
            if (p2.next != null){
                p2 = p2.next;
            } else {
                throw new RuntimeException("Invalid args: k > 链表的节点数");
            }
        }

        while (p2.next != null){
            p1 = p1.next;
            p2 = p2.next;
        }
        return p1;}
```

### 反转链表

```java
    public static ListNode reverseList(ListNode head) {
        if (head == null) {
            return null;
        }
        ListNode root = new ListNode(); //逻辑头节点
        root.next = null;
        ListNode next;
        while (head != null){
            next = head.next;
            head.next = root.next;
            root.next = head;
            head = next;
        }
        return root.next;}
```

### 合并两个排序的链表 

```java
 * 题目：输入两个递增排序的链表，合并这两个链表并使新链表中的结点仍然是按照递增排序的
 *
 * 【解】：
 *      仅用4个辅助变量，实现时间复杂度:O(M+N)，空间复杂度:O(1)

    /**
     * 书上的解法：递归，
     * @return
     */
    private static ListNode mergeSortedLists3(ListNode head1, ListNode head2) {
        if (head1 == null) {
            return head2;
        } else if (head2 == null) {
            return head1;
        }

        ListNode tmp = new ListNode();
        if (head1.value <= head2.value){
            tmp = head1;    //head1.next = head2;
            tmp.next = mergeSortedLists3(head1.next, head2);
        }else {
            tmp = head2;    //head2.next = head1;
            tmp.next = mergeSortedLists3(head1, head2.next);
        }
        return tmp;
    }

    * 空间复杂度：O(1)
    * 使用4个辅助变量， head2往head1上合并
```

### 树的子结构

```java
 * 题目：输入两棵二叉树 A 和 B，判断 B 是不是 A 的子结构。
 *
 * 【解】：
 *      利用二叉树的神级遍历(空间复杂度：O(1)， 时间复杂度：O(h))
 *      转化成字符串匹配问题：KMP算法（时间：O(N)）
 *      总的时间复杂度：O(N)

    private static boolean hasSubtreeWithRecursion(BinaryTreeNode head1, BinaryTreeNode head2) {
        boolean result = false;

        if (head1.value == head2.value){    //找到了相同结点
            result = match(head1, head2);
        }

        if (!result){   //此结点未匹配出B树，继续考察 A.left
            result = hasSubtreeWithRecursion(head1.left, head2);
        }
        if (!result){   //左子树不行 换右子树
            result = hasSubtreeWithRecursion(head1.right, head2);
        }
        return result;
    }
    
        private static boolean match(BinaryTreeNode head1, BinaryTreeNode head2){
            if (head1.value == head2.value){
                return match(head1.left, head2.left) && match(head1.right, head2.right);
            }
            return false;
        }
```

### 二叉树的镜像

```java
 * 题目：请完成一个函数，输入一个二叉树，该函数输出它的镜像。
 *
 * 【思路】
 *  先序遍历这棵树的每个结点，
 *  如果遍历到的结点有子结点，就交换它的两个子结点。
 *  当交换完所有非叶子结点的左右子结点之后，就得到了树的镜像。

    private static void mirrorRecursivelly(BinaryTreeNode head) {
        if (head == null){
            return;
        }
        if (head.left==null && head.right==null){
            return;
        }
        BinaryTreeNode tmp = head.left;
        head.left = head.right;
        head.right = tmp;
        mirrorRecursivelly(head.left);
        mirrorRecursivelly(head.right);
    }
```

### 顺时针打印矩阵

```java
    public static void printMatrixClockWisely(int[][] numbers) {
        if (numbers == null || numbers.length==0){
            System.out.println("数组不能为空");
            return;
        }
        int rows = numbers.length;
        int columns = numbers[0].length;
        int start = 0;  //每一行起点的位置：(1,1), (2,2), (3,3), ...
        while (columns>2*start && rows>2*start){
            printMatrixInCircle(numbers, rows, columns, start);
            start++;
        }}

    private static void printMatrixInCircle(int[][] numbers, int rows, int columns, int start) {
        //打印一圈，分四步
        int endX = columns-1-start;
        int endY = rows-1-start;
        //1. 从左到右 打印一行
        for (int i = start; i <= endX; i++) {
            System.out.print(numbers[start][i]+" ");
        }

        //2. 从上到下打印一列
        //  环的高度至少是2，才会输出右边的一列
        //  即 要终止行号 > 起始行号
        if (endY > start){
            for (int i = start+1; i <= endY; i++) {
                System.out.print(numbers[i][endX]+" ");
            }
        }

        //3. 从右往左打印一行
        //  环的高度>=2 && 环的宽度至少也得是2，才会输出下面的那一行
        if (start<endY && start<endX){
            for (int i = endX; i >= start; i--) {
                System.out.print(numbers[endY][i]+" ");
            }
        }

        //4. 从下到上打印一行
        //  环的高度至少是3 && 宽度>=2
        if (start<endY-1 && start<endX){
            for (int i = endY-1; i >= start; i--) {
                System.out.print(numbers[i][start]+" ");
            }}}
```

### 包含min函数的栈

```java
 * 题目： 定义栈的数据结构，请在该类型中实现一个能够得到栈的最小素的 min 函数。
 * 在该栈中，调用 min、push 及 pop 的时间复杂度都是 0(1)
 * 【解】: 在push()的时候，再用一个辅助栈，来存放当前最小值

    public static class StackWithMin<T extends Comparable<T>>{
        private Stack<T> dataStack;
        private Stack<Integer> minStack;    //放的是最小元素的位置
        public StackWithMin(){
            this.dataStack = new Stack<>();
            this.minStack = new Stack<>();
        }

        public void push(T t){
            if (t == null){
                throw new RuntimeException("can not push null");
            }
            if (dataStack.isEmpty()){
                dataStack.push(t);
                minStack.push(0);
            } else {
                dataStack.push(t);
                T e = dataStack.get(minStack.peek());
                minStack.push(t.compareTo(e)<0? dataStack.size()-1: minStack.peek());
            }
        }

        public T pop(){
            if (dataStack.isEmpty()){
                throw new RuntimeException("Stack is empty!");
            }
            minStack.pop();
            return dataStack.pop();
        }

        public T min(){
            if (minStack.isEmpty()){
                throw new RuntimeException("Stack is empty!");
            }
            return dataStack.get(minStack.peek());}}
```

### 栈的压入$弹出序列

```java
* 题目: 输入两个整数序列，第一个序列表示栈的压入顺序，请判断二个序列是否为该栈的弹出顺序。假设压入栈的所有数字均不相等
* 例如序列1、2、3 、4、5 是某栈压栈序列，
*     序列4、5、3、2、1 是该压栈序列对应的一个弹出序列，
*       但4、3、5、1、2 就不可能是该压棋序列的弹出序列。
*
    private static boolean isPopOrder(int[] push, int[] pop) {
        if (push==null || pop==null || push.length==0  || pop.length==0 || push.length!=pop.length){
            return false;
        }
        int pushIndex = 0;
        int popIndex = 0;
        Stack<Integer> stack = new Stack<>();
        while (popIndex<pop.length){
            while (pushIndex<push.length && (stack.isEmpty() || pop[popIndex]!=stack.peek())){
                stack.push(push[pushIndex++]);
            }
            if (pop[popIndex] == stack.peek()){
                stack.pop();
                popIndex++;
            } else {
                return false;
            }
        }
        return true;}}
```

### 从上往下打印二叉树
Easy

```java
    private static void printFromToBottom(BinaryTreeNode head) {
        if (head == null){
            return;
        }
        Queue<BinaryTreeNode> queue = new LinkedList<>();
        queue.add(head);
        BinaryTreeNode curNode = null;
        while (!queue.isEmpty()){
            curNode = queue.remove();
            System.out.print(curNode.value + " ");
            if (curNode.left != null){
                queue.add(curNode.left);
            }
            if (curNode.right != null){
                queue.add(curNode.right);
            }
        }
    }
```

### 二叉搜索树的后序遍历序列 

注意限定是二叉搜索树, 也就是左孩子节点的值小于 root, 右孩子节点的值大于 root

```java
 * 题目：输入一个整数数组，判断该数组是不是某二叉搜索树的后序遍历的结果。如果是则返回 true。否则返回 false。
 * 假设输入的数组的任意两个数字都互不相同。

    private static boolean verifySequenceOfBST(int[] squence) {
        if (squence == null || squence.length<=0){
            return false;
        }
        return verifySequenceOfBST(squence, 0, squence.length-1);
    }

    private static boolean verifySequenceOfBST(int[] squence, int start, int end) {
        if (start >= end){
            return true;
        }
        
        int root = squence[end];    //end位置为根结点
        int index = 0;
        //找到第一个比根结点大的数的位置
        while (squence[index]<root && index < end-1){
            index++;
        }
        
        int mid = index;
        while (squence[index]>root && index<end-1){
            index++;
        }
        if (index != end-1){    //end-1位置应该为root的右子树的根结点
            return false;   //出现了异常，无法完成此出站序列
        }
        return verifySequenceOfBST(squence,start,mid-1) && verifySequenceOfBST(squence,mid,end-1);
    }
```

### 二叉树中和为某一值的路径

搜索算法

```java

 * 题目：输入一棵二叉树和一个整数， 打印出二叉树中结点值的和为输入整数的所有路径。
 * [从树的根结点开始往下一直到叶结点所经过的结点形成一条路径]

    private static void findPath(BinaryTreeNode head, int expectedSum) {
        if (head == null){
            return;
        }
        
        List<Integer> path = new ArrayList<>();
        int currentSum = 0;
        findPath(head, expectedSum, path, currentSum);
    }

    private static void findPath(BinaryTreeNode head, int expectedSum, List<Integer> path, int currentSum) {
        currentSum += head.value;
        path.add(head.value);
        if (head.left==null && head.right==null && currentSum == expectedSum){
            System.out.println(path);
        }
        if (head.left != null){
            findPath(head.left, expectedSum, path, currentSum);
        }
        if (head.right != null){
            findPath(head.right, expectedSum, path, currentSum);
        }
        path.remove(path.size()-1);
    }
```

### 复杂链表的复制

```java
 * 题目：请实现函数 ComplexListNode clone(ComplexListNode head)，复制一个复杂链表。
 * 在复杂链表中，每个结点除了有一个 next 域指向下一个结点外，还有一个 sibling 指向链表中的任意结点或者 null。
 *  【解】：
 *  法一：
 *      用HashMap保存(1,1.value)、(2,2.value)、...之间的对应关系
 *
 *  法二：
 *      先将链表化成 1->1'->2->2'->...

    /**
     * 法2： 1->1'->2->2'->...
     *
     * @param head
     * @return
     */
    private static ComplexListNode clone2(ComplexListNode head) {
        if (head == null) {
            return null;
        }

        ComplexListNode cur = head;
        ComplexListNode curCopy = null;
        while (cur != null) {
            curCopy = new ComplexListNode(cur.value);
            curCopy.sibling = cur.sibling;
            curCopy.next = cur.next;
            cur.next = curCopy;
            cur = curCopy.next;
        }

        cur = head;
        ComplexListNode clone = null;
        ComplexListNode res = cur.next;
        while (cur != null) {   //将clone链表从原链表中剥离出来，并恢复原链表
            clone = cur.next;
            clone.next = cur.next.next;
            cur.next = clone.next;
            cur = cur.next;
        }
        return res;
    }
```

### 二叉搜索树与双向链表

```java
 * 题目：输入一棵二叉搜索树，将该二叉搜索树转换成一个排序的双向链表。
 * 要求不能创建任何新的结点，只能调整树中结点指针的指向。
 *
 * [解]: 中序遍历

     * 书上解法：
     *     利用一个二级指针，保存每次已经处理好的当前链表的尾结点
    
    private static BinaryTreeNode convert2(BinaryTreeNode head) {
        if (head == null){
            return null;
        }
        BinaryTreeNode[] last = new BinaryTreeNode[1];
        convertNode(head, last);
        head = last[0];
        while (head!=null && head.left!=null){
            head = head.left;
        }
        return head;
    }

    private static void convertNode(BinaryTreeNode head, BinaryTreeNode[] last) {
        if (head == null){
            return;
        }
        if (head.left != null){
            convertNode(head.left, last);
        }
        head.left = last[0];   //设置当前结点的前驱结点

        if (last[0] != null) {
            last[0].right = head;  //设置尾结点的后继
        }
        last[0] = head; //记录当前结点为已处理好的双向链表的尾结点

        if (head.right != null){
            convertNode(head.right, last);
        }}
```

### 字符串的排列与组合

递归解法, 应该是随手就能写出来的

考虑出现重复的情况应该怎么处理

```java
    private static void permutation(String str) {
        if (str.isEmpty()){
            return;
        }
        char[] chars = str.toCharArray();
        permutation(chars, 0);
    }

    private static void permutation(char[] chars, int begin) {
        if (chars==null || chars.length<=0){
            return;
        }

        int length = chars.length;
        if (begin == length-1){
            System.out.print(new String(chars) + " ");
        } else {
            for (int i = begin; i < length; i++) {
                if (IsSwap(chars, begin, i)){
                    std.swap(chars, begin, i);
                    permutation(chars, begin+1);
                    std.swap(chars, begin, i);
                }}}}
```

```java
 * 题目：输入一个字符串，打印出该字符串中字符的所有组合。
 * 例如 输入字符串 abc。（输入字符不重复）
 *      输出：a、b、c、ab、ac、bc、abc、

    private static void combination(String str) {
        if (str.isEmpty()){
            return;
        }
        char[] chars = str.toCharArray();
        StringBuffer output = new StringBuffer(chars.length);

        combination(chars, chars.length, output, 0);
    }

    private static void combination(char[] chars, int length, StringBuffer output, int begin) {
        if (begin == length){
            return;
        } else {
            for (int i = begin; i < length; i++) {
                output.append(chars[i]);
                System.out.print(output.toString() + " ");
                combination(chars, length, output, i+1);
                output.deleteCharAt(output.length()-1);
            }}}
```

### 数组中出现次数超过一半的数字

每次拿出两个丢弃

```java
    private static int MoreThanHalfNum2(int[] arr) {
        if (arr==null || arr.length<=0){
            throw new RuntimeException("args should not be null or empty");
        }
        int result = arr[0];
        int times = 1;
        for (int i = 1; i < arr.length; i++) {
            if (result == arr[i]){
                times++;
            } else {
                if (times == 0){
                    result = arr[i];
                    times = 1;
                } else {
                    times--;
                }
            }
        }
        if (!checkMoreThanHalf(arr, result)){
            throw new RuntimeException("not exist");
        }
        return result;
    }
```

### 最小的k个数

```java
 * 题目： 输入n个整数，找出其中最小的k个数。
 * 例如输入 4 、5 、1、6、2、7、3 、8 这 8 个数字，
 *    则最小的 4 个数字是 1 、2、3 、4

 * 【法1】：O（N）(改变了数组原有的顺序)
 *     基于上一题的Partition函数：
 *     以第k个数为基准，从头开始遍历，
 *      使得比第k个数字小放在arr[k]的左边，比第k个数字大的放在右边
 *
 * 【法2】：O（N*logk) 基于堆排的思想
 *      先建立一个含有k个元素的大顶堆，
 *      每次新来一个数 和堆顶比较，如果大于堆顶，跳过
 *                               如果小于堆顶，交换，然后重新调整二叉堆
 *      直到arr结束
法1:
    private static void getLeastNumbers(int[] arr, int k) {
        if (arr==null || arr.length<=0 || k>arr.length){
            throw new IllegalArgumentException("Invalid args");
        }

        int index = partition(arr, 0, arr.length-1);
        while (index != k-1){
            if (index > k-1){
                index = partition(arr, 0, index-1);
            } else {
                index = partition(arr, index+1, arr.length-1);
            }
        }

        for (int i = 0; i < k; i++) {
            System.out.print(arr[i] + ", ");
        }
    }

法2:
    private static void getLeastNumbers2(int[] arr, int k) {
        if (arr==null || arr.length<=0 || k>arr.length){
            throw new RuntimeException("Invalid args");
        }

        int[] output = new int[k];
        for (int i = 0; i < k; i++) {
            output[i] = arr[i];
        }
        //建立大顶堆
        int N = k-1;
        for (int i = k/2; i >= 0; i--) {    //从 从右像左数 第一个含有子节点的非叶子结点开始调整
            sink(output, i, N);
        }
        
        //下面开始调整堆
        int i = k;
        while (i<arr.length){
            if (arr[i]<output[0]){
                output[0] = arr[i];
                sink(output, 0, N);
            }
            i++;
        }

        stdOut.print(output);
    }

    private static void sink(int[] arr, int k, int N) {
        while (2*k < N){
            int j = 2*k + 1;
            if (j<N && arr[j]<arr[j+1]){
                j++;    //arr[j]永远指向 a[k]的大儿子
            }
            if (arr[k]>arr[j]){
                break;  //a[k]比他的大儿子还大，无需调整
            } else {
                std.swap(arr, k, j);
                k = j;
            }
        }
    }
```

### 连续子数组的最大和

```java
 * 题目：输入一个整型数组，数组里有正数也有负数。数组中一个或连续的多个整数组成一个子数组。求所有子数组的和的最大值。
 * 要求时间复杂度为 O(n)。

     cur 依次累加各个元素，一旦cur为负数时，则将cur清为零。
     并尝试更新一次 result（最大值）
     最终返回 result 即为 两个无重合部分的子数组的最大累加和
     解释：
     因为最大和的子数组：其任意数量的前缀一定不为负。
     也就是 cur 如果没有累加出到负数，就继续往下走。   即模拟了“前缀不可能为负数”的情况。
    
    private static int findGreatestSumOfSubArray(int[] arr) {
        if (arr==null || arr.length<0){
            return 0;
        }

        int cur = arr[0];
        int res = cur;
        for (int i = 1; i < arr.length; i++) {
            cur += arr[i];
            res = Math.max(res, cur);
            cur = cur>0? cur:0;
        }
        return res;
    }
```

### 从1到n整数中1出现的次数

比较难, 不看了

```java
 * 题目：输入一个整数 n 求从 1 到 n 这 n 个整数的十进制表示中 1 出现的次数。
 * 例如输入 12 ，从 1 到 12 这些整数中包含 1 的数字有 1、10、11 和 12，1 一共出现了 5 次。
 *
 * 【解】：从数字的规律着手
 *  假设N = 54687
 *  1. 我们可以把N分成两部分：0-4687；4688-54687
 *      0-4687：继续分成两部分...（递归）
 *  2. 对于4688-54687：
 *     我们同样可以分成两部分来求：
 *      - 10000-19999  【只求最高位上的1的数目共(10^4)个】
 *      - 4688-9999，10000-19999(不看最高位上的这个1了)，20000-54687 ==> 5 个 0-9999
 *                     【1个次数：5* 4*10^3】
```

### 把数组排成最小的数

```java
 * 题目：输入一个正整数数组，把数组里所有数字拼接起来排成一个数，打印能拼接出的所有数字中最小的一个。
 * 例如输入数组{3， 32, 321}，
 * 则扫描输出这 3 个数字能排成的最小数字 321323。
 *【解】：自定义排序规则，
 *      两个数字 m和n
     *      mn>nm： m>n
     *      mn==nm：m=n
     *      mn<nm： m<n
 *      但是这里隐含这一个大数问题，（mn拼接之后有可能超出int范围）
 *      所以，我们可以把数值的比较大小 直接转化为字符串的compare

    static class StringComparator implements Comparator<String>{
        @Override
        public int compare(String str1, String str2) {
            if (str1==null || str2==null){
                throw new IllegalArgumentException("args should not be null or empty");
            }

            return (str1+str2).compareTo(str2+str1);
        }
    }

    private static void quickSort(String[] arr, int lo, int hi, StringComparator comparator) {
        if (lo >= hi){
            return;
        }
        //partition
        int flag = lo;
        for (int i = lo; i < hi; i++) {
            if (comparator.compare(arr[i], arr[hi]) < 0){
                std.swap(arr, i, flag++);
            }
        }
        std.swap(arr, hi, flag);
        quickSort(arr, lo, flag-1, comparator);
        quickSort(arr, flag+1, hi, comparator);
    }
```

### 丑数

```java
 * 题目：我们把只包含因子 2、3 和 5 的数称作丑数（Ugly Number）。求从小到大的顺序的第 1500 个丑数
 * 例如 6、8 都是丑数，但 14 不是，它包含因子 7。习惯上我们把 1 当做第一个丑数。

 * 【解】：用空间换时间
     * 假设当前已经求到了第M个UglyNumber
     * 则第M+1个丑数一定是前面的某个数字 *2、*3、*5的最小值！
     * 是谁，谁对应的指针 ++，
     * 其他的指针也要跟进到 *2、*3、*5之后的值 <= UglyNumber[M+1]!!
 *

    private static int getUglyNumber(int n) {
        if (n<=0){
            return 0;
        }
        int[] UglyNumbers = new int[n];
        UglyNumbers[0] = 1; //第1个丑数是1
        int nextIndex = 1;
        
        int p2 = 0;
        int p3 = 0;
        int p5 = 0;

        while (nextIndex < n){
        
            int min = min(UglyNumbers[p2]*2, UglyNumbers[p3]*3, UglyNumbers[p5]*5);
        
            UglyNumbers[nextIndex] = min;
        
            while (UglyNumbers[p2]*2 <= UglyNumbers[nextIndex]){
                p2++;
            }
            while (UglyNumbers[p3]*3 <= UglyNumbers[nextIndex]){
                p3++;
            }
            while (UglyNumbers[p5]*5 <= UglyNumbers[nextIndex]){
                p5++;
            }
            nextIndex++;
        }
        return UglyNumbers[--nextIndex];
    }
```


