## Upgrading PHP extensions from PHP5 to NG
---
许多经常被使用的API方法改变了，比如HashTable的API；这篇文章将尽可能多的将这些实际影响扩展和核心代码编写的地方列出来。在读这边文章之前，强烈建议现将另一篇介绍PHPNG实现的文章phpng-int先好好阅读一下。

这篇文章不可能覆盖所有的情况。在这里将介绍一些被使用的最多最频繁的例子。我希望它能帮助大多数用户级的扩展升级。However if you did not find some information here, found a solution and think it may be useful for others - feel free to add your recipe。

### General Advice
---
- 尝试在**PHPNG**中编译你的扩展。查看编译错误和警告。这些信息起码能告诉你75%需要修改的地方。
- 在调试模式下编译和测试扩展（configure PHP with –enable-debug）。运行时使用assert()方法将会捕获一些错误信息。你还可以看到有关内存泄露的信息。

### zval
---
- **PHPNG**不再使用zval的二级指针。大多数出现的zval\*\*变量和参数都将改变成zval\*。相应的，使用在这些变量上的宏**Z\_\*\_PP()**也需要变成**Z\_\*\_P()**。
- 在很多地方，**PHPNG**是直接使用zval（从而消除了分配与重新分配），在这些情况下，相应的**zval\***就需要转换成纯**zval**，相应的宏也需要从**Z\_\*P**转换成**Z\_\*()**,以及相应的创建宏从**ZVAL\_\*(var,…)**转换成**ZVAL\_\*(&var,…)**。在传递zval的地址或者使用**&**操作符的时候要始终小心。**PHPNG**几乎从来没有要求过传递**zval\***的地址。在某些地方，需要删除掉**&**操作符。
- 分配**zval**内存的宏**ALLOC_ZVAL**、**ALLOC_INIT_ZVAL**、**MAKE_STD_ZVAL**被删掉了。在大都数情况下，它们的使用表明zval*需要被转换成纯zval。宏**INIT_PZVAL**也被删除了，在大多数使用它的地方只需要删除掉就行了。

```
-  zval *zv;
-  ALLOC_INIT_ZVAL();
-  ZVAL_LONG(zv, 0);
+  zval zv;
+  ZVAL_LONG(&zv, 0);
```

- **zval**结构也完全改变了，现在它的定义形式如下：

```
struct _zval_struct {
	zend_value        value;			/* value */
	union {
		struct {
			ZEND_ENDIAN_LOHI_4(
				zend_uchar    type,			/* active type */
				zend_uchar    type_flags,
				zend_uchar    const_flags,
				zend_uchar    reserved)	    /* various IS_VAR flags */
		} v;
		zend_uint type_info;
	} u1;
	union {
		zend_uint     var_flags;
		zend_uint     next;                 /* hash collision chain */
		zend_uint     str_offset;           /* string offset */
		zend_uint     cache_slot;           /* literal cache slot */
	} u2;
};
```

zend_value的定义如下：

```
typedef union _zend_value {
	long              lval;				/* long value */
	double            dval;				/* double value */
	zend_refcounted  *counted;
	zend_string      *str;
	zend_array       *arr;
	zend_object      *obj;
	zend_resource    *res;
	zend_reference   *ref;
	zend_ast_ref     *ast;
	zval             *zv;
	void             *ptr;
	zend_class_entry *ce;
	zend_function    *func;
} zend_value;
```

- **PHPNG**和以前的**PHP5.x**主要的区别是在处理标量和复杂类型时候的不同。PHP不再在堆上为标量分配空间，而是直接在VM的栈上、**HashTable**以及对象的内部。引用计数和垃圾回收不再作用于它们。标量没有引用计数器，并且不再支持宏**Z_ADDREF\*()**、**Z_DELREF\*()**、**Z_REFCOUNT\*()**以及**Z_SET_REFCOUNT\*()**。在大多数情况下，你在使用这些宏之前，需要检查zval是否支持它们。否则你会得到一个assert()或者直接崩溃。

```
- Z_ADDREF_P(zv)
+ if (Z_REFCOUNTED_P(zv)) {Z_ADDREF_P(zv);}
# or equivalently
+ Z_TRY_ADDREF_P(zv);
```

