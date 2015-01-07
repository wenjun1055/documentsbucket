# PHPNG与PHP5.4类型对比
---
在之前的PHP(5.6之前的版本)中，我们一般认为变量类型有8种

* null
* long
* double
* boolean
* array
* object
* string
* resource

在phpng中进行了一些扩展和分裂，变成了1种(

* undef
* null
* false
* true
* long
* double
* string
* array
* object
* resource
* reference


Note:本文从zval结构开始对比，然后对比HashTable、Object等结构，最后再来说PHP5.4特有的结构和PHPNG特有的结构


结构中可能出现的宏

```
PHPNG：
	#ifdef WORDS_BIGENDIAN
		# define ZEND_ENDIAN_LOHI(lo, hi)          hi; lo;
		# define ZEND_ENDIAN_LOHI_3(lo, mi, hi)    hi; mi; lo;
		# define ZEND_ENDIAN_LOHI_4(a, b, c, d)    d; c; b; a;
		# define ZEND_ENDIAN_LOHI_C(lo, hi)        hi, lo
		# define ZEND_ENDIAN_LOHI_C_3(lo, mi, hi)  hi, mi, lo,
		# define ZEND_ENDIAN_LOHI_C_4(a, b, c, d)  d, c, b, a
	#else
		# define ZEND_ENDIAN_LOHI(lo, hi)          lo; hi;
		# define ZEND_ENDIAN_LOHI_3(lo, mi, hi)    lo; mi; hi;
		# define ZEND_ENDIAN_LOHI_4(a, b, c, d)    a; b; c; d;
		# define ZEND_ENDIAN_LOHI_C(lo, hi)        lo, hi
		# define ZEND_ENDIAN_LOHI_C_3(lo, mi, hi)  lo, mi, hi,
		# define ZEND_ENDIAN_LOHI_C_4(a, b, c, d)  a, b, c, d
	#endif
```

关于上面的**WORDS_BIGENDIAN**是关于网络字节序的，详情请看  
[Big and Little Endian](http://baike.baidu.com/view/2368412.htm)  
[字节序](http://baike.baidu.com/view/2194385.htm)



### zval
---
PHP5.4  

zval:

```
struct _zval_struct {
    /* Variable information */
    zvalue_value value;     /* value */
    zend_uint refcount__gc;
    zend_uchar type;    /* active type */
    zend_uchar is_ref__gc;
};

typedef union _zvalue_value {
    long lval;                  /* long value */
    double dval;                /* double value */
    struct {
        char *val;
        int len;
    } str;
    HashTable *ht;              /* hash table value */
    zend_object_value obj;
} zvalue_value;
```

PHPNG:

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

### HashTable
---
PHP5.4:

```
typedef struct _hashtable {
    uint nTableSize;
    uint nTableMask;
    uint nNumOfElements;
    ulong nNextFreeElement;
    Bucket *pInternalPointer;   /* Used for element traversal */
    Bucket *pListHead;
    Bucket *pListTail;
    Bucket **arBuckets;
    dtor_func_t pDestructor;
    zend_bool persistent;
    unsigned char nApplyCount;
    zend_bool bApplyProtection;
#if ZEND_DEBUG
    int inconsistent;
#endif
} HashTable;

typedef struct bucket {
    ulong h;                        /* Used for numeric indexing */
    uint nKeyLength;
    void *pData;
    void *pDataPtr;
    struct bucket *pListNext;
    struct bucket *pListLast;
    struct bucket *pNext;
    struct bucket *pLast;
    const char *arKey;
} Bucket;

typedef struct _zend_hash_key {
    const char *arKey;
    uint nKeyLength;
    ulong h;
} zend_hash_key;
```

PHPNG:

```
typedef struct _HashTable {	
	zend_uint         nTableSize;
	zend_uint         nTableMask;
	zend_uint         nNumUsed;
	zend_uint         nNumOfElements;
	long              nNextFreeElement;
	Bucket           *arData;
	zend_uint        *arHash;
	dtor_func_t       pDestructor;
	zend_uint         nInternalPointer; 
	union {
		struct {
			ZEND_ENDIAN_LOHI_3(
				zend_uchar    flags,
				zend_uchar    nApplyCount,
				zend_ushort   reserve)
		} v;
		zend_uint flags;
	} u;
} HashTable;

typedef struct _Bucket {
	zend_ulong        h;                /* hash value (or numeric index)   */
	zend_string      *key;              /* string key or NULL for numerics */
	zval              val;
} Bucket;
```

### Object
---
PHP5.4:

```
typedef struct _zend_object {
    zend_class_entry *ce;
    HashTable *properties;
    zval **properties_table;
    HashTable *guards; /* protects from __get/__set ... recursion */
} zend_object;

struct _zend_class_entry {
    char type;
    const char *name;
    zend_uint name_length;
    struct _zend_class_entry *parent;
    int refcount;
    zend_uint ce_flags;

    HashTable function_table;
    HashTable properties_info;
    zval **default_properties_table;
    zval **default_static_members_table;
    zval **static_members_table;
    HashTable constants_table;
    int default_properties_count;
    int default_static_members_count;

    union _zend_function *constructor;
    union _zend_function *destructor;
    union _zend_function *clone;
    union _zend_function *__get;
    union _zend_function *__set;
    union _zend_function *__unset;
    union _zend_function *__isset;
    union _zend_function *__call;
    union _zend_function *__callstatic;
    union _zend_function *__tostring;
    union _zend_function *serialize_func;
    union _zend_function *unserialize_func;

    zend_class_iterator_funcs iterator_funcs;

    /* handlers */
    zend_object_value (*create_object)(zend_class_entry *class_type TSRMLS_DC);
    zend_object_iterator *(*get_iterator)(zend_class_entry *ce, zval *object, int by_ref TSRMLS_DC);
    int (*interface_gets_implemented)(zend_class_entry *iface, zend_class_entry *class_type TSRMLS_DC); /* a class implements s interface */
    union _zend_function *(*get_static_method)(zend_class_entry *ce, char* method, int method_len TSRMLS_DC);

    /* serializer callbacks */
    int (*serialize)(zval *object, unsigned char **buffer, zend_uint *buf_len, zend_serialize_data *data TSRMLS_DC);
    int (*unserialize)(zval **object, zend_class_entry *ce, const unsigned char *buf, zend_uint buf_len, zend_unserialize_data *data TSRMLS_DC);

    zend_class_entry **interfaces;
    zend_uint num_interfaces;

    zend_class_entry **traits;
    zend_uint num_traits;
    zend_trait_alias **trait_aliases;
    zend_trait_precedence **trait_precedences;

    union {
        struct {
            const char *filename;
            zend_uint line_start;
            zend_uint line_end;
            const char *doc_comment;
            zend_uint doc_comment_len;
        } user;
        struct {
           const struct _zend_function_entry *builtin_functions;
            struct _zend_module_entry *module;
        } internal;
    } info;
};
```

PHPNG:

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

struct _zend_class_entry {
    char type;
    zend_string *name;
    struct _zend_class_entry *parent;
    int refcount;
    zend_uint ce_flags;

    HashTable function_table;
    HashTable properties_info;
    zval *default_properties_table;
    zval *default_static_members_table;
    zval *static_members_table;
    HashTable constants_table;
    int default_properties_count;
    int default_static_members_count;

    union _zend_function *constructor;
    union _zend_function *destructor;
    union _zend_function *clone;
    union _zend_function *__get;
    union _zend_function *__set;
    union _zend_function *__unset;
    union _zend_function *__isset;
    union _zend_function *__call;
    union _zend_function *__callstatic;
    union _zend_function *__tostring;
    union _zend_function *__debugInfo;
    union _zend_function *serialize_func;
    union _zend_function *unserialize_func;

    zend_class_iterator_funcs iterator_funcs;

    /* handlers */
    zend_object* (*create_object)(zend_class_entry *class_type TSRMLS_DC);
    zend_object_iterator *(*get_iterator)(zend_class_entry *ce, zval *object, int by_ref TSRMLS_DC);
    int (*interface_gets_implemented)(zend_class_entry *iface, zend_class_entry *class_type TSRMLS_DC); /* a class implements this interface */
    union _zend_function *(*get_static_method)(zend_class_entry *ce, zend_string* method TSRMLS_DC);

    /* serializer callbacks */
    int (*serialize)(zval *object, unsigned char **buffer, zend_uint *buf_len, zend_serialize_data *data TSRMLS_DC);
    int (*unserialize)(zval *object, zend_class_entry *ce, const unsigned char *buf, zend_uint buf_len, zend_unserialize_data *data TSRMLS_DC);

    zend_class_entry **interfaces;
    zend_uint num_interfaces;

    zend_class_entry **traits;
    zend_uint num_traits;
    zend_trait_alias **trait_aliases;
    zend_trait_precedence **trait_precedences;

    union {
        struct {
            zend_string *filename;
            zend_uint line_start;
            zend_uint line_end;
            zend_string *doc_comment;
        } user;
        struct {
            const struct _zend_function_entry *builtin_functions;
            struct _zend_module_entry *module;
        } internal;
    } info;
};

struct _zend_object_handlers {
    /* offset of real object header (usually zero) */
    int                                     offset;
    /* general object functions */
    zend_object_free_obj_t                  free_obj;
    zend_object_dtor_obj_t                  dtor_obj;
    zend_object_clone_obj_t                 clone_obj;
    /* individual object functions */
    zend_object_read_property_t             read_property;
    zend_object_write_property_t            write_property;
    zend_object_read_dimension_t            read_dimension;
    zend_object_write_dimension_t           write_dimension;
    zend_object_get_property_ptr_ptr_t      get_property_ptr_ptr;
    zend_object_get_t                       get;
    zend_object_set_t                       set;
    zend_object_has_property_t              has_property;
    zend_object_unset_property_t            unset_property;
    zend_object_has_dimension_t             has_dimension;
    zend_object_unset_dimension_t           unset_dimension;
    zend_object_get_properties_t            get_properties;
    zend_object_get_method_t                get_method;
    zend_object_call_method_t               call_method;
    zend_object_get_constructor_t           get_constructor;
    zend_object_get_class_entry_t           get_class_entry;
    zend_object_get_class_name_t            get_class_name;
    zend_object_compare_t                   compare_objects;
    zend_object_cast_t                      cast_object;
    zend_object_count_elements_t            count_elements;
    zend_object_get_debug_info_t            get_debug_info;
    zend_object_get_closure_t               get_closure;
    zend_object_get_gc_t                    get_gc;
    zend_object_do_operation_t              do_operation;
    zend_object_compare_zvals_t             compare;
};
```

### zend_refcounted (only phpng)
---

```
struct _zend_refcounted {
	zend_uint         refcount;			/* reference counter 32-bit */
	union {
		struct {
			ZEND_ENDIAN_LOHI_3(
				zend_uchar    type,
				zend_uchar    flags,    /* used for strings & objects */
				zend_ushort   gc_info)  /* keeps GC root number (or 0) and color */
		} v;
		zend_uint type_info;
	} u;
};
```

### zend_string (only phpng)
---

```
struct _zend_string {
	zend_refcounted   gc;
	zend_ulong        h;                /* hash value */
	int               len;
	char              val[1];
};
```

### zend_array (only phpng)
---

```
struct _zend_array {
	zend_refcounted   gc;
	HashTable         ht;
};
```

### zend_resource (only phpng)
---

```
struct _zend_resource {
	zend_refcounted   gc;
	long              handle; // TODO: may be removed ???
	int               type;
	void             *ptr;
};
```

### zend_reference (only phpng)
---

```
struct _zend_reference {
	zend_refcounted   gc;
	zval              val;
};
```

### zend_ast_ref (only phpng)
---

```
struct _zend_ast_ref {
	zend_refcounted   gc;
	zend_ast         *ast;
};

struct _zend_ast {
    unsigned short kind;
    unsigned short children;
    union {
        zval      val;
        zend_ast *child;
    } u;
};
```

