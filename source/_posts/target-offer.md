---
title: 剑指 offer——Python 实现
date: 2018-06-11 09:38:24
tags: Python
categories: 编程
---

# 题 2：单例模式



python 中的单例模式有四种实现方式：

1. 模块
2. 使用 `__new__` 方法
3. 使用装饰器
4. 使用元类

这里解释使用 `__new__` 方法的实现。

```python
# coding:utf-8

# 单例模式
class Singleton(object):
    _instance = None

    def __new__(cls, *args, **kwargs):
        if not isinstance(cls._instance, cls):
            # cls._instance = super(Singleton, cls).__new__(cls, *args, **kwargs)
            cls._instance = object.__new__(cls, *args, **kwargs)
        return cls._instance


# 测试
class Myclass(Singleton):
    a = 1


A = Myclass()
B = Myclass()

print id(A), id(B)

```

# 题 3：数组中重复的数字

## 1. 找出数组中重复的数字

```python
# -*- coding:utf-8 -*-
class Solution:
    # 这里要特别注意~找到任意重复的一个值并赋值到duplication[0]
    # 函数返回True/False
    def duplicate(self, numbers, duplication):
        if numbers is None or len(numbers) <= 0:
            return False
        else:
            for i in numbers:
                if i < 0 or i > len(numbers) - 1:
                    return False
            for i in range(len(numbers)):
                while numbers[i] != i:
                    if numbers[i] == numbers[numbers[i]]:
                        duplication[0] = numbers[i]
                        return True
                    else:
                        index = numbers[i]
                        numbers[i], numbers[index] = numbers[index], numbers[i]
            return False
```

## 2. 找出数组中重复的数字（不修改数组）

```python
# coding:utf-8


class Solution:

    def count_range(self, numbers, length, start, end):
        if numbers is None:
            return False
        count = 0
        for i in range(0, length):
            if start <= numbers[i] <= end:
                count += 1
        return count  # 返回区间的数目

    def getDuplication(self, numbers, length):
        if numbers is None or len(numbers) <= 0:
            return False
        else:
            start = 1  # 1
            end = length - 1  # n
            while end >= start:
                middle = ((end - start) >> 1) + start
                count = self.count_range(numbers, length, start, middle)
                if end == start:
                    if count > 1:
                        return start
                    else:
                        break
                if count > (middle - start + 1):
                    end = middle
                else:
                    start = middle + 1
            return False


# 测试
if __name__ == '__main__':
    num = [2, 3, 5, 4, 3, 2, 6, 7]
    s = Solution()
    print s.getDuplication(num, 8)
```

# 题 4：二维数组中的查找

```Python
# -*- coding:utf-8 -*-


class Solution:
    # array 二维列表
    def Find(self, matrix, rows, columns, number):
        # write code here
        found = False

        if matrix is not None and rows > 0 and columns > 0:
            row = 0
            column = columns - 1
            while row < rows and column >= 0:
                if matrix[row][column] == number:
                    found = True
                    break
                elif matrix[row][column] > number:
                    column -= 1
                else:
                    row += 1

        return found

# 测试
if __name__ == '__main__':
    matrix = [[1, 2, 8, 9],
              [2, 4, 9, 12],
              [4, 7, 10, 13],
              [6, 8, 11, 15]]
    s = Solution()
    print s.Find(matrix, 4, 4, 7)
    
    
    
# 牛客网中写法

# -*- coding:utf-8 -*-
class Solution:
    # array 二维列表
    def Find(self, number, matrix):
        # write code here
        found = False
        rows = len(matrix)
        columns = len(matrix[0])
        if matrix is not None and rows > 0 and columns > 0:
            row = 0
            column = columns - 1
            while row < rows and column >= 0:
                if matrix[row][column] == number:
                    found = True
                    break
                elif matrix[row][column] > number:
                    column -= 1
                else:
                    row += 1
        return found
```

# 题 5：替换空格

```python
# -*- coding:utf-8 -*-

class Solution:
    # s 源字符串
    def replaceSpace(self, s):
        # write code here
        s_list = list(s)
        s_replace = []
        for item in s_list:
            if item == ' ':
                s_replace.append('%')
                s_replace.append('2')
                s_replace.append('0')
            else:
                s_replace.append(item)
        return "".join(s_replace)
```

# 题 6：从尾到头打印链表

```python
class ListNode:
    def __init__(self, x=None):
        self.val = x
        self.next = None

# 单链表的一个实现函数
listnode = ListNode(1)
p = listnode
for i in xrange(2,11):
    listnode.next = ListNode(i)
    listnode = listnode.next
while p:
    print p.val
    p = p.next

# 利用栈
class Solution:
    def printListFromTailToHead(self, listNode):
        if listNode.val is None:
            return
        l = []
        head = listNode
        while head:
            l.insert(0, head.val)
            head = head.next
        return l


s = Solution()
print s.printListFromTailToHead(p)
```

# 题 7：重建二叉树

