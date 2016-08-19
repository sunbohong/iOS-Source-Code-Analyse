
#深入理解Block之Block的类型

&nbsp;

&nbsp;&nbsp;&nbsp;&nbsp;当我在 2012 年刚刚开始从事 iOS 开发工作时，对 Block 的使用开始逐渐在 iOS 开发者中推广开来(Block 的第一个稳定 ABI 版本是在 Mac OS X 10.6 被引入的。)。作为 iOS 开发中非常吸引我的一个特性，对其的深入分析自然必不可少。

&nbsp;

> **重要声明**：虽然我已经仔细的检查了自己的相关代码和相关的措辞，但是请不要盲目相信本文的正确性。我已经见过非常多的经验开发者对于 Block 有错误的理解（我也不会例外）。请一定保持一颗怀疑的心。


* [类型简介](#1) 
    * [_NSConcreteGlobalBlock & _NSConcreteStackBlock](#1.1)
    * [_NSConcreteMallocBlock](#1.2) 
    * [_NSConcreteFinalizingBlock & _NSConcreteAutoBlock](#1.3)
    * [_NSConcreteWeakBlockVariable](#1.3)
* [ARC环境的特殊处理](#2)

&nbsp;

#<a name="1"></a>类型简介


对 block 稍微有所了解的人都知道，block 会在编译过程中，会被当做结构体进行处理。 其结构[Block-ABI-Apple](http://clang.llvm.org/docs/Block-ABI-Apple.html#id2)大概是这样的:

```
struct Block_literal_1 {
    void *isa; // initialized to &_NSConcreteStackBlock or &_NSConcreteGlobalBlock
    int flags;
    int reserved;
    void (*invoke)(void *, ...);
    struct Block_descriptor_1 {
    unsigned long int reserved;         // NULL
        unsigned long int size;         // sizeof(struct Block_literal_1)
        // optional helper functions
        void (*copy_helper)(void *dst, void *src);     // IFF (1<<25)
        void (*dispose_helper)(void *src);             // IFF (1<<25)
        // required ABI.2010.3.16
        const char *signature;                         // IFF (1<<30)
    } *descriptor;
    // imported variables
};
```

`isa` 指针会指向 block 所属的类型，用于帮助运行时系统进行处理。

Block 常见的类型有三种，分别是 `_NSConcreteStackBlock` `_NSConcreteMallocBlock` `_NSConcreteGlobalBlock` 。
    
另外还包括只在GC环境下使用的 ` _NSConcreteFinalizingBlock ` ` _NSConcreteAutoBlock ` ` _NSConcreteWeakBlockVariable `。

下面摘自 [libclosure-65 - Block_private.h-213](http://opensource.apple.com/source/libclosure/libclosure-65/runtime.c)


``` 
// the raw data space for runtime classes for blocks
// class+meta used for stack, malloc, and collectable based blocks
BLOCK_EXPORT void * _NSConcreteMallocBlock[32]
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_2);
BLOCK_EXPORT void * _NSConcreteAutoBlock[32]
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_2);
BLOCK_EXPORT void * _NSConcreteFinalizingBlock[32]
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_2);
BLOCK_EXPORT void * _NSConcreteWeakBlockVariable[32]
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_2);
// declared in Block.h
// BLOCK_EXPORT void * _NSConcreteGlobalBlock[32];
// BLOCK_EXPORT void * _NSConcreteStackBlock[32];
``` 


## <a name="1.1"></a>_NSConcreteGlobalBlock & _NSConcreteStackBlock

 `_NSConcreteGlobalBlock` & `_NSConcreteStackBlock` 是 block 初始化时设置的类型(上文中 [Block-ABI-Apple](http://clang.llvm.org/docs/Block-ABI-Apple.html#id2) 已经提及，并且 [CGBlocks_8cpp_source.html#l00141](http://clang.llvm.org/doxygen/CGBlocks_8cpp_source.html#l00141) 也提到过）。
 
在以下情况中，block 会初始化为 `_NSConcreteGlobalBlock` ：

* 未捕获外部变量。
    
    在 [static void computeBlockInfo(CodeGenModule &CGM, CodeGenFunction *CGF,CGBlockInfo &info)](http://clang.llvm.org/doxygen/CGBlocks_8cpp_source.html#l00326) 函数内的 [334行](http://clang.llvm.org/doxygen/CGBlocks_8cpp_source.html#l00334) 至 [339行](http://clang.llvm.org/doxygen/CGBlocks_8cpp_source.html#l00339)，通过判断 block(以及嵌套的block) 是否捕捉了本地存储(原文为：local storage)，未捕获时，block 会初始化为 `_NSConcreteGlobalBlock` 。


 ``` 
  if (!block->hasCaptures()) {
    info.StructureType =
      llvm::StructType::get(CGM.getLLVMContext(), elementTypes, true);
    info.CanBeGlobal = true;
    return;
  }
 ``` 

* 当需要布局（layout）的变量的数量为0时。

    在
[static void computeBlockInfo(CodeGenModule &CGM, CodeGenFunction *CGF,CGBlockInfo &info)](http://clang.llvm.org/doxygen/CGBlocks_8cpp_source.html#l00326)函数内，通过计算 block 的布局(layout)。当需要布局的变量为0时，block 会初始化为 `_NSConcreteGlobalBlock` 。

    
     计算布局的顺序：
    * `this` (为了访问 `c++` 的成员变量和函数，需要 `this` 指针)
    * 依次按下列规则处理捕获的变量：
        * 不需要计算布局的变量：
            * 生命周期为静态的变量（被 `const` `static` 修饰的变量，不被函数包含的静态常量，c++中生命周期为静态的变量）
            * 函数参数
        * 需要计算布局的变量：被 `__block` 修饰的变量[4]，以上未提到的类型（比如block）


    > 当布局顺序计算完毕后，会按照以下顺序进行一次稳定排序。
    
    *  __strong 修饰的变量
    *  ByRef 类型
    *  __weak 修饰的变量
    *  其它类型





## <a name="1.2"></a>_NSConcreteMallocBlock

在非垃圾收集环境下，当 `_NSConcreteStackBlock` 类型的block 被真正复制时，在 `_Block_copy_internal` 方法内部，会转换为 `_NSConcreteMallocBlock` [libclosure-65/runtime.c](http://opensource.apple.com/source/libclosure/libclosure-65/runtime.c)

```
// Its a stack block.  Make a copy.
if (!isGC) {
    struct Block_layout *result = malloc(aBlock->descriptor->size);
    if (!result) return NULL;
    memmove(result, aBlock, aBlock->descriptor->size); // bitcopy first
    // reset refcount
    result->flags &= ~(BLOCK_REFCOUNT_MASK|BLOCK_DEALLOCATING);    // XXX not needed
    result->flags |= BLOCK_NEEDS_FREE | 2;  // logical refcount 1
    result->isa = _NSConcreteMallocBlock;
    _Block_call_copy_helper(result, aBlock);
    return result;
}
```

## <a name="1.3"></a>_NSConcreteFinalizingBlock&_NSConcreteAutoBlock

在垃圾收集环境下，当 block 被复制时，如果block 有 ctors & dtors 时，则会转换为 `_NSConcreteFinalizingBlock` 类型，反之，则会转换为 `_NSConcreteAutoBlock` 类型
```
if (hasCTOR) {
    result->isa = _NSConcreteFinalizingBlock;
}
else {
    result->isa = _NSConcreteAutoBlock;
}
```

## _NSConcreteWeakBlockVariable

GC环境下，当对象被 ` __weak __block ` 修饰，且从栈复制到堆时，block 会被标记为 ` _NSConcreteWeakBlockVariable ` 类型。

```
bool isWeak = ((flags & (BLOCK_FIELD_IS_BYREF|BLOCK_FIELD_IS_WEAK)) == (BLOCK_FIELD_IS_BYREF|BLOCK_FIELD_IS_WEAK));
// if its weak ask for an object (only matters under GC)
struct Block_byref *copy = (struct Block_byref *)_Block_allocator(src->size, false, isWeak);
copy->flags = src->flags | _Byref_flag_initial_value; // non-GC one for caller, one for stack
copy->forwarding = copy; // patch heap copy to point to itself (skip write-barrier)
src->forwarding = copy;  // patch stack to point to heap copy
copy->size = src->size;
if (isWeak) {
  copy->isa = &_NSConcreteWeakBlockVariable;  // mark isa field so it gets weak scanning
}
```

# <a name="2"></a>ARC环境的特殊处理


> 下面的代码均通过添加 `objc_retainBlock` `_Block_copy` 和 `_Block_copy_internal` 符号断点进行测试


* 在 ARC 下，block 类型通过`=`进行传递时，会导致调用`objc_retainBlock`->`_Block_copy`->`_Block_copy_internal`方法链。并导致 `__NSStackBlock__` 类型的 block 转换为 `__NSMallocBlock__` 类型。


  [
objc4-680/runtime/NSObject.mm-193](http://opensource.apple.com/source/objc4/objc4-680/runtime/NSObject.mm)  提及到了这一点。

    ```
    //
    // The -fobjc-arc flag causes the compiler to issue calls to objc_{retain/release/autorelease/retain_block}
    //
    
    id objc_retainBlock(id x) {
        return (id)_Block_copy(x);
    }

    ...

    void *_Block_copy(const void *arg) {
       return _Block_copy_internal(arg, true);
    }
    ```
    
  测试代码：

  ```
  void test() {
      __block int i = 0;
      dispatch_block_t block  = ^(){NSLog(@"%@", @(i)); };
      dispatch_block_t block1 = block;
      NSLog(@"初始化为变量后再打印：%@", block1);
  
      NSLog(@"直接打印：%@", ^(){NSLog(@"%@", @(i)); });
  }
  ```

  日志：

  ```
  
  "objc_retainBlock 函数被调用"
  
  "_Block_copy 函数被调用"
  
  "_Block_copy_internal 函数被调用"
  
  "objc_retainBlock 函数被调用"
  
  "_Block_copy 函数被调用"
  
  "_Block_copy_internal 函数被调用"
  
  初始化为变量后再打印：<__NSMallocBlock__: 0x7fb05b605800>
  直接打印：<__NSStackBlock__: 0x7fff55ccc568>
 
  ```

* 在 ARC 下，不同的属性修饰符以及不同赋值、取值方式均会对方法调用产生影响。下表为测试结果。

  \            |    strong    |    retain    |    copy
------------ | ------------ | ------------ | -----------
直接赋值      | `_Block_copy`->`_Block_copy_internal`| `_Block_copy`->`_Block_copy_internal`|无
间接赋值      | `_Block_copy`->`_Block_copy_internal`| `_Block_copy`->`_Block_copy_internal`|`_Block_copy`->`_Block_copy_internal`
通过属性取值   | `_Block_copy`->`_Block_copy_internal`-> `_Block_copy`->`_Block_copy_internal` | `_Block_copy`->`_Block_copy_internal`-> `_Block_copy`->`_Block_copy_internal` | `_Block_copy`->`_Block_copy_internal`-> `_Block_copy`->`_Block_copy_internal` | `_Block_copy`->`_Block_copy_internal`-> `_Block_copy`->`_Block_copy_internal` 
通过变量取值   |无|无|无|

    
  直接赋值：
    
  ```
  NSString *str = @"sun";
  dispatch_block_t block = ^(){
      NSLog(@"%@", str);
  };
  self.block = block;
  ```
    
  间接赋值：
    
  ```
  self.block = ^(){
      NSLog(@"%@", str);
  };
  ```
  通过属性取值

  ```
  self.block
  ```
  通过变量取值

  ```
  self->_block
  ```

  测试代码:
  
  ```
    
  - (void)test {

      NSString *str = @"sun";
      {
          NSLog(@"直接赋值开始");
          {
              self.copyBlock = ^(){
                  NSLog(@"%@", str);
              };

              NSLog(@"copy 属性修饰的 block：%@", self->_copyBlock);
          }
          {
              self.strongBlock = ^(){
                  NSLog(@"%@", str);
              };

              NSLog(@"strong 属性修饰的 block：%@", self->_strongBlock);
          }
          {
              self.retainBlock = ^(){
                  NSLog(@"%@", str);
              };

              NSLog(@"retain 属性修饰的 block：%@", self->_retainBlock);
          }
          NSLog(@"直接赋值结束");
      }
      {
          dispatch_block_t copyBlock = ^(){
              NSLog(@"%@", str);
          };
          dispatch_block_t strongBlock = ^(){
              NSLog(@"%@", str);
          };
          dispatch_block_t retainBlock = ^(){
              NSLog(@"%@", str);
          };
          NSLog(@"间接赋值开始");
          {
              self.copyBlock = copyBlock;

              NSLog(@"copy 属性修饰的 block：%@", self->_copyBlock);
          }
          {
              self.strongBlock = strongBlock;

              NSLog(@"strong 属性修饰的 block：%@", self->_strongBlock);
          }
          {
              self.retainBlock = retainBlock;

              NSLog(@"retain 属性修饰的 block：%@", self->_retainBlock);
          }
          NSLog(@"间接赋值结束");
      }
      {
          NSLog(@"通过属性获取开始");
          {
              NSLog(@"copy 属性修饰的 block：%@", self.copyBlock);

              NSLog(@"strong 属性修饰的 block：%@", self.strongBlock);

              NSLog(@"retain 属性修饰的 block：%@", self.retainBlock);
          }

          NSLog(@"获取结束");
      }

      {
          NSLog(@"通过变量获取开始");
          {
              NSLog(@"copy 属性修饰的 block：%@", self->_copyBlock);

              NSLog(@"strong 属性修饰的 block：%@", self->_strongBlock);

              NSLog(@"retain 属性修饰的 block：%@", self->_retainBlock);
          }

          NSLog(@"获取结束");
      }
  }
  ```


  日志：   
  
  ```
  
  间接赋值开始
  
  
  "_Block_copy 函数被调用"
  
  "_Block_copy_internal 函数被调用"
  
  copy 属性修饰的 block：<__NSMallocBlock__: 0x7fab00fa4c30>
  
  
  "_Block_copy 函数被调用"
  
  "_Block_copy_internal 函数被调用"
  
  strong 属性修饰的 block：<__NSMallocBlock__: 0x7fab00d1a970>
  
  
  "_Block_copy 函数被调用"
  
  "_Block_copy_internal 函数被调用"
  
  retain 属性修饰的 block：<__NSMallocBlock__: 0x7fab00f08ad0>
  
  
  间接赋值结束
  
  
  通过属性获取开始
  
  
  "_Block_copy 函数被调用"
  
  "_Block_copy_internal 函数被调用"
  
  "_Block_copy 函数被调用"
  
  "_Block_copy_internal 函数被调用"
  
  copy 属性修饰的 block：<__NSMallocBlock__: 0x7fab00fa4c30>
  
  
  "_Block_copy 函数被调用"
  
  "_Block_copy_internal 函数被调用"
  
  "_Block_copy 函数被调用"
  
  "_Block_copy_internal 函数被调用"
  
  strong 属性修饰的 block：<__NSMallocBlock__: 0x7fab00d1a970>
  
  
  "_Block_copy 函数被调用"
  
  "_Block_copy_internal 函数被调用"
  
  "_Block_copy 函数被调用"
  
  "_Block_copy_internal 函数被调用"
  
  retain 属性修饰的 block：<__NSMallocBlock__: 0x7fab00f08ad0>
  
  
  获取结束
  
  通过变量获取开始
  
  copy 属性修饰的 block：<__NSMallocBlock__: 0x7fab00fa4c30>
  strong 属性修饰的 block：<__NSMallocBlock__: 0x7fab00d1a970>
  retain 属性修饰的 block：<__NSMallocBlock__: 0x7fab00f08ad0>
  
  获取结束
  (lldb) 
    
  ```        
        
