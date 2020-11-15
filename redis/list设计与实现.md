## `list`设计与实现

#### `list`的定义

```c
//迭代器遍历方向
#define AL_START_HEAD 0
#define AL_START_TAIL 1
//链表节点结构
typedef struct listNode {
    struct listNode *prev;//前置节点地址
    struct listNode *next;//后置节点地址
    void *value;//节点的值
} listNode;
//链表迭代器结构
typedef struct listIter {
    listNode *next;//下一个节点地址
    int direction;//访问方向
} listIter;
//双向链表结构
typedef struct list {
    listNode *head;//头节点地址
    listNode *tail;//尾节点地址
    void *(*dup)(void *ptr);//节点value复制函数
    void (*free)(void *ptr);//节点value释放函数
    int (*match)(void *ptr, void *key);//节点value比较函数
    unsigned long len;//链表的长度
} list;
```

- 此处双向链表定义与一般的双向链表定义比较类似，不同点在于`listNode`结构中的`value`属性并不是确定的某一种数据类型，而是`void*`类型，这是因为`C`相比于`C++`而言没有提供模板来实现静态多态。要想`listNode`可以存储任意类型的数据，就需要`void*`类型指针指向实际的数据地址（`void*`指针可以指向任意类型），这样可以实现不同类型的链表或者同一链表存储不同类型的节点。
- 由于`listNode`中存储`value`的数据类型是不确定的，那么不同数据类型之间复制，释放，比较的方式也有所不同。`list`结构中的`dup`，`free`和`match`为函数指针，指定`value`属性复制，释放，比较的方式。
- `listIter`结构是`list`的迭代器，可以很方便的实现对链表的遍历，无需用户涉及详细的指针操作。
- 函数指针的解释：
  - `dup`指向的函数实现对`ptr`的复制，若复制成功则返回复制完成之后的节点地址;反之返回`NULL`
  - `free`指向的函数实现对`ptr`内存的释放，无返回值。
  - `match`指向的函数实现`ptr`和`key`的比较，相同返回非`0`，不同返回`0`

#### `list` API

##### `list`相关API

```c
#define listLength(l) ((l)->len)
#define listFirst(l) ((l)->head)
#define listLast(l) ((l)->tail)
//创建一个新的list
//返回值：如果创建成功则返回新建链表的地址;反之返回NULL
list *listCreate(void);

//使用一个链表来创建新链表
//如果设置了dup属性则使用dup函数来复制新节点;反之执行潜拷贝，两个节点value指针指向同一块内存
//返回值：如果复制成功则返回新建链表的地址;反之返回NULL
list *listDup(list *orig);

//释放整个链表，由调用者设置的free属性来释放value私有属性,如果没有设置free函数,可能会造成内存泄漏
//返回值：该函数肯定会执行成功
void listRelease(list *list);
```

##### `listNode`相关API

```c
#define listPrevNode(n) ((n)->prev)
#define listNextNode(n) ((n)->next)
#define listNodeValue(n) ((n)->value)

#define listSetDupMethod(l,m) ((l)->dup = (m))
#define listSetFreeMethod(l,m) ((l)->free = (m))
#define listSetMatchMethod(l,m) ((l)->match = (m))

#define listGetDupMethod(l) ((l)->dup)
#define listGetFree(l) ((l)->free)
#define listGetMatchMethod(l) ((l)->match)
//在链表的头部插入新节点value,value是指针,需要调用者在调用该函数之前手动为value分配内存
//返回值：若插入成功返回传入链表的地址;反之返回NULL，原先的链表不发生改变
list *listAddNodeHead(list *list, void *value);

//在链表的尾部插入新节点value,其余同头部插入一样
list *listAddNodeTail(list *list, void *value);

//在链表的old_node节点前或后插入新节点value,插入方向与after的正负相关
//返回值：若插入成功返回传入链表的地址;反之返回NULL，原先的链表不发生改变
list *listInsertNode(list *list, listNode *old_node, void *value, int after);

//在指定链表中删除指定节点
//由调用者设置的free属性来释放value私有属性,如果没有设置free函数，可能会造成内存泄漏
//返回值:该函数一定会执行成功
void listDelNode(list *list, listNode *node);

//在链表中使用match函数进行查询匹配，若没有设置match的话则key直接和value进行地址匹配
//返回值：匹配成功返回第一个匹配的节点地址;反之返回NULL
listNode *listSearchKey(list *list, void *key);
//根据索引在链表中查询节点
//index的正负表示查询方向,0表示第一个节点,-1表示最后一个节点
listNode *listIndex(list *list, long index);

//将list的尾部节点移动到头部
void listRotate(list *list);
```

##### `listIter`相关API

```c
//创建链表迭代器,direction指定迭代的方向
//返回值：如果创建成功则返回新建迭代器;反之返回NULL
listIter *listGetIterator(list *list, int direction);

//通过链表迭代器遍历整个链表
//返回值：链表遍历过程中的节点地址
//使用例子：
/*
 * iter = listGetIterator(list,<direction>);
 * while ((node = listNext(iter)) != NULL) {
 * 	doSomethingWith(listNodeValue(node));
 * }
*/
listNode *listNext(listIter *iter);

//重置迭代器指向list的头部位置
void listRewind(list *list, listIter *li);

//重置迭代器指向list的头部位置
void listRewindTail(list *list, listIter *li);

//释放迭代器内存空间
//返回值：该函数一定会执行成功
void listReleaseIterator(listIter *iter);
```