- **zval**的值应该使用宏**ZVAL_COPY_VALUE()**来复制。
- 在必要的时候可以使用宏**ZVAL_COPY**来复制和递增引用计数器。
- 使用宏**ZVAL_DUP()**来完成zval的复制（zval_copy_ctor）。
- 如果你将一个zval*转换成zval，并且使用NULL来预定义一个没有定义的值，那么你现在可以使用**IS_UNDEF**类型来替代前面的工作。你可以使用**ZVAL_UNDEF(&zv)**来进行设置，使用**if (Z_ISUNDEF(zv))**来进行检查。
- 如果你仅仅是想获取long/double/string的值而不改变原始的zval，你可以使用**zval_get_long(zv)**、**zval_get_double(zv)**以及**zval_get_string(zv)**- 这些API来简化代码

```
- zval tmp;
- ZVAL_COPY_VALUE(&tmp, zv);
- zval_copy_ctor(&tmp);
- convert_to_string(&tmp);
- // ...
- zval_dtor(&tmp);
+ zend_string *str = zval_get_string(zv);
+ // ...
+ STR_RELEASE(str);
```

查看zend_types.h文件可以获得更多更详细的信息：[https://github.com/php/php-src/blob/phpng/Zend/zend_types.h](https://github.com/php/php-src/blob/phpng/Zend/zend_types.h)

### References
---
PHPNG中的**zval**不再需要**is_ref**标志。引用使用了一个独立复合的引用计数类型**IS_REFERENCE**来实现。你也可以一直使用宏**Z_ISREF\*()**来检查一个zval是否为引用类型。其实，它只是检查给出的zval的类型是否为**IS_REERENCE**。用在**is_ref**上的宏**Z_SET_ISREF\*()**、**Z_UNSET_ISREF\*()**、**Z_SET_ISREF_TO\*()**全部被删除了。在下面的方式中，他们的使用方法需要改变一下。

```
- Z_SET_ISREF_P(zv);
+ ZVAL_MAKE_REF(zv);

- Z_UNSET_ISREF_P(zv);
+ if (Z_ISREF_P(zv)) {ZVAL_UNREF(zv);}
```

之前的引用可以直接检查引用类型。现在我们必须通过宏**Z_REFVAL\*()**间接的检查它。

```
- if (Z_ISREF_P(zv) && Z_TYPE_P(zv) == IS_ARRAY) {
+ if (Z_ISREF_P(zv) && Z_TYPE_P(Z_REFVAL_P(zv)) == IS_ARRAY) {
```

或者使用宏**ZVAL_DEREF**进行手动解引用

```
- if (Z_ISREF_P(zv)) {...}
- if (Z_TYPE_P(zv) == IS_ARRAY) {
+ if (Z_ISREF_P(zv)) {...}
+ ZVAL_DEREF(zv);
+ if (Z_TYPE_P(zv) == IS_ARRAY) {
```

### Booleans
---
**IS_BOOL**不再存在，但是**IS_TRUE**和**IS_FALSE**分别成了两个类型：

```
- if ((Z_TYPE_PP(item) == IS_BOOL || Z_TYPE_PP(item) == IS_LONG) && Z_LVAL_PP(item)) {
+ if (Z_TYPE_P(item) == IS_TRUE || (Z_TYPE_P(item) == IS_LONG && Z_LVAL_P(item))) {
```

宏**Z_BVAL\*()**被移除了。一定要注意，**Z_LVAL\*()**的返回值是IS_FALSE还是IS_TRUE，这个是不确定的。

### Strings
---
获取字符串的值或者长度还是使用宏**Z_STRVAL\*()**和**Z_STRLEN\*()**。但是要强调的是，表示字符串的数据结构是**zend_sring**（将在下面的章节详细描述）。通过宏**Z_STR\*()**来访问zval中的**zend_string**。也可以通过宏**Z_STRHASH\*()**得到字符串的hash值。

假释代码需要检查给定的字符串是否为常量字符串，现在需要使用zend_string(而不是char*)来完成：

```
- if (IS_INTERNED(Z_STRVAL_P(zv))) {
+ if (IS_INTERNED(Z_STR_P(zv))) {
```

创建字符串zval有一点变化。此前，类似**ZVAL_STRING()**的宏有一个额外的参数告知给定的字符是否需要被复制。现在，这些宏总是会创建**zend_string**结构，因此，这个参数就变得没用了。但是，如果它的实际值是0，你必须释放掉原始的字符串以防止内存泄露。

```
- ZVAL_STRING(zv, str, 1);
+ ZVAL_STRING(zv, str);

- ZVAL_STRINGL(zv, str, len, 1);
+ ZVAL_STRINGL(zv, str, len);

- ZVAL_STRING(zv, str, 0);
+ ZVAL_STRING(zv, str);
+ efree(str);

- ZVAL_STRINGL(zv, str, len, 0);
+ ZVAL_STRINGL(zv, str, len);
+ efree(str);
```

对于类似的宏像**RETURN_STRING()**、**RETVAL_STRINGL()**等，以及一些内部API函数也是如此。

```
- add_assoc_string(zv, key, str, 1);
+ add_assoc_string(zv, key, str);

- add_assoc_string(zv, key, str, 0);
+ add_assoc_string(zv, key, str);
+ efree(str);
```

直接使用zend_string的api以及直接从zend_string创建zval可以避免双重再分配。

```
- char * str = estrdup("Hello");
- RETURN_STRING(str);
+ zend_string *str = STR_INIT("Hello", sizeof("Hello")-1, 0);
+ RETURN_STR(str);
```

**Z_STRVAL\*()**宏现在应该被用于只读对象。不应该再用它分配任何事物。更改独立的字符是可能的，但是在做之前，你需要确定这个字符串不是一个引用（它不是常量并且它的引用计数器为1），在完成了在原来的基础上修改字符串后，你需要重新计算它的hash值。

```
  SEPARATE_ZVAL(zv);
  Z_STRVAL_P(zv)[0] = Z_STRVAL_P(zv)[0] + ('A' - 'a');
+ STR_FORGET_HASH_VAL(Z_STR_P(zv))
```

### zend_string API
---
Zend有一个新的**zend_string**API，在zval中**zend_string**是字符串实现的底层结构，these structures are also used throughout much of the codebase where char* and int were used before。

**zend_string**（not IS_STRING zvals）可以使用宏**STR_INIT(char *val, int len, int persistent)**来创建。通过str>val来访问实际的字符值，通过str>len获得字符串的长度。字符串的hash值使用宏**STR_HASH_VAL()**来获取。如有必要，它会重新计算hash值。

应该使用宏**STR_RELEASE()**释放字符串，这并不需要释放内存，因为同样的字符串可能会在其他地方被引用。

如果你要在某个地方保存**zend_string**的指针，那么你需要递增它的引用计数器或者使用宏**STR_COPY()**，它可以替你做这件事。在很多代码复制字符只是想保持值（不修改）的地方，可以使用这个宏替代。

```
- ptr->str = estrndup(Z_STRVAL_P(zv), Z_STRLEN_P(zv));
+ ptr->str = STR_COPY(Z_STR_P(zv));
  ...
- efree(str);
+ STR_RELEASE(str);
```

如果被复制的字符串需要做修改，可以使用宏**STR_DUP()**替代之前的操作：

```
- zend_string *str = estrndup(Z_STRVAL_P(zv), Z_STRLEN_P(zv));
+ char *str = STR_DUP(Z_STR_P(zv));
  ...
- efree(str);
+ STR_RELEASE(str);
```

使用了旧的宏的代码也需要被支持，因此，没必要都切换到新的。

在某些情况下，在不知道实际的字符串值之前分配字符串缓冲区是非常有意义的。你可以使用宏**STR_ALLOC()**和**STR_REALLOC**来做这个工作。

```
- char *ret = emalloc(16+1);
- md5(something, ret);
- RETURN_STRINGL(ret, 16, 0);
+ zend_string *ret = STR_ALLOC(16, 0);
+ md5(something, ret->val);
+ RETRUN_STR(ret);
```

不是所有的扩展都必须用**zend_string**来替代**char\***。它取决于扩展维护者来决定在特定的情况下哪种类型更适合。


查看zend_string.h文件可以获得更多更详细的信息：[https://github.com/php/php-src/blob/phpng/Zend/zend_string.h](https://github.com/php/php-src/blob/phpng/Zend/zend_string.h)

### smart_str and smart_string
---
为了一致的命名规范，将拉偶的**smart_str**API重命名为**smart_string**。除了使用了新名字外，使用方式跟以前一样。

```
- smart_str str = {0};
- smart_str_appendl(str, " ", sizeof(" ") - 1);
- smart_str_0(str);
- RETURN_STRINGL(implstr.c, implstr.len, 0);
+ smart_string str = {0};
+ smart_string_appendl(str, " ", sizeof(" ") - 1);
+ smart_string_0(str);
+ RETVAL_STRINGL(str.c, str.len);
+ smart_str_free(&str);
```

此外，我们引入了新的**zend_str**API直接与**zend_string**一起工作

```
- smart_str str = {0};
- smart_str_appendl(str, " ", sizeof(" ") - 1);
- smart_str_0(str);
- RETURN_STRINGL(implstr.c, implstr.len, 0);
+ smart_str str = {0};
+ smart_str_appendl(str, " ", sizeof(" ") - 1);
+ smart_str_0(str);
+ if (str.s) {
+   RETURN_STR(str.s);
+ } else {
+   RETURN_EMPTY_STRING();
+ }
```

smart_str定义如下：

```
typedef struct {
    zend_string *s;
    size_t a;
} smart_str;
```

**smart_str**与**smart_string**的API非常相似，实际上他们复用了在PHP5中的API。因此采用新的api不会造成多大的问题。最大的问题是，特定情况下API的选择，但这取决于最终使用的结果。

请注意，以前检查空smart_str的方式需要变一下

```
- if (smart_str->c) {
+ if (smart_str->s) {
```

### strpprintf
---
除了**spprintf()**和**vspprintf()**函数，我们介绍一个类似的函数，跟使用**zend_string**替代**char\***一样。这得由你来决定什么时候应该更改为新的变种。

```
PHPAPI zend_string *vstrpprintf(size_t max_len, const char *format, va_list ap);
PHPAPI zend_string *strpprintf(size_t max_len, const char *format, ...);
```

### Arrays
---
数组的实现跟以前或多或少有些相同。以前的底层结构是一个指向**HashTable**的指针，而现在是一个指向**zend_array**的指针，而将**HashTable**存放于它的内部。以前使用宏**Z_ARRVAL\*()**来读取HashTable，而现在不可能将指针改为指向**HashTable**。唯一的获取和设置指向整个**zend_array**的指针的方式是通过宏**Z_ARR\*()**。

创建数组最好的方式是使用老的**array_init()**函数，但是也可以使用宏**ZVAL_NEW_ARR()**来创建一个新的未初始化的数组，或者使用宏   **ZVAL_ARR()**来初始化它的**zend_array**结构。

一些数组可能是不变的（使用宏**Z_IMMUTABLE()**来检测）。在一些代码中需要修改他们，那么他们首先必须被复制。不能使用内部位置指针来遍历不变数组。可以通过老的使用了外部位置指针的API来实现不变数组的遍历，或者使用新的**HashTable**遍历API来实现。

### HashTable API
---
**HashTable** API改变了很多，因此在移植扩展的时候可能会造成一些麻烦。
- 首先，现在所有的HashTable始终保存了zval。既是我们存储任意的指针，它都会被使用IS_PTR类型封装到zval中。无论如何，这简化了zval的工作。

```
- zend_hash_update(ht, Z_STRVAL_P(key), Z_STRLEN_P(key)+1, (void*)&zv, sizeof(zval**), NULL) == SUCCESS) {
+ if (zend_hash_update(EG(function_table), Z_STR_P(key), zv)) != NULL) {
```

- 大多数API方法直接返回请求的值（而不是使用额外的引用参数以及返回SUCCESS/FAILURE）。

```
- if (zend_hash_find(ht, Z_STRVAL_P(key), Z_STRLEN_P(key)+1, (void**)&zv_ptr) == SUCCESS) {
+ if ((zv = zend_hash_find(ht, Z_STR_P(key))) != NULL) {
```
- key用**zend_string**来表示。大多数方法有两种形式，一种是接收**zend_string**作为key，另一种就是使用char*和length这对参数。
- 重要注意事项：字符串key的长度不包含尾部的0.在一些地方，需要移除或者加上+1/-1操作：

```
- if (zend_hash_find(ht, "value", sizeof("value"), (void**)&zv_ptr) == SUCCESS) {
+ if ((zv = zend_hash_str_find(ht, "value", sizeof("value")-1)) != NULL) {
```

这也适用于其他的zend_hash外的其他哈希表相关的API。例如：

```
- add_assoc_bool_ex(&zv, "valid", sizeof("valid"), 0);
+ add_assoc_bool_ex(&zv, "valid", sizeof("valid") - 1, 0);
```

- API提供了一组独立的函数用于任意的指针。这些函数有一个以相同的后缀_ptr的名字：

```
- if (zend_hash_find(EG(class_table), Z_STRVAL_P(key), Z_STRLEN_P(key)+1, (void**)&ce_ptr) == SUCCESS) {
+ if ((func = zend_hash_find_ptr(ht, Z_STR_P(key))) != NULL) {

- zend_hash_update(EG(class_table), Z_STRVAL_P(key), Z_STRLEN_P(key)+1, (void*)&ce, sizeof(zend_class_entry*), NULL) == SUCCESS) {
+ if (zend_hash_update_ptr(EG(class_table), Z_STR_P(key), ce)) != NULL) {
```

- API提供了一组独立的函数来存储任意大小的内存块。这些函数有一个相同的后缀_mem的名字，它们是以相应的_ptr函数的内敛包装来实现的。这并不意味着有什么东西是使用_mem或者_ptr的变种来存储的。它总是可以使用**zend_hash_find_ptr()**来访问。

```
- zend_hash_update(EG(function_table), Z_STRVAL_P(key), Z_STRLEN_P(key)+1, (void*)func, sizeof(zend_function), NULL) == SUCCESS) {
+ if (zend_hash_update_mem(EG(function_table), Z_STR_P(key), func, sizeof(zend_function))) != NULL) {
```

- 一些新的优化功能作为新的元素被加入了进来。他们的目的是在只有新的元素加入的情况下被使用，不能与已存在的键重叠。比如，当你从一个HashTable中复制一些元素到一个新的HashTable中。这些方法都有_news后缀。

```
zval* zend_hash_add_new(HashTable *ht, zend_string *key, zval *zv);
zval* zend_hash_str_add_new(HashTable *ht, char *key, int len, zval *zv);
zval* zend_hash_index_add_new(HashTable *ht, pzval *zv);
zval* zend_hash_next_index_insert_new(HashTable *ht, pzval *zv);
void* zend_hash_add_new_ptr(HashTable *ht, zend_string *key, void *pData);
...
```

- HashTable的析构函数现在总是接收zval*（即使我们使用zend_hash_add_ptr或zend_hash_add_mem添加元素）。宏**Z_PTR_P()**可以用于在析构函数中获取到实际的指针值。另外，如果使用zend_hash_add_mem添加元素，那么，析构函数还要负责自身指针的释放。

```
- void my_ht_destructor(void *ptr)
+ void my_ht_destructor(zval *zv)
  {
-    my_ht_el_t *p = (my_ht_el_t*) ptr;
+    my_ht_el_t *p = (my_ht_el_t*) Z_PTR_P(zv);
     ...
+    efree(p); // this efree() is not always necessary necessary
  }
);
```

- 回调对于所有的**zend_hash_apply_\*()**函数，以及对于**zend_hash_copy()**和**zend_hash_merge()**，在同样的析构方式中，应将接收void*&&改为zval*。其中的一些函数也可以接收**zend_hash_key**结构的指针。在下面的方式中，它的定义已经被改变。对于一个字符串索引，h包含了实际字符串的hash值，而key则是实际的字符串。对于数字索引，h则是数字索引值，而key是NULL。

```
typedef struct _zend_hash_key {
	ulong        h;
	zend_string *key;
} zend_hash_key;
```

在一些情况下，将使用**zend_hash_apply\*()**替换成新的HashTable迭代APi是非常有用的。这可能将代码优化的更小和更高效。

阅读zend_hash.h是非常好的主意：[https://github.com/php/php-src/blob/phpng/Zend/zend_hash.h](https://github.com/php/php-src/blob/phpng/Zend/zend_hash.h)

### HashTable Iteration API
---
我们提供一些专门的宏用于遍历HashTable的元素或者键。这些宏的第一个参数是hashtable，而其他的参数则是在每个迭代步骤中指定的变量。

- ZEND_HASH_FOREACH_VAL(ht, val)
- ZEND_HASH_FOREACH_KEY(ht, h, key)
- ZEND_HASH_FOREACH_PTR(ht, ptr)
- ZEND_HASH_FOREACH_NUM_KEY(ht, h)
- ZEND_HASH_FOREACH_STR_KEY(ht, key)
- ZEND_HASH_FOREACH_STR_KEY_VAL(ht, key, val)
- ZEND_HASH_FOREACH_STR_KEY_VAL(ht, h, key, val)

这些宏最适合用来替代老的reset、current以及move函数。

```
- HashPosition pos;
  ulong num_key;
- char *key;
- uint key_len;
+ zend_string *key;
- zval **pzv;
+ zval *zv;
-
- zend_hash_internal_pointer_reset_ex(&ht, &pos);
- while (zend_hash_get_current_data_ex(&ht, (void**)&ppzval, &pos) == SUCCESS) {
-   if (zend_hash_get_current_key_ex(&ht, &key, &key_len, &num_key, 0, &pos) == HASH_KEY_IS_STRING){
-   }
+ ZEND_HASH_FOREACH_KEY_VAL(ht, num_key, key, val) {
+   if (key) { //HASH_KEY_IS_STRING
+   }
    ........
-   zend_hash_move_forward_ex(&ht, &pos);
- }
+ } ZEND_HASH_FOREACH_END();
```

### Objects
---
TODO:...

###Custom Objects
---
TODO:...

zend_object结构定义如下：

```
struct _zend_object {
    zend_refcounted   gc;
    zend_uint         handle; // TODO: may be removed ???
    zend_class_entry *ce;
    const zend_object_handlers *handlers;
    HashTable        *properties;
    HashTable        *guards; /* protects from __get/__set ... recursion */
    zval              properties_table[1];
};
```

为了更好的访问性能，我们内联了properties_table,但是也带来了一些问题，我们曾经定义了一个类似下面这样的自定义对象：

```
struct custom_object {
   zend_object std;
   void  *custom_data;
}
 
 
zend_object_value custom_object_new(zend_class_entry *ce TSRMLS_DC) {
 
   zend_object_value retval;
   struct custom_object *intern;
 
   intern = emalloc(sizeof(struct custom_object));
   zend_object_std_init(&intern->std, ce TSRMLS_CC);
   object_properties_init(&intern->std, ce);
   retval.handle = zend_objects_store_put(intern,
        (zend_objects_store_dtor_t)zend_objects_destroy_object,
        (zend_objects_free_object_storage_t) custom_free_storage,
        NULL TSRMLC_CC);
   intern->handle = retval.handle;
   retval.handlers = &custom_object_handlers;
   return retval;
}
 
struct custom_object* obj = (struct custom_object *)zend_objects_get_address(getThis());
```

但是现在，zend_object是可变长度的（内联properties_table）。因此，上面的代码需要变成下面这样。

```
struct custom_object {
   void  *custom_data;
   zend_object std;
}
 
zend_object * custom_object_new(zend_class_entry *ce TSRMLS_DC) {
     # Allocate sizeof(custom) + sizeof(properties table requirements)
     struct custom_object *intern = ecalloc(1,
         sizeof(struct custom_object) +
         sizeof(zval) * (ce->default_properties_count - 1));
     # Allocating:
     # struct custom_object {
     #    void *custom_data;
     #    zend_object std;
     # }
     # zval[ce->default_properties_count-1]
     zend_object_std_init(&intern->std, ce TSRMLS_CC);
     ...
     custom_object_handlers.offset = XtOffsetof(struct custom_obj, std);
     custom_object_handlers.free_obj = custom_free_storage;
 
     return &intern->std;
}
 
# Fetching the custom object:
 
#define struct custom_object * php_custom_object_fetch_object(zend_object *obj) {
      return (struct custom_object *)((char *)obj - XtOffsetOf(struct custom_object, std));
}
 
#define Z_CUSTOM_OBJ_P(zv) php_custom_object_fetch_object(Z_OBJ_P(zv));
 
struct custom_object* obj = Z_CUSTOM_OBJ_P(getThis());
```

### zend_object_handlers
---
一个新的项偏移被加入了zend_object_handlers，在你自己的对象结构中需要定义它作为zend_object的偏移。

通过zend_objects_store_*来定位分配的内存的准确的开始地址。

```
// An example in spl_array
memcpy(&spl_handler_ArrayObject, zend_get_std_object_handlers(), sizeof(zend_object_handlers));
spl_handler_ArrayObject.offset = XtOffsetOf(spl_array_object, std);
```

现在对象的内存会通过zend_objects_store_*释放掉，因此你不需要在你自定义的对象的free_obj句柄上释放内存。

### Resources
---
**IS_RESOURCE**不再是资源类型的处理的类型。资源的处理不能再通过**Z_LVAL\*()**来获取了。而是通过宏**Z_RES\*()**来直接获取资源的记录。资源记录通过**zend_resource**结构来表示。它包含*type*-资源类型，*ptr*-指向实际值的指针，*handle*-数字资源索引（兼容性）以及提供引用计数器的字段。实际上，这个**zend_resurce**结构是间接引用**zend_rsrc_list_entry**的替代品。所有使用了**zend_rsrc_list_entry**都应该被替换为**zend_resource**。

* <del>**zend_list_find()**</del>方法已经被删除了，因为资源可以直接的存取了。

```
- long handle = Z_LVAL_P(zv);
- int  type;
- void *ptr = zend_list_find(handle, &type);
+ long handle = Z_RES_P(zv)->handle;
+ int  type = Z_RES_P(zv)->type;
+ void *ptr = = Z_RES_P(zv)->ptr;
```

* <del>**Z_RESVAL_\*()**</del>宏被删除，使用**Z_RES\*()**来替换它

```
- long handle = Z_RESVAL_P(zv);
+ long handle = Z_RES_P(zv)->handle;
```

* **ZEND_FETCH_RESOURCE()**宏的第三个参数为**zval\***

```
- ZEND_FETCH_RESOURCE2(ib_link, ibase_db_link *, &link_arg, link_id, LE_LINK, le_link, le_plink);
+ ZEND_FETCH_RESOURCE2(ib_link, ibase_db_link *, link_arg, link_id, LE_LINK, le_link, le_plink);
```

* <del>**zend_list_addref()**</del>和<del>**zend_list_delref()**</del>都被删掉。资源使用同**zval**相同的引用计数原理。

```
- zend_list_addref(Z_LVAL_P(zv));
+ Z_ADDREF_P(zv);
```

和下面的操作是一样的

```
- zend_list_addref(Z_LVAL_P(zv));
+ Z_RES_P(zv)->gc.refcount++;
```

* **zend_list_delete()**由资源处理句柄换成了指向**zend_resource**结构的指针。

```
- zend_list_delete(Z_LVAL_P(zv));
+ zend_list_delete(Z_RES_P(zv));
```

* 在大多数用户扩展函数中，像mysql_close，你需要使用**zend_list_close**来替换**zend_list_delete()**。这样就关闭了实际的连接以及释放了扩展特定的数据结构，但是它不释放**zend_reference**结构。这样也就可能一直都是zval(s)的引用。这样也不会递减资源引用计数器。

```
- zend_list_delete(Z_LVAL_P(zv));
+ zend_list_close(Z_RES_P(zv));
```

### Parameters Parsing API changes
---
* **PHPNG**不再会出现使用**zval\*\***的情况，因此就不再需要**'Z'**说明符了。必须使用**'z'**来代替它。

```
- zval **pzv;
- if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "Z", &pzv) == FAILURE) {
+ zval *zv;
+ if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "z", &zv) == FAILURE) {
```

* 除了**'s'**指代字符串外，**PHPNG**引入**'S'**也用来指代字符串，但是用来放置参数到**zend_string**变量。在某些情况下，优先直接使用**zend_string**。（例如接收一个字符串作为HashTable的键的API）

```
- char *str;
- int len;
- if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "s", &str, &len) == FAILURE) {
+ zend_string *str;
+ if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "S", &str) == FAILURE) {
```

* **'+'**和**'\*'**现在仅仅返回数组的zvals（替代以前的数组的**zval\*\***）。

```
- zval ***argv = NULL;
+ zval *argv = NULL;
  int argn;
  if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "+", &argv, &argn) == FAILURE) {
```

* 通过引用传递参数应该被分配到引用的值里面。分离这些参数，然后在第一个地方获取到引用的值。

```
- zval **ret;
- if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "Z", &ret) == FAILURE) {
+ zval *ret;
+ if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "z/", &ret) == FAILURE) {
    return;
  }
- ZVAL_LONG(*ret, 0);
+ ZVAL_LONG(ret, 0);
```

### Call Frame Changes(zend_execute_data)
---
有关每个函数调用的信息会被记录在一个由zend_execute_data结构组成的链表中。EG(current_execute_data)指向当前正在执行的函数的调用帧（以前只为用户级的PHP函数创建zend_execute_data结构）。我会按照结构成员字段尽可能详细的解释旧的和新的调用帧之前的区别。

* **zend_execute_data.opline** - 当前正在执行的用户函数的指令指针。对于内部函数它的值是未定义的（在旧的结构中它的值是NULL）。
* **<del>zend_execute_data.function_state</del>** - 这个字段被删除。使用**zend_execute_data.call**来代替它。
* **zend_execute_data.call** - 在以前它是一个纸箱当前调用槽的指针。现在它是一个指向当前正在调用的函数的**zend_execute_data**结构的指针。这个字段最初是NULL，然后被ZEND_INIT_FCALL（或类似的）字节码改变，最终又被ZEND_FO_FCALL还原回来。嵌套函数调用语法，类似foo($a, bar($c))，通过**zend_execute_data.prev_nested_call**构造一个这种结构的调用链表。
* **<del>zend_execute_data.op_array</del>** - 这个字段被**zend_execute_data.func**代替，因为现在它不仅仅表示用户函数，也表示内部函数。
* **zend_execute_data.func** - 当前被执行的函数。
* **zend_execute_data.object** - 当前正在被执行的method所属类（$this）（以前是是zval\*，现在是zend_object\*）。
* **zend_execute_data.symbol_table** - 当前的符号表或NULL
* **zend_execute_data.prev_execute_data** - 回溯调用连接
* **<del>original_return_value</del>，<del>current_scope</del>，<del>current_called_scope</del>，<del>current_this</del>** - 这些字段用于保存旧的值，在函数调用完成后恢复他们的值。现在他们被移除了。
* **zend_execute_data.scope** - 当前被执行的函数的范围（新字段）。
* **zend_execute_data.called_scope** - 当前被执行的函数的调用范围（新字段）
* **zend_execute_data.run_time_cache** - 当前正在被执行的函数的运行时缓存。这是一个新字段，实际上它是op_array.run_time_cache的一个副本。
* **zend_execute_data.num_args** - 传递给函数的参数的个数（新字段）
* **zend_execute_data.return_value** - 当前正在被执行的op_array需要存储结果的zval*结构的指针。如果调用不关心最终的返回值的话，他的值为NULL（新字段）。

函数的参数直接被存储在zval槽中，并在**zend_execute_data**结构之后。使用宏**ZEND_CALL_ARG(execute_data, arg_num)**访问。For user PHP functions first argument overlaps with first compiled cariable - CV0, etc. In case caller passes more arguments that callee receives, all extra arguments are copied to be after all used by calee CVs and TMP variables。

### Executor Globals - EG() Changes
---
* **EG(symbol_table)** - 变成了**zend_array**（以前是**HashTable**）。到达HashTable的底层结构不是什么大问题。

```
- symbols = zend_hash_num_elements(&EG(symbol_table));
+ symbols = zend_hash_num_elements(&EG(symbol_table).ht);
```

* **<del>EG(uninitialized_zval_ptr)</del>**和**<del>EG(error_zval_ptr)</del>**都被删除了。使用**&EG(uninitialized_zval)**和**&EG(error_zval)**来替代。
* **EG(current_execute_data)** - 这个字段的意思改变了一点。以前它是一个指向上一个被执行函数的调用栈的指针，而现在是指向上一个调用栈的指针（不用关心它是用户还是内部函数）。可以通过遍历调用链表得到上一个op_array的**zend_execute_data**结构。

```
  zend_execute_data *ex = EG(current_execute_data);
+ while (ex && (!ex->func || !ZEND_USER_CODE(ex->func->type))) {
+    ex = ex->prev_execute_data;
+ }
  if (ex) {
```

* **<del>EG(opline_ptr)</del>** - 被删除。使用**execute_data->opline**来代替。
* **<del>EG(return_value_ptr_ptr)</del>** - 被删除。使用**execute_data→return_value**来代替。
* **<del>EG(active_symbol_table)</del>** - 被删除。使用**execute_data→symbol_table**来代替。
* **<del>EG(active_op_array)</del>** - 被删除。使用**execute_data→func**来代替。
* **<del>EG(called_scope)</del>** - 被删除。使用**execute_data→called_scope**来代替。
* **EG(This)** - 变成了**zval**，以前它是**zval**指针，用户代码不应该修改它。
* **EG(in_execution)** - 被删除。如果EG(current_excute_data)不是NULL，那么我们正在执行一些东西。
* **EG(exception)**和**EG(prev_exception)** - 变成了指向**zend_object**的指针，以前他们是**zval**的指针。

### Opcodes changes
---
* **<del>ZEND_DO_FCALL_BY_NAME</del>** - 删除，增加了**ZEND_INIT_FCALL_BY_NAME**。
* **ZEND_BIND_GLOBAL** - 添加来处理“global $var”。
* **ZEND_STRLEN** - 添加进来替换strlen函数。
* **ZEND_TYPE_CHECK** - 如果可能的话，添加进来替换is_array/is_int/is_*
* **ZEND_DEFINED** - 若果可能的话，添加来替换zif_defined（如果仅仅只有一个参数，并且是常数字符串，并且不是namespace风格）
* **ZEND_SEND_VAR_EX** - 如果条件不能稳定在编译时刻的话，它用于比ZEND_SEND_VAR做更多的检查。
* **ZEND_SEND_VAL_EX** - 如果条件不能稳定在编译时刻的话，它用于比ZEND_SEND_VAL做更多的检查。
* **ZEND_INIT_USER_CALL** - 如果可以的话，在编译时刻不能找到函数的情况下替换call_user_func(_array)，否则它会被转换成ZEND_INIT_FCALL。
* **ZEND_SEND_ARRAY** - 被添加用于在被转换成opcode之后传递call_user_func_array的第二个数组参数
* **ZEND_SEND_USER** - 被添加用于在被转换成opcode之后传递call_user_fun的参数

### temp_variable
---

### PCRE
---
一些pcre API现在也在使用或者返回zend_string。例如，php_pcre_replace返回一个zend_string并且第一个参数为zend_string。仔细检查他们的声明以及编译器警告,这很有可能是错误的参数类型。



phpng-upgrading.txt · Last modified: 2014/07/26 15:49 by [laruence](https://people.php.net/laruence)