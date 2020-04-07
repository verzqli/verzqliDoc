#### 题目

> 定义栈的数据结构，请在该类型中实现一个能够得到栈的最小元素的min函数。在该栈中，调用min、push及pop的时间复杂度都是0(1)

#### 分析

栈的特效是先进后出，如果先前存进去最小值想要获得就需要再来一个变量来保存这个最小元素，但是再弹出一个最小元素后，我们需要得到次一个元素，因此，不仅要保存最小元素，次一级小的元素也要保存。

所以方案是创建一个辅助栈，每次往栈中压入一个数据时，判断当前数据是否比栈中最小值小，如果是，就压入赋值栈，如果不是，就再压一个当前最小值到辅助栈中

| 步骤 | 操作        | 数据站  | 辅助栈  | 最小值 |
| ---- | ----------- | ------- | ------- | ------ |
| 1    | 压入3       | 3       | 3       | 3      |
| 2    | 压入4       | 3，4    | 3，3    | 3      |
| 3    | 压入1       | 3，4，1 | 3，3，1 | 1      |
| 4    | 弹出（pop） | 3，4    | 3，3    | 3      |
| 5    | 弹出（pop） | 3       | 3       | 3      |
| 6    | 压入2       | 3，2    | 3，2    | 2      |

```java
 Stack<Integer> dataStack = new Stack<>();
    Stack<Integer> minStack = new Stack<>();

    public static void main(String[] args) {

    }

    public void push(int x) {
        if (dataStack.empty()) {
            dataStack.push(x);
            minStack.push(x);
        } else {
            int min = minStack.peek();
            dataStack.push(x);
            minStack.push(Math.min(x, min));
        }
    }

    public void pop() {
        dataStack.pop();
        minStack.pop();
    }

    public int top() {
        return dataStack.peek();
    }

    public int min() {
        return minStack.peek();
    }
```

