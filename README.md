# Runtime

## 一. isa 指针详解

学习 runtime 之前,要了解下它底层的一些常用数据结构.如 isa 指针.

在 `arm64 架构之前`, isa 就是一个普通的指针,存储着 `Class 、 Meta-Class 对象的内存地址`.

在 `arm64 架构之后`, 对 isa 进行了优化,变成了一个`共用体(union)结构`,还使用`位域`来存储更多的信息.

通过看`objc`源码.可以看到`isa` 的结构:

```c
struct objc_object {
private:
    isa_t isa;

public:

...

```

下面主要分析 `arm64` 架构下的

```c
union isa_t 
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;

#if SUPPORT_PACKED_ISA

    // extra_rc must be the MSB-most field (so it matches carry/overflow flags)
    // nonpointer must be the LSB (fixme or get rid of it)
    // shiftcls must occupy the same bits that a real class pointer would
    // bits + RC_ONE is equivalent to extra_rc + 1
    // RC_HALF is the high bit of extra_rc (i.e. half of its range)

    // future expansion:
    // uintptr_t fast_rr : 1;     // no r/r overrides
    // uintptr_t lock : 2;        // lock for atomic property, @synch
    // uintptr_t extraBytes : 1;  // allocated with extra bytes

# if __arm64__
#   define ISA_MASK        0x0000000ffffffff8ULL
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
    struct {
    
        /** 1. nonpointer
         * 为0时 : 表示普通的指针,存储着 Class、Meta-Class 对象的内存地址
         * 为1时 : 表示 isa 优化过,使用位域存储着更多的信息.
         */
        uintptr_t nonpointer        : 1;
        
        /** 2. has_assoc
         * 是否有设置过关联对象
         * 如果没有,释放时会更快
         */
        uintptr_t has_assoc         : 1;
        
        /** 3. has_cxx_dtor
         *  是否有 C++ 的析构函数(.cxx_destruct)
         *  如果没有,释放时会更快
         */
        uintptr_t has_cxx_dtor      : 1;
        
        /** 4. shiftcls
         * 存储着  Class、Meta-Class 对象的内存地址信息
         */
        uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000
        
        /** 5. magic
         * 在调试时用来分辨对象是否未完成初始化
         */
        uintptr_t magic             : 6;
        
        /** 6. weakly_referenced
         * 是否有被弱引用指向过,如果没有,释放时会更快
         */
        uintptr_t weakly_referenced : 1;
        
        /** 7. deallocating
         *  对象是否正在释放
         */
        uintptr_t deallocating      : 1;
        
        /** 8. has_sidetable_rc
         * 引用计数器是否过大无法存储在 isa 中
         * 如果为1, 那么引用计数会存储在一个叫 `SideTable` 的类的属性中.
         */
        uintptr_t has_sidetable_rc  : 1;
        
        /** 9. extra_rc
         * 里面存储的值是引用计数器减1
         */
        uintptr_t extra_rc          : 19;
#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };

# elif __x86_64__
#   define ISA_MASK        0x00007ffffffffff8ULL
#   define ISA_MAGIC_MASK  0x001f800000000001ULL
#   define ISA_MAGIC_VALUE 0x001d800000000001ULL
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 44; // MACH_VM_MAX_ADDRESS 0x7fffffe00000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 8;
#       define RC_ONE   (1ULL<<56)
#       define RC_HALF  (1ULL<<7)
    };

# else
#   error unknown architecture for packed isa
# endif

// SUPPORT_PACKED_ISA
#endif


#if SUPPORT_INDEXED_ISA

# if  __ARM_ARCH_7K__ >= 2

#   define ISA_INDEX_IS_NPI      1
#   define ISA_INDEX_MASK        0x0001FFFC
#   define ISA_INDEX_SHIFT       2
#   define ISA_INDEX_BITS        15
#   define ISA_INDEX_COUNT       (1 << ISA_INDEX_BITS)
#   define ISA_INDEX_MAGIC_MASK  0x001E0001
#   define ISA_INDEX_MAGIC_VALUE 0x001C0001
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t indexcls          : 15;
        uintptr_t magic             : 4;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 7;
#       define RC_ONE   (1ULL<<25)
#       define RC_HALF  (1ULL<<6)
    };

# else
#   error unknown architecture for indexed isa
# endif

// SUPPORT_INDEXED_ISA
#endif

};

```

## 二. Class 的结构

元类对象其实是一种特殊的类对象.

meta-Class对象 和 Class对象的内部结构基本相同,只是各自存储的内容有所差别.

### 1.Class 对象的底层结构.

#### objc_class

```c++
struct objc_class {
    Class isa;
    Class superclass;
    // 方法缓存
    cache_t cache;
    // 用来获取具体的类信息
    class_data_bits_t bits;
}
```

#### class_rw_t
其中 `class_data_bits_t` 它`&`上掩码`FAST_DATA_MASK` 得到 `class_rw_t`,它是可读可写的,在运行时可以改其内容.

```c++
 class_rw_t * data() {
        return (class_rw_t *)(bits & FAST_DATA_MASK);
 }
```

`class_rw_t` 的结构:

