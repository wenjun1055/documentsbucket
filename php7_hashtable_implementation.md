# PHP7的HashTable实现
---
PHP开发的主干已经切换到大版本7了，这个版本Zend引擎改变很大，可以说是一个全新的版本，因此新的Zend引擎被命名为NG。在这个大版本中变量以及hash表变动也很大。所以在这里把自己的学习过程和成果记录下来，毕竟好记性不如烂笔头啊。

## 数据结构
---
HashTable的结构定义在*Zend/zend_types.h，*之前的php版本中被定义在*Zend/Zend_hash.h*文件中。
HashTable中的每一个元素都被保存在一个Bucket结构中，新的结构定义如下：

```
typedef struct _Bucket {
	zval              val;
	zend_ulong        h;                /* hash value (or numeric index)   */
	zend_string      *key;              /* string key or NULL for numerics */
} Bucket;
```

新的Bucket的结构已经相当简洁了，不像原来的里面还有链表指针什么的。HashTable的结构定义如下：

```
typedef struct _HashTable {	
	union {
		struct {
			ZEND_ENDIAN_LOHI_3(
				zend_uchar    flags,
				zend_uchar    nApplyCount,	/* 循环遍历保护 */
				uint16_t      reserve)
		} v;
		uint32_t flags;
	} u;
	uint32_t          nTableSize;			/* hash表的大小 */
	uint32_t          nTableMask;			/* 掩码,用于根据hash值计算存储位置,永远等于nTableSize-1 */
	uint32_t          nNumUsed;				/* arData数组已经使用的数量 */
	uint32_t          nNumOfElements;		/* hash表中元素个数 */
	uint32_t          nInternalPointer; 	/* 用于HashTable遍历 */
	zend_long         nNextFreeElement;	/* 下一个空闲可用位置的数字索引 */
	Bucket           *arData;				/* 存放实际数据 */
	uint32_t         *arHash;				/* Hash表 */
	dtor_func_t       pDestructor;			/* 析构函数 */
} HashTable;
```

## 创建HashTable
---

创建一个HashTable有很多途径，但是最终调用的都是**_zend_hash_init**函数，下面是这个函数的实现,我们省掉windows的实现部分

```
ZEND_API void _zend_hash_init(HashTable *ht, uint32_t nSize, dtor_func_t pDestructor, zend_bool persistent ZEND_FILE_LINE_DC)
{
	...
	
	/* size should be between 8 and 0x80000000 */
	nSize = (nSize <= 8 ? 8 : (nSize >= 0x80000000 ? 0x80000000 : nSize));
# if defined(__GNUC__)
	ht->nTableSize =  0x2 << (__builtin_clz(nSize - 1) ^ 0x1f);
# else
	nSize -= 1;
	nSize |= (nSize >> 1);
	nSize |= (nSize >> 2);
	nSize |= (nSize >> 4);
	nSize |= (nSize >> 8);
	nSize |= (nSize >> 16);
	ht->nTableSize = nSize + 1;
# endif

	ht->nTableMask = 0;
	ht->nNumUsed = 0;
	ht->nNumOfElements = 0;
	ht->nNextFreeElement = 0;
	ht->arData = NULL;
	ht->arHash = (uint32_t*)&uninitialized_bucket;
	ht->pDestructor = pDestructor;
	ht->nInternalPointer = INVALID_IDX;
	ht->u.flags = (persistent ? HASH_FLAG_PERSISTENT : 0) | HASH_FLAG_APPLY_PROTECTION;
}
```

pDestructor是HashTable元素的析构指针，在更新/销毁HashTable中的元素时候会使用。上面的一系列得nSize的计算只是为了得到一个大于等于原始nSize的2的整数次幂的最小值,在这里并没有初始化arData，它会在使用HashTable的时候来创建。

## 操作HashTable

