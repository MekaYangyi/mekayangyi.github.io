# redis源码阅读-整数集合

创建一个只包含整数值元素的集合，同时元素数量不多时，redis会使用整数集合作为键的底层实现。

## 整数集合的实现

```c
typedef struct intset {
    uint32_t encoding; // 编码方式
    uint32_t length; // 元素数量
    int8_t contents[]; // 保存元素的数组
} intset;
```

encoding表示整数集合的编码模式，目前提供三种模式

```c
#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))
```
<!--more-->

## 升级

整数集合有三种编码模式，为了能够节省空间，一般采用能够符合所有元素要求的编码。

当新添加元素比整数集合中现有元素类型都长，那么就需要进行升级，将编码位数提升，负荷新添加元素类型长度。

步骤：

1. 根据新元素的类型，扩展整数集合底层数组空间大小，并为新元素分配空间。

2. 将底层数组现有的所有元素都转换为新元素相同的类型，并将类型转换后的元素放置到正确的位置上。

3. 将新元素添加到数组中。

```c
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    uint8_t curenc = intrev32ifbe(is->encoding); // 当前编码
    uint8_t newenc = _intsetValueEncoding(value); // 获取编码
    int length = intrev32ifbe(is->length); // 元素数量
    int prepend = value < 0 ? 1 : 0; // 根据情况判断添加到数组的最前还是最后（要升级只有这种可能）

    is->encoding = intrev32ifbe(newenc); // 更新编码方式
    // 重新分配空间
    is = intsetResize(is,intrev32ifbe(is->length)+1);

    // 从后往前重新编码
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    // 根据情况添加到尾部后者头部
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```
## 整数集合API
### 创建

```c
intset *intsetNew(void) {
    // 分配空间
    intset *is = zmalloc(sizeof(intset));
    // 设置初始编码
    is->encoding = intrev32ifbe(INTSET_ENC_INT16);
    // 初始化元素数量
    is->length = 0;
    return is;
}
```

### 添加元素

判断数据大小，如果超出现有编码的范围，升级。

如果没有，则插入到指定位置。

```c
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    uint8_t valenc = _intsetValueEncoding(value); // 计算新插入值编码
    uint32_t pos;
    if (success) *success = 1;

    // 如果需要则升级
    if (valenc > intrev32ifbe(is->encoding)) {
        return intsetUpgradeAndAdd(is,value);
    } else {
        
        // 查找，如果存在返回失败信息
        if (intsetSearch(is,value,&pos)) {
            if (success) *success = 0;
            return is;
        }

        // 重新分配空间
        is = intsetResize(is,intrev32ifbe(is->length)+1);
        // 如果插入中间位置，则将该位置之后的值移动到尾部
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }

    // 设置值
    _intsetSet(is,pos,value);

    // 计数器增加
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

### 移除数据

```c
intset *intsetRemove(intset *is, int64_t value, int *success) {
    uint8_t valenc = _intsetValueEncoding(value); // 计算编码方式
    uint32_t pos;
    if (success) *success = 0;
    // 查找值，并删除
    if (valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,&pos)) {
        uint32_t len = intrev32ifbe(is->length);
        // 删除成功标志
        if (success) *success = 1;
        // 删除数据
        if (pos < (len-1)) intsetMoveTail(is,pos+1,pos);
        // 调整内存大小
        is = intsetResize(is,len-1);
        // 更新length值
        is->length = intrev32ifbe(len-1);
    }
    return is;
}
```

### 其他API

1. intsetFind 判断值是否在集合中
2. intsetRandom 随机返回整数集合中的一个数
3. intsetGet 取出底层数组在给定索引上的元素
4. intsetLen 返回整数集合中的元素个数
5. intsetloblen 返回整数集合占用的内存字节数