```c++
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    // 只读(readOnly),不可更改,里面有成员变量的结构体.
    const class_ro_t *ro;

    // 方法列表 (包含类及分类的)
    method_array_t methods;
    // 属性列表 (包含类及分类的)
    property_array_t properties;
    // 协议列表 (包含类及分类的)
    protocol_array_t protocols;

    Class firstSubclass;
    Class nextSiblingClass;

    char *demangledName;

#if SUPPORT_INDEXED_ISA
    uint32_t index;
#endif

    void setFlags(uint32_t set) 
    {
        OSAtomicOr32Barrier(set, &flags);
    }

    void clearFlags(uint32_t clear) 
    {
        OSAtomicXor32Barrier(clear, &flags);
    }

    // set and clear must not overlap
    void changeFlags(uint32_t set, uint32_t clear) 
    {
        assert((set & clear) == 0);

        uint32_t oldf, newf;
        do {
            oldf = flags;
            newf = (oldf | set) & ~clear;
        } while (!OSAtomicCompareAndSwap32Barrier(oldf, newf, (volatile int32_t *)&flags));
    }
};
```

##### method_array_t

- `method_array_t` 是个二维数组,它里面装的是 `method_list_t`.

- 而 `method_list_t` 里面装的是 `method_t`.
- `property_array_t` 和`protocol_array_t`同理.


```c++
class method_array_t : 
    public list_array_tt<method_t, method_list_t> 
{
    typedef list_array_tt<method_t, method_list_t> Super;

 public:
    method_list_t **beginCategoryMethodLists() {
        return beginLists();
    }
    
    method_list_t **endCategoryMethodLists(Class cls);

    method_array_t duplicate() {
        return Super::duplicate<method_array_t>();
    }
};
```

#### class_ro_t

其中的 `baseMethodList` 、 `baseProtocols` 、 `ivars` 、 `baseProperties` 都是一维数组.并且是只读的.只包含当前类的初始内容,不包含分类的内容.

```c++
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    
    // 实例对象占用的内存空间
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    // 类名
    const char * name;
    // 方法列表(一维数组)
    method_list_t * baseMethodList;
    // 协议列表(一维数组)
    protocol_list_t * baseProtocols;
    
    // 成员变量列表
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    // 属性列表(一维数组)
    property_list_t *baseProperties;

    method_list_t *baseMethods() const {
        return baseMethodList;
    }
};
```

#### method_t

`method_t` 是对方法/函数的封装.

```c++
struct method_t {

    // 函数名
    SEL name;
    
    // 编码(返回值类型、参数类型)
    const char *types;
    
    // 指向函数的指针(函数地址)
    IMP imp;

    struct SortBySELAddress :
        public std::binary_function<const method_t&,
                                    const method_t&, bool>
    {
        bool operator() (const method_t& lhs,
                         const method_t& rhs)
        { return lhs.name < rhs.name; }
    };
};
```

#### cache_t

使用`散列表`这种数据结构来缓存曾经调用过的方法,可以提高方法的查找速度.

比如一个对象要调用其对象方法,其原理就是通过这个对象的`isa`指针,找到这个对象的方法列表`method_list_t`(二维数组,里面存的是`method_t`).然后遍历这个方法列表,找到对应的方法实现. 如果当前对象没有找到的话,那么会通过 `superclass` 指针,找到其父类,然后再遍历,如果还找不到,再通过`superclass`指针找其父类,直到找到基类为止.

`如果没有方法缓存的话,每次都重复上面的流程,效率是非常低的.`

所以当我们第一次调用方法时,如果方法存在,那么就会被缓存到当前这个类的方法缓存中.(如果一个类调用一个方法,这个类和其父类都找到,而是在基类中找到的,那么就会将这个方法缓存到`这个类`的方法缓存中),下次再调用这个方法的话,会通过`isa`找到这个类对象,然后直接从其方法缓存中去拿.

- `cache_t`的底层结构如下

```c++
struct cache_t {

    // 数组,里面装的散列表
    struct bucket_t *_buckets;
    
    // 散列表的长度 - 1. (比如散列表长度为10, 那么其值就是 10-1 = 9)
    mask_t _mask;
    
    // 已经缓存的方法数量
    mask_t _occupied;

public:
    struct bucket_t *buckets();
    mask_t mask();
    mask_t occupied();
    void incrementOccupied();
    void setBucketsAndMask(struct bucket_t *newBuckets, mask_t newMask);
    void initializeToEmpty();

    mask_t capacity();
    bool isConstantEmptyCache();
    bool canBeFreed();

    static size_t bytesForCapacity(uint32_t cap);
    static struct bucket_t * endMarker(struct bucket_t *b, uint32_t cap);

    void expand();
    void reallocate(mask_t oldCapacity, mask_t newCapacity);
    struct bucket_t * find(cache_key_t key, id receiver);

    static void bad_cache(id receiver, SEL sel, Class isa) __attribute__((noreturn));
};
```

- `struct bucket_t`

找到 key, 如果和 SEL 相同,那么直接拿到函数地址,进行调用方法.

```c++
struct bucket_t {
private:

    // SEL 作为 key
    cache_key_t _key;
    
    // 函数的内存地址
    IMP _imp;

public:
    inline cache_key_t key() const { return _key; }
    inline IMP imp() const { return (IMP)_imp; }
    inline void setKey(cache_key_t newKey) { _key = newKey; }
    inline void setImp(IMP newImp) { _imp = newImp; }

    void set(cache_key_t newKey, IMP newImp);
};
```