### 添加更新元素
---
在新的实现中，将以前的最底层添加和更新元素的api**_zend_hash_add_or_update**变成了现在的**_zend_hash_add_or_update_i**，其他的对HashTable的新增和更新的操作全部是基于**_zend_hash_add_or_update_i**实现的宏。下面是**_zend_hash_add_or_update_i**的实现：

```
static zend_always_inline zval *_zend_hash_add_or_update_i(HashTable *ht, zend_string *key, zval *pData, uint32_t flag ZEND_FILE_LINE_DC)
{
	zend_ulong h;
	uint32_t nIndex;
	uint32_t idx;
	Bucket *p;

	IS_CONSISTENT(ht);

	if (UNEXPECTED(!(ht->u.flags & HASH_FLAG_INITIALIZED))) {	/* 检查hashtable是否初始化 */
		CHECK_INIT(ht, 0);
		goto add_to_hash; 
	} else if (ht->u.flags & HASH_FLAG_PACKED) {	/* ? */
		zend_hash_packed_to_hash(ht);
	} else if ((flag & HASH_ADD_NEW) == 0) {	/* 新增 */
		p = zend_hash_find_bucket(ht, key);	/* 根据key查是否已经存在 */

		if (p) {	/* 当前的key已经存在 */
			zval *data;

			if (flag & HASH_ADD) {	/* key已经存在产生添加冲突，退出 */
				return NULL;
			}
			ZEND_ASSERT(&p->val != pData);	/* key存在的情况下，值不一样做更新操作 */
			data = &p->val;
			if ((flag & HASH_UPDATE_INDIRECT) && Z_TYPE_P(data) == IS_INDIRECT) {
				data = Z_INDIRECT_P(data);
			}
			HANDLE_BLOCK_INTERRUPTIONS();
			if (ht->pDestructor) {
				ht->pDestructor(data);	/* 释放掉原来的data */
			}
			ZVAL_COPY_VALUE(data, pData);	/* 将新的pData值复制给原来的data */
			HANDLE_UNBLOCK_INTERRUPTIONS();
			return data;
		}
	}
	
	ZEND_HASH_IF_FULL_DO_RESIZE(ht);		/* If the Hash table is full, resize it */

add_to_hash:
	HANDLE_BLOCK_INTERRUPTIONS();
	idx = ht->nNumUsed++;	/* 已使用计数+1，并且用老的位置来做为索引 */
	ht->nNumOfElements++;	/* 元素个数加1 */
	if (ht->nInternalPointer == INVALID_IDX) {
		ht->nInternalPointer = idx;
	}
	p = ht->arData + idx; 	/* 指针加法移位 */
	p->h = h = zend_string_hash_val(key);	/* 计算key的hash值 */
	p->key = key;
	zend_string_addref(key);
	ZVAL_COPY_VALUE(&p->val, pData);
	nIndex = h & ht->nTableMask;	/* 与tablemask进行计算得出hash索引 */
	Z_NEXT(p->val) = ht->arHash[nIndex];	/* 新的元素的hash冲突链表的next指向当前冲突链表的首部元素 */
	ht->arHash[nIndex] = idx;		/* 新的元素放到当前hash冲突链表的头部 */
	HANDLE_UNBLOCK_INTERRUPTIONS();

	return &p->val;
}

#define Z_NEXT(zval)		(zval).u2.next
```

从上面的api **_zend_hash_add_or_update_i**可以看出，其实更新操作很简单的，验证key是否存在，key存在的情况下如果值相等的话不做任何的操作，值不相同做更新操作。

这里比较重要的是hash表的新增，这里会涉及hash索引以及hash冲突链表。上面代码中的**add_to_hash**部分的代码主要就是在做这部分的做操。从上面的代码中可以看出，**ht->arData** 其实就是一个不定长的Bucket数组，它利用 **ht->nNumUsed** 来计算这个数组下一个空闲位置,然后将新的Bucket放到这个位置上。计算得到传入的key的hash值，然后与 **ht->nTableMask** 进行按位与得出新的bucket对应的 **ht->arHash上的位置** ,这里就完成了hash表的构造。