```python
# coding:utf-8

'''
输入某二叉树的前序遍历和中序遍历的结果，请重建出该二叉树。
假设输入的前序遍历和中序遍历的结果中都不含重复的数字。
例如输入前序遍历序列{1,2,4,7,3,5,6,8}和中序遍历序列{4,7,2,1,5,3,8,6}，则重建二叉树并返回。
'''


# -*- coding:utf-8 -*-
class TreeNode:
    def __init__(self, x):
        self.val = x
        self.left = None
        self.right = None


class Solution:
    # 返回构造的TreeNode根节点
    def reConstructBinaryTree(self, pre, tin):
        # write code here
        if not pre and not tin:
            return None
        root = TreeNode(pre[0])
        if set(pre) != set(tin):
            return None
        i = tin.index(pre[0])
        root.left = self.reConstructBinaryTree(pre[1:i + 1], tin[:i])
        root.right = self.reConstructBinaryTree(pre[i + 1:], tin[i + 1:])
        return root


pre = [1, 2, 3, 5, 6, 4]
tin = [5, 3, 6, 2, 4, 1]
test = Solution()
newTree = test.reConstructBinaryTree(pre, tin)


# 按层序遍历输出树中某一层的值
def PrintNodeAtLevel(treeNode, level):
    if not treeNode or level < 0:
        return 0
    if level == 0:
        print(treeNode.val)
        return 1
    PrintNodeAtLevel(treeNode.left, level - 1)
    PrintNodeAtLevel(treeNode.right, level - 1)


# 已知树的深度按层遍历输出树的值
def PrintNodeByLevel(treeNode, depth):
    for level in range(depth):
        PrintNodeAtLevel(treeNode, level)

```

# 题 8：二叉树的下一个节点

```python
'''
给定一个二叉树和其中的一个结点，请找出中序遍历顺序的下一个结点并且返回。
注意，树中的结点不仅包含左右子结点，同时包含指向父结点的指针。
'''
# -*- coding:utf-8 -*-
class TreeLinkNode:
    def __init__(self, x):
        self.val = x
        self.left = None
        self.right = None
        self.next = None
class Solution:
    def GetNext(self, pNode):
        if pNode == None:
            return
        pNext = None
        if pNode.right != None:
            pRight = pNode.right
            while pRight.left != None:
                pRight = pRight.left
            pNext= pRight
        elif pNode.next != None:
            pCurrent = pNode
            pParent = pCurrent.next
            while pParent != None and pCurrent == pParent.right:
                pCurrent = pParent
                pParent = pCurrent.next
            pNext = pParent
        return pNext

class Solution2:
    def GetNext(self, pNode):
        # 输入是一个空节点
        if pNode == None:
            return None
        # 注意当前节点是根节点的情况。所以在最开始设定pNext = None, 如果下列情况都不满足, 说明当前结点为根节点, 直接输出None
        pNext = None
        # 如果输入节点有右子树，则下一个结点是当前节点右子树中最左节点
        if pNode.right:
            pNode = pNode.right
            while pNode.left:
                pNode = pNode.left
            pNext = pNode
        else:
            # 如果当前节点有父节点且当前节点是父节点的左子节点, 下一个结点即为父节点
            if pNode.next and pNode.next.left == pNode:
                pNext = pNode.next
            # 如果当前节点有父节点且当前节点是父节点的右子节点, 那么向上遍历
            # 当遍历到当前节点为父节点的左子节点时, 输入节点的下一个结点为当前节点的父节点
            elif pNode.next and pNode.next.right == pNode:
                pNode = pNode.next
                while pNode.next and pNode.next.right == pNode:
                    pNode = pNode.next
                # 遍历终止时当前节点有父节点, 说明当前节点是父节点的左子节点, 输入节点的下一个结点为当前节点的父节点
                # 反之终止时当前节点没有父节点, 说明当前节点在位于根节点的右子树, 没有下一个结点
                if pNode.next:
                    pNext = pNode.next
        return pNext
```

# 题 9：用两个栈实现队列

```python
# -*- coding:utf-8 -*-
class Solution:
    def __init__(self):
        self.stack1 = []
        self.stack2 = []

    def push(self, node):
        self.stack1.append(node)
    # write code here
    def pop(self):
        if len(self.stack2) == 0 and len(self.stack1) == 0:
            return
        elif len(self.stack2) == 0:
            while len(self.stack1) > 0:

                self.stack2.append(self.stack1.pop())
        return self.stack2.pop()
# return xx
```

# 题 10：斐波那契数列

```python
# -*- coding:utf-8 -*-
class Solution:
    def Fibonacci(self, n, cache=None):
        if cache is None:
            cache = {}
        if n in cache:
            return cache[n]
        if n == 0:
            return 0
        if n == 1:
            return 1

        cache[n] = self.Fibonacci(n - 1, cache) + self.Fibonacci(n - 2, cache)
        return cache[n]
```

# 题 11：旋转数组的最小数字

```python
# coding:utf-8

class Solution:
    def minNumberInRotateArray(self, array):
        if len(array) == 0:
            return 0
        head, end = 0, len(array) - 1
        mid = 0
        while array[head] >= array[end]:
            if end - head == 1:
                mid = end
                break
            mid = (end + head) // 2
            if array[head] == array[end] and array[head] == array[mid]:
                return self.order_min(array, head, end)
            if array[mid] >= array[head]:
                head = mid
            elif array[mid] <= array[end]:
                end = mid
        return array[mid]

    def order_min(self, array, head, end):
        result = array[0]
        for i in range(len(array)):
            if i < result:
                result = i
        return result
```

