# alloc_pages要和__free_pages搭配 而不是free_pages
## 产生原因：
- 使用alloc_pages返回的page结构体直接作为free_pages的参数进行页面释放；显示*Unable to handle kernel paging request at virtual address*
## 参数类型兼容性
__free_pages直接操作page结构体和order，属于底层接口

free_pages接受虚拟地址，内部转换后调用__free_pages
alloc_pages返回的是page结构体，自然要和同样操作page结构体的__free_pages配对
- alloc_pages 返回值
alloc_pages 返回分配内存的 struct page*（物理页描述符），例如：

```
struct page *page = alloc_pages(GFP_KERNEL, order); // 返回page指针
unsigned long addr = __get_free_pages(GFP_KERNEL, order);
```
- 释放函数的参数要求
__free_pages 的第一个参数是 struct page*，与 alloc_pages 返回类型一致：
free_pages 的第一个参数是虚拟地址 (unsigned long addr)：
```
void __free_pages(struct page *page, unsigned int order);
void free_pages(unsigned long addr, unsigned int order);
```
若错误使用 free_pages 释放 alloc_pages 分配的页，需额外转换：
```
free_pages((unsigned long)page_address(page), order); // 需先通过page_address()转为虚拟地址
```
## 接口层级设计差异
- alloc_pages/__free_pages：
属于伙伴系统底层接口，直接操作物理页描述符（struct page）。适用于需要直接管理物理页的场景（如页表操作、DMA映射）。
- __get_free_pages/free_pages：
属于上层封装接口，返回虚拟地址。适用于需要直接使用虚拟地址的普通内存申请（如 kmalloc 内部调用 __get_free_pages）。
## 内存释放的内部逻辑
- __free_pages 直接接收 struct page*，可高效定位物理页块并归还给伙伴系统，过程中可能触发页块合并（通过 expand 函数合并空闲块）
- free_pages 需先通过虚拟地址反向查找对应的 struct page*（例如通过 virt_to_page），增加额外开销，且易因地址转换错误导致崩溃。
## 引用计数与复合页处理
alloc_pages 分配的页可能属于复合页（Compound Page），其引用计数由 struct page 管理。__free_pages 能正确处理复合页的释放逻辑，而 free_pages 设计上未考虑此场景，强制混用可能导致内存泄漏或内核崩溃。
|分配函数   |释放函数	   |适用场景   |
| ------------ | ------------ | ------------ |
|alloc_pages	   |__free_pages   |直接操作物理页（如驱动 DMA）   |
|__get_free_pages   |free_pages   |需要虚拟地址的普通内存申请   |
## 总结
|分配函数	   |释放函数	   |参数类型	   |适用场景   |
| ------------ | ------------ | ------------ | ------------ |
|alloc_page	   |free_page/__free_page   |struct page*	   |单页物理内存分配   |
|alloc_pages	   |__free_pages   |struct page*	   |多页物理内存分配   |
|__get_free_page	   |free_pages	   |虚拟地址(unsigned long)	   |需要虚拟地址的单页分配   |
|kmalloc	   |kfree   |虚拟地址(void*)	   |小对象分配（数组之类内存）   |
