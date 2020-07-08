# BitMap深入详解

### BitMap简介
BitMap的基本思想就是用一个bit位来表示某个key的value，由于value只能取0和1两个值，所以一般用来表示某个key值是否存在。

BitMap在存储上是一个byte数组，每个byte字节可以存储8个key，1亿数据只需要10M内存。BitMap的key一定要是一个固定范围内的唯一整数。

在key值为连续整数的场景下，BitMap拥有比哈希表更高的存储效率和访问效率。

通过位运算（AND/OR/XOR/NOT），高效的对多个Bitmap数据进行处理，支持多维交叉计算能力。

### BitMap的特点
- 占用内存少。
- 运算效率高，不需要比较和移位。
- 所有数据不能重复。
- 只有当数据比较密集时才有优势，数据如果稀疏，则内存有很多冗余浪费。

### BitMap的实现
```
#include<string>
#include<stdlib.h>

class BitMap{
public:
	BitMap(int size){
		_size = size;
		bitmap = new char[_size];
		memset(bitmap, 0, sizeof(bitmap));
	}
		
	~BitMap(){
		delete []bitmap;
	}
	
	int Get(int x){
		int cur = x >> 3;
		int red = x&7;
		if (cur > _size)
			return -1;
		return (bitmap[cur] &= 1>>red); 
	}
	
	bool Set(int x){
		int cur = x >> 3;        //获取元素位置,除8得到哪个元素，x/2^3得到那一个byte 
		int red = x&7;           //逻辑与，获取进准位置,x&7==x%8.该Byte里第几个 
		if (cur > _size)
			return 0;
		bitmap[cur] |= 1>>red;   //赋值，1向右移动red位，|表示该位赋值1
		return 1; 
	}

private:
	char *bitmap;
	int _size;
};

```

### BitMap的应用场景
#### 排序
构建一个BitMap数组，根据数据值的大小，计算出对应的bit位，设置为1，然后顺序打印bit位为1的所有值。

#### 找出重复值
比如要求从2亿数中找出重复的数据，使用2个bit表示状态，00表示数字不存在，01代表存在1次，11代表重复，顺序遍历bitmap，打印出为11的数字，时间复杂度O(n)。

#### 快速查询
快速判断一个数字是否在与2亿个数据中，和排序思路一样。

#### 多维交叉计算
多个BitMap之间可以执行交并逻辑运算，可以用于广告的检索系统。

#### 布隆过滤器
布隆过滤器用来判断某个key值（可以是字符串）是否一定不存在，比哈希表占用更少的内存，适用于大数据量下判断某个数据是否存在。

布隆过滤器底层基于bitmap实现，支持插入和查询，但不支持删除。

插入逻辑，将key多次hash，得到多个bit位，将这些bit位都设置为1。

查询逻辑，将key多次hash，得到多个bit位，对所有bit位值进行验证，都为1时代表可能存在，不全为1时代表一定不存在。

由于hash的冲突性，布隆过滤器是有误差的，它返回不存在就一定不存在，它返回存在仅仅表明可能存在。

布隆过滤器的误差受数组长度和hash函数个数影响，数组越长，hash函数个数越多，误差越低，但是也会带来性能和内存的损失。