但是还有个非常重要的地方，就是hash冲突的解决。hash冲突一直都是以一个链表来解决的，当出现hash冲突时候会将相同hash值的bucket连接成一个链表，然后进行查找时候是一个个的遍历，一个个的对比key值是否相同。和NG之前不同的是，现在的hash冲突链表被放到了 **__zval_struct** 结构中，而不是以前的Bucket里面。下面先是新老结构的对比

```
Old Bucket
typedef struct bucket {
    ulong h;                        /* Used for numeric indexing */
    uint nKeyLength;
    void *pData;
    void *pDataPtr;
    struct bucket *pListNext;		/* 指向哈希链表中下一个元素 */
    struct bucket *pListLast;		/* 指向哈希链表中的前一个元素 */
    struct bucket *pNext;			/* 相同哈希值的下一个元素（哈希冲突用） */
    struct bucket *pLast;			/* 相同哈希值的上一个元素（哈希冲突用） */
    const char *arKey;
} Bucket;

New Bucket
typedef struct _Bucket {
	zval              val;
	zend_ulong        h;                /* hash value (or numeric index)   */
	zend_string      *key;              /* string key or NULL for numerics */
} Bucket;
```

从上面结构的对比可以看出，新的 **_Bucket** 只是一个单纯存储key以及value的结构了，而不承担以前的hash链表以及hash冲突链表的构建工作了。新的**HashTable**中，hash链表的构建工作由 **HashTable->arHash** 来承担，而解决hash冲突的链表则被放到了**_zval_struct**了。在新的实现当中，**HashTable->arHash[]** 会指向冲突链表的头，然后利用 **_zval_struct.u2.next** 来指向冲突链表中的下一个Bucket对应的 **arHash** 的位置。这样，现在的冲突链表俨然由原来的双向链表变成了现在的单向链表了。当新的冲突产生的时候，会将新的 **Bucket** 放入这个冲突链表的头部，然后让这个链表所对应的 **arHash** 的位置重新指向它，然后它的 **_zval_struct.u2.next**  指向原来的冲突链表头元素。就这样就构建好了一个冲突链表。

