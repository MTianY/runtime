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


