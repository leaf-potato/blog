## `SDS`设计与实现

#### `SDS`的定义

```c
typedef char *sds;//指向buf地址
struct sdshdr {
    unsigned int len;//buf数组中已经使用的字节数量
    unsigned int free;//buf数组中未使用的字节数量
    char buf[];//字节数组，用于保存字符串。
};
```

- `sds`始终指向`struct sdshdr`中的`buf`数组，在使用过程中声明`sds`类型的变量即可，`struct sdshdr`对于用户来说是透明的，`sds`可以通过指针运算来找到`struct sdshdr`的地址（具体如何计算的请阅读下一点），从而实现访问`len`和`free`属性。
- 特别注意`struct sdshdr`中的`buf`属性，其是长度不确定的`char`数组（柔性数组，动态数组），而非`char*`类型，对`struct sdshdr`求`sizeof`大小的话，会发现无论`buf`数组为多大，其结果为常量，不会占用内存空间，通过`sds - sizeof(struct sdshdr)`表达式可以得到`struct sdshdr`的首地址。

#### `SDS`与`C`字符串的异同点

相同点：

- `sds`遵循`C`语言普通字符串风格，字符串总是会以`\0`字符结尾，但其并不计算在`len`属性中，这样既可以复用`C`语言中原有的字符串操作，同时不影响`sds`特有的功能。

不同点：

- `C`语言字符串需要调用`strlen()`函数遍历整个字符串，`sds`以空间换取时间，获取长度的只需要访问`len`即可，时间复杂度更低。
- `C`语言字符串在使用`strcat()`函数进行拼接操作之前需要手动分配足够的内存空间，否则可能会造成缓冲区溢出。`sds`的`sdscat()`封装了检测内存空间是否足够，自动分配内存等操作，预防了缓冲区溢出。这些操作对于用户来说都是透明的，用户只管调用`sdscat()`即可，无需手动管理内存。
- `C`语言字符串不可以预先分配内存，每次分配内存大小与字符串长度相关。在字符串长度变化频繁时，内存分配造成的开销将大大增大。而`sds`通过`free`属性来记录预先分配的内存，减少内存分配的开销。

#### 内存预分配和惰性释放

**内存预分配**

若在`sds`中添加新的字符串，当前内存空间不够时，`sds`会自动分配新的内存空间，分配大小满足一下策略：

- 若添加完新的字符串后，其长度（`len`属性）小于1024个字节时，其会分配`2*len`大小的内存空间（`free`属性和`len`属性的值相同）
- 若添加完新的字符串后，其长度（`len`属性）大于等于1024个字节时，其会分配`len + 1MB`大小的内存空间（`free`属性的值为1024）

**内存惰性释放**

当删除`sds`中的字符时，其所占用的内存空间并未真正释放给操作系统，调整`len`和`free`属性的值，用`free`属性记录这些空间的内存空间，以备将来使用。

#### `SDS` API

**创建`sds`相关API**

```c
//创建一个长度为initlen的sds
//若init不为空，则用init指向的长度为inilen的字符串进行初始化
//若init为空，则每个字节都初始化为0
sds sdsnewlen(const void *init, size_t initlen);
//创建一个用C语言字符串初始化的sds
sds sdsnew(const char *init);
//创建一个给定sds的副本
sds sdsdup(const sds s);
//创建一个空的sds
sds sdsempty(void);
//用整数来初始化sds
sds sdsfromlonglong(long long value);
```

**销毁sds相关API**

```c
//释放一个给定的sds
void sdsfree(sds s);
```

**`sds`内存相关API**

```c
//内存预分配，扩展sds的长度，确保sds的free空间大于addlen
//预分配大小按照上文所说的分配策略进行分配
sds sdsMakeRoomFor(sds s, size_t addlen);   
//移除sds中的空闲内存空间
sds sdsRemoveFreeSpace(sds s);
//计算sds总共占用的内存空间
//1.sds的header占用空间 
//2.字符串占用的空间
//3.预分配的空间
//4.'\0'占用的空间
size_t sdsAllocSize(sds s)
```

**`sds`拼接相关API**

```c
//将t指向的长度为len的字符串追加到sds中
//追加之后原有的sds失效，需要使用返回后的sds
sds sdscatlen(sds s, const void *t, size_t len)
//将C语言字符串追加到sds中
sds sdscat(sds s, const char *t); 
//将给定sds字符串追加到另一个sds字符串
sds sdscatsds(sds s, const sds t);
```

**`sds`复制相关API**

```c
//将t指向的长度为len的字符串复制到sds中，覆盖原来的sds
sds sdscpylen(sds s, const char *t, size_t len);
//将C语言字符串复制到sds中
sds sdscpy(sds s, const char *t)
```

**字符串和整数转换相关API**

```c
//将整数value转换为字符串并存储到s指向的内存空间中
//s指向的内存空间长度至少为21
//返回s字符串的长度
int sdsll2str(char *s, long long value);
//与sdsll2str功能类似，只是针对unsigned long long 类型
int sdsull2str(char *s, unsigned long long v);
```



```c
//返回sds的len属性，即当前字符串的长度
static inline size_t sdslen(const sds s) {
    //通过指针运算找到struct sdshdr的地址
    struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));
    return sh->len;
}
//返回sds的free属性，即buf中未使用内存大小
static inline size_t sdsavail(const sds s);
//更改sds的长度
//incr为正数时，增加len长度，减少free长度
//incr为负数时，减少free长度，增加len长度
//sdsIncrLen与sdsMakeRoomFor搭配使用可以将内核的数据直接追加到sds中，无需使用中间缓冲区
void sdsIncrLen(sds s, int incr);

//将sds的长度扩展到指定长度，不属于原来长度的字节初始化为0
//若sds原来的长度大于len的话，不进行任何操作
sds sdsgrowzero(sds s, size_t len);
//清空sds字符串的内容，但不回收内存空间
void sdsclear(sds s);    
```