![新的HashTable结构图](http://wenjunblog-images.stor.sinaapp.com/original/172597d72309ff4d05a3be933e82bc30.jpg)

### 删除元素
---
Zend提供了一系列删除HashTable中元素的API，，定义如下：

```
ZEND_API int zend_hash_del(HashTable *ht, zend_string *key);
ZEND_API int zend_hash_del_ind(HashTable *ht, zend_string *key);
ZEND_API int zend_hash_str_del(HashTable *ht, const char *key, size_t len);
ZEND_API int zend_hash_str_del_ind(HashTable *ht, const char *key, size_t len);
ZEND_API int zend_hash_index_del(HashTable *ht, zend_ulong h);
```

接下来看看 **zend_hash_del** 的代码：

```
ZEND_API int zend_hash_del(HashTable *ht, zend_string *key)
{
	zend_ulong h;
	uint32_t nIndex;
	uint32_t idx;
	Bucket *p;
	Bucket *prev = NULL;

	IS_CONSISTENT(ht);

	h = zend_string_hash_val(key);		/* 根据key计算出hash值 */
	nIndex = h & ht->nTableMask;		/* 与nTableMask按位与计算出在hashtable中的位置 */

	idx = ht->arHash[nIndex];
	while (idx != INVALID_IDX) {
		p = ht->arData + idx;		/* 指针移位 */
		if ((p->key == key) ||		/* 比较key值 */
			(p->h == h &&
		     p->key &&
		     p->key->len == key->len &&
		     memcmp(p->key->val, key->val, key->len) == 0)) {
			_zend_hash_del_el_ex(ht, idx, p, prev);		/* 做实际删除动作的函数 */
			return SUCCESS;
		}
		prev = p;
		idx = Z_NEXT(p->val);	/* 遍历hash冲突链表 */
	}
	return FAILURE;
}
```

上面的函数函数非常简单，就是进行hash查找，然后比较key值，最后调用函数 **_zend_hash_del_el_ex** 进行实际的删除动作，我们接下来看看 **_zend_hash_del_el_ex** d的代码实现：

```
static zend_always_inline void _zend_hash_del_el_ex(HashTable *ht, uint32_t idx, Bucket *p, Bucket *prev)
{
	HANDLE_BLOCK_INTERRUPTIONS();
	if (!(ht->u.flags & HASH_FLAG_PACKED)) {	/* 非自然key序数组 */
		if (prev) {	/* 找到的元素不在hash冲突链表的首位，将它的前一个的next指针指向它的下一个元素 */
			Z_NEXT(prev->val) = Z_NEXT(p->val);
		} else {	/* 找到的元素位于hash冲突链表的首位，将arHash对应的位置指向它的下一个元素 */
			ht->arHash[p->h & ht->nTableMask] = Z_NEXT(p->val);
		}
	}
	if (ht->nNumUsed - 1 == idx) {	/* 元素位于arData数组的尾部 */
		do {
			ht->nNumUsed--;
		} while (ht->nNumUsed > 0 && (Z_TYPE(ht->arData[ht->nNumUsed-1].val) == IS_UNDEF));		/* 循环自减，去掉删除当前元素后遗留的尾部的UNDEF元素 */
	}
	ht->nNumOfElements--;	/* hash表中元素个数减1 */
	if (ht->nInternalPointer == idx) {
		while (1) {
			idx++;
			if (idx >= ht->nNumUsed) {	/* 当idx指向的元素位于hash表最后一个元素时候，重置hash遍历指针为无效指针 */
				ht->nInternalPointer = INVALID_IDX;
				break;
			} else if (Z_TYPE(ht->arData[idx].val) != IS_UNDEF) {	/* 当前元素不在hash表最后，并且指向的元素是非UNDEF的，将hash遍历指针指向它 */
				ht->nInternalPointer = idx;
				break;
			}
		}
	}
	if (p->key) {	/* 释放key */
		zend_string_release(p->key);
	}
	if (ht->pDestructor) {	/* 调用析构函数释放value的值 */
		zval tmp;
		ZVAL_COPY_VALUE(&tmp, &p->val);
		ZVAL_UNDEF(&p->val);
		ht->pDestructor(&tmp);
	} else {
		ZVAL_UNDEF(&p->val);
	}
	HANDLE_UNBLOCK_INTERRUPTIONS();
}
```

有一个注意的就是上面出现了一个宏 **HASH_FLAG_PACKED** 。根据鸟哥原话的解释：
> 表示一个自然key序的数组，就是可以随机访问的数组，就好比是C语言的数
> 
> array(1,2,3,4)，那么$arr[2]就是ht->arData[2]
> 
> array(1,2,3,4) packed / array(1 => 2, 3=> 2, 2=> 1) 非packed

个人理解就是自然排序(packed)就是Zend引擎自己生成的一个key，而不是程序员指定的。

### 清空HashTable
---

Zend提供了一个api来清空整个HashTable **zend_hash_clean**，下面是它的实现代码：

```
ZEND_API void zend_hash_clean(HashTable *ht)
{
	Bucket *p, *end;

	IS_CONSISTENT(ht);

	if (ht->nNumUsed) {		/* 非空HashTable */
		p = ht->arData;		/* 数组头 */
		end = p + ht->nNumUsed;		/* 数组尾 */
		if (ht->pDestructor) {
			if (ht->u.flags & HASH_FLAG_PACKED) {		/* 自然key序数组 */
				do {
					if (EXPECTED(Z_TYPE(p->val) != IS_UNDEF)) {
						ht->pDestructor(&p->val);
					}
				} while (++p != end);	/* 循环遍历，挨个调用析构函数进行销毁 */
			} else {
				do {
					if (EXPECTED(Z_TYPE(p->val) != IS_UNDEF)) {
						ht->pDestructor(&p->val);		/* 调用析构函数进行数据销毁 */
						if (EXPECTED(p->key)) {		/* 存在key，销毁key并释放内存 */
							zend_string_release(p->key);
						}
					}
				} while (++p != end);	/* 循环遍历进行销毁 */
			}
		} else {
			if (!(ht->u.flags & HASH_FLAG_PACKED)) {		/* 非自然key序数组 */
				do {
					if (EXPECTED(Z_TYPE(p->val) != IS_UNDEF)) {
						if (EXPECTED(p->key)) {
							zend_string_release(p->key);
						}
					}
				} while (++p != end);	/* 循环遍历，挨个销毁key */
			}
		}
		if (!(ht->u.flags & HASH_FLAG_PACKED)) {		/* 非自然key序数组，重置arHash */
			memset(ht->arHash, INVALID_IDX, ht->nTableSize * sizeof(uint32_t));	
		}
	}
	ht->nNumUsed = 0;	/* arData计数置为0 */
	ht->nNumOfElements = 0;		/* hash表中元素个数置为0 */
	ht->nNextFreeElement = 0;	/* 下一个空闲可用位置的数字索引置为0 */
	ht->nInternalPointer = INVALID_IDX;	/* Hash遍历指针置为不可用 */
}
```

### 销毁HashTable
---

**HashTable** 的销毁是由 **zend_hash_destroy** 来负责的，实现如下：

```
ZEND_API void zend_hash_destroy(HashTable *ht)
{
	Bucket *p, *end;

	IS_CONSISTENT(ht);

	if (ht->nNumUsed) {
		p = ht->arData;
		end = p + ht->nNumUsed;
		if (ht->pDestructor) {
			SET_INCONSISTENT(HT_IS_DESTROYING);

			if (ht->u.flags & HASH_FLAG_PACKED) {
				do {
					if (EXPECTED(Z_TYPE(p->val) != IS_UNDEF)) {
						ht->pDestructor(&p->val);
					}
				} while (++p != end);
			} else {
				do {
					if (EXPECTED(Z_TYPE(p->val) != IS_UNDEF)) {
						ht->pDestructor(&p->val);
						if (EXPECTED(p->key)) {
							zend_string_release(p->key);
						}
					}
				} while (++p != end);
			}
		
			SET_INCONSISTENT(HT_DESTROYED);
		} else {
			if (!(ht->u.flags & HASH_FLAG_PACKED)) {
				do {
					if (EXPECTED(Z_TYPE(p->val) != IS_UNDEF)) {
						if (EXPECTED(p->key)) {
							zend_string_release(p->key);
						}
					}
				} while (++p != end);
			}
		}
	} else if (EXPECTED(!(ht->u.flags & HASH_FLAG_INITIALIZED))) {
		return;
	}
	pefree(ht->arData, ht->u.flags & HASH_FLAG_PERSISTENT);		/* 释放arData的内存 */
}
```

其实 **zend_hash_destroy** 和 **zend_hash_clean** 的实现很相近，释放元素的过程几乎一样，唯一的区别就是最后的工作，**zend_hash_destroy** 最后是将 **arData** 彻底释放了。

## 结尾

**HashTable**在PHP的内核中的地位非常重要，数组、作用域等都是靠**HashTable**来实现的，PHP数组设计的操作肯定不止这点，更多的操作还包括排序、rehash等。不过这些都是最普遍的增删改查。更多的实现请自行阅读PHP源码 [*zend_hash.c*](https://github.com/php/php-src/blob/master/Zend/zend_hash.c)         [*zend_hash.h*](https://github.com/php/php-src/blob/master/Zend/zend_hash.h) 