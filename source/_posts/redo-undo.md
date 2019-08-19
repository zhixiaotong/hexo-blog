---
title: 基于链表的前进撤销实现思路
date: 2019-08-16 17:29:38
tags: ["Javascript"]
category: "前端大爆炸"
thumbnail: /gallery/thumbnails/lianbiao.jpg
copyright: true
---
## 基于链表的考虑
在工作中，处理一组结构类似的数据，往往会采用**数组**的方式。**数组**和**链表**其实都可以作为存储类型类似的数组的选择。为什么要选择链表呢？
<!-- more -->
### 数组：
- 优点：数据连续存放在内存中，通过下标获取访问方便，查询效率较高。
- 缺点：由于是顺序存储，添加或删除元素，需要操作移动其他元素，效率较低，可能会导致内存浪费。
###链表：
- 优点：通过指针的移动可以快速添加或删除元素，效率较高。
- 缺点：查找会根据指针从头遍历，无法通过下标快速定位，查询效率较低。

> 二者的关系犹如**矛**与**盾**，你唯一的缺点就是没有我......

前进与撤销需要多次对数据进行添加和删除，因此在实践中，我选择了链表作为存储数据的结构。当然，如果是固定存储长度的实现，比如：只存储10次操作，多于的操作废弃，这种情况下，数组长度不至于过大，也是可以通过数组实现此类工的。
## 链表选择
链表分为单向链表和双向链表，由于前进后撤是需要前后对象的查找，因此选择双向链表。在查找上，双向链表可通过二分法的方式，头尾节点同时查找提高查询效率。
## 实现思路
双向链表的节点只有数据、指向前一个节点的指针和指向后一个节点的指针，在这里我声明Node作为链表的单个节点，Stack记录链表对象的状态。
```javascript
function Node(ele) {
    this.element = ele; // 存储的数据
    this.prev = null; // 前驱指针：指向前一个Node
    this.next = null; // 后继指针：指向后一个Node
}

class Stack() {
    constructor(maxLen) {
        this.maxLen = maxLen || 10; // 链表最大长度，如不加长度限制则可以不写
        this.length = 0; // 当前链表长度
        this.head = null; // 头结点指针
        this.tail = null; // 尾节点指针
        this.finger = 0; // 指针
    }
    ...
}
```
## 前进撤销的链表操作
> 链表的增加、删除和清空操作，网上已经有大量代码讲述，代码类似，在此不会贴出全部代码，仅提供思路。
### 增加
增加分为**push**（追加）和**insert**（插入）
```javascript
const node = new Node(element); // 被插入链表的节点
```
- **push**：将现有链表的tail尾指针指向**node**，同时前一个节点的next指针指向**node**。如果链表长度为0，说明没有节点被插入，直接将头结点和尾节点均指向**node**。
```javascript
    class Stack(maxLen) {
        ...
        /**
         * @param element 存储的数据
         */
        push(element) {
            // 被插入链表的节点
            const node = new Node(element);
            let current;
            if (this.length === 0) {
                this.head = node;
                this.tail = node;
            } else {
                current = this.head
                // 循环，直至获取到当前链表的最后一个节点
                while(current.next) {
                    current = current.next
                }
                // 由于是双向链表，node的前置指针需要指向current，current的后继指针需要指向node
                current.next = node
                node.prev = current
                // node自然是tail尾节点
                this.tail = node
            }
            // 链表长度累加
            this.length++
            // 如果声明了maxLen，当length超出了maxLen，需要从链表头部开始截取第一个节点
            // if (this.length - 1 === this.maxLen) {
                // this.removeNode(0);
            // }
            // 指针变动
            this.finger = this.length - 1;
        }
        ...
    }
```
- **insert**：
插入需要指定元素被插入的**下标**，分为**头部插入**，**中间插入**和**尾部插入**三种情况，不管是哪种插入，在前进撤销操作中，被插入的node都将作为tail节点，并删除下标对应节点后续的其他节点。
- **头部插入**：前进撤销的头部插入，即将当前node同时指向头结点和尾节点，并将原head节点的next置空。
- **中间插入**：遍历获取到插入节点的current，将node的prev指向到current，将current的next指向node，将node的next置空。
- **尾部插入**：node节点的prev指向到原tail节点（若超出maxLen，则删除链表的第一个节点）。

以下是部分代码：
```javascript
    class Stack(maxLen) {
        ...
        /**
         * @param index 待插入的下标
         * @param element 存储的数据
         */
        insert(index, element) {
            // 防止越界
            if (position < 0 || position >= this.length) {
                throw new SyntaxError('插入元素下标越界')
            }
            // 被插入链表的节点
            const node = new Node(element);
            switch(index) {
                case 0:
                    // 头部插入
                    node.next = null;
                    this.head = node;
                    this.length = 1;
                    break;
                case this.length:
                    // 尾部插入
                    var tail = this.tail;
                    tail.next = node;
                    node.prev = tail;
                    // 累加长度
                    this.length++;
                    // 超出截取
                    // if (this.length - 1 === this.maxLen) {
                        // this.removeNode(0);
                    // }
                    break;
                default:
                    // 中间插入
                    var idx = 0;
                    var previous;
                    var current = this.head;
                    while (idx < index) {
                      idx++;
                      previous = current;
                      current = current.next;
                    }
                    // 添加节点之间的关联
                    previous.next = node;
                    current.prev = node;
                    node.prev = previous;
                    this.length = idx;
                    break;
              }
              // 不管是什么操作，tail节点都是node
              this.tail = node;
              // 指针变动
              this.finger = this.length -1;
          }
          ...
    }
```
### 删除
删除同修改一样，与修改不同的是，将tail置为null，如果删除下标为0，则清空链表。
### 清空
链表的清空操作很简单，只需要重置头尾节点，指针清零即可，javascript内存机制中垃圾回收机制会自动回收不再使用的数据内存，这使得我们不需要手动释放内存。
```javascript
    class Stack(maxLen) {
        ...
        // 清空
        empty() {
            this.head = null; // 重置头指针
            this.tail = null; // 重置尾指针
            this.length = 0; // 重置长度
            this.finger = 0; // 重置指针变动
        }
        ...
    }
```

> 前进撤销的基本思路是点击按钮切换Stack的finger指针下标，根据下标获取对应的Node节点，将节点中的数据返回到页面上重新渲染。

至此，基于链表的前进撤销基本操作完成了，undo（撤销）和redo（前进）改变finger指针的逻辑，是不是很容易实现呢？^ _ ^