# 题 12：矩阵中的路径

```python
# -*- coding:utf-8 -*-
class Solution:
    def hasPath(self, matrix, rows, cols, path):
        if matrix == None or rows < 1 or cols < 1 or path == None:
            return False
        visited = [0] * (rows * cols)

        pathLength = 0
        for row in range(rows):
            for col in range(cols):
                if self.hasPathCore(matrix, rows, cols, row, col, path, pathLength, visited):
                    return True
        return False

    def hasPathCore(self, matrix, rows, cols, row, col, path, pathLength, visited):
        if len(path) == pathLength:
            return True

        hasPath = False
        if row >= 0 and row < rows and col >= 0 and col < cols and matrix[row * cols + col] == path[pathLength] and not \
        visited[row * cols + col]:

            pathLength += 1
            visited[row * cols + col] = True

            hasPath = self.hasPathCore(matrix, rows, cols, row, col - 1, path, pathLength, visited) or \
                      self.hasPathCore(matrix, rows, cols, row - 1, col, path, pathLength, visited) or \
                      self.hasPathCore(matrix, rows, cols, row, col + 1, path, pathLength, visited) or \
                      self.hasPathCore(matrix, rows, cols, row + 1, col, path, pathLength, visited)

            if not hasPath:
                pathLength -= 1
                visited[row * cols + col] = False

        return hasPath
```

# 题 13：机器人的运动范围

```python
# -*- coding:utf-8 -*-
class Solution:
    def movingCount(self, threshold, rows, cols):
        if threshold < 0 or rows <= 0 or cols <= 0:
            return False
        visited = [False] * (rows * cols)
        count = self.movingCountCore(threshold, rows, cols, 0, 0, visited)
        return count

    def movingCountCore(self, threshold, rows, cols, row, col, visited):
        count = 0
        if self.check(threshold, rows, cols, row, col, visited):
            visited[row * cols + col] = True
            count = 1 + self.movingCountCore(threshold, rows, cols, row, col - 1, visited) + \
                    self.movingCountCore(threshold, rows, cols, row, col + 1, visited) + \
                    self.movingCountCore(threshold, rows, cols, row - 1, col, visited) + \
                    self.movingCountCore(threshold, rows, cols, row + 1, col, visited)
        return count

    def check(self, threshold, rows, cols, row, col, visited):
        if row >= 0 and row <= rows and col >= 0 and col <= cols and self.getDigitSum(row) + self.getDigitSum(
                col) <= threshold and not visited[row * cols + col]:
            return True
        return False

    def getDigitSum(self, number):
        sum = 0
        while number > 0:
            sum += number % 10
            number = number // 10
        return sum
```

# 题 14：剪绳子

```python
# coding:utf-8


class Solution:
    def max_product_after_cutting1(self, length): # 使用动态规划，自下而上解决
        if length < 2:
            return 0
        if length == 2:
            return 1
        if length == 3:
            return 2
        products = [0] * (length + 1)
        products[0] = 0
        products[1] = 1
        products[2] = 2
        products[3] = 3
        for i in range(4, length + 1):
            max = 0
            for j in range(1, i / 2 + 1):
                product = products[j] * products[i - j]
                if max < product:
                    max = product
                products[i] = max
        max = products[length]
        return max

    def max_product_after_cutting2(self, length):  # 贪婪算法
        if length < 2:
            return 0
        if length == 2:
            return 1
        if length == 3:
            return 2
        timesof3 = length / 3
        if length - timesof3 * 3 == 1:
            timesof3 -= 1
        timesof2 = (length - timesof3 * 3) / 2
        max = (3 ** timesof3) * (2 ** timesof2)
        return max


if __name__ == '__main__':
    s = Solution()
    result1 = s.max_product_after_cutting1(8)
    result2 = s.max_product_after_cutting2(8)
    print result1, result2
```

# 题 15：二进制中 1 的个数

```python
# -*- coding:utf-8 -*-
class Solution:
    def NumberOf1(self, n):  # 死循环
        # n = bin(n)
        count = 0
        while n:
            if n & 1:
                count += 1
            n = n >> 1
        return count

    def Numberof1_2(self, n):
        l = len(bin(n))
        count = 0
        flag = 1
        for i in range(l):
            if n & flag:
                count += 1
            flag = flag << 1
        return count

    def Numberof1_3(self, n):  # 最佳解法
        count = 0
        while n:
            count += 1
            n = (n - 1) & n
        return count

    def Numberof1_4(self, n):
        if n < 0:
            s = bin(n & 0xffffffff)
        else:
            s = bin(n)
        return s.count('1')


if __name__ == '__main__':
    s = Solution()
    result1 = s.NumberOf1(9)
    result2 = s.Numberof1_2(78787)
    result3 = s.Numberof1_3(989798798798)
    result4 = s.Numberof1_4(9)
    print result1, result2, result3, result4
```