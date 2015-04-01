#OS lab2 report

>参考并理解了lab2_result的代码


##[练习0]填写已有实验
>运用meld对lab1的代码进行了移植。

##[练习1]实现first-fit连续物理内存分配算法

first-fit 算法实现主要在`kern/mm/default_pmm.c` 中。需要修改其中的三个函数：初始化`default_init_memmap`, 分配` default_alloc_pages`, 释放 `default_free_pages`.

初始化函数 `default_init_memmap` 需要将以`base`为基址的`n`个页面依次加入链表，设置好`flags`，不是链表头的页`property` 属性置`0`， 链表头`property` 属性置1。

分配函数`default_alloc_pages`找到第一块`size` 大于指定大小`n` 的块并分配。首先检查`n` 是否比剩余的空间还大， 如果是则肯定找不到合适的块，返回空。由链表头遍历，若找不到页数大于`n` 的块，则失败，返回空。如果找到一个块满足条件， 则将其前`n` 页分配，并对各页设好`flags`。若还有剩余页，将剩余页的`list entry` 的`property`属性置为原节点的`property`-`n`。

释放函数`default_free_pages` 将要释放的页面添加进`free_list`, 并合并相邻的块。根据释放块的基址， 在`free_list` 找到合适的位置， 将页面插入其中， 并修改相应标记。如果相邻块可以合并，则合并，并修改`property`属性。

算法改进：尚未找到合适的算法。

##[练习2]实现寻找虚拟地址对应的页表项
- 获取PDE
- 若PDE不是当前`entry`, 则为页表分配新的页， 设置引用并用memset清零，设置权限。
- 返回PTE， 即`return &((pte_t*)KADDR(PDE_ADDR(*pdep)))[PTX(la)]`。

1.描述页目录项（Page Director Entry）和页表（Page Table Entry）中个部分含义及其对ucore的潜在用处。
>

2.如果ucore执行过程中访问内存，出现页访问异常，请问硬件要做哪些事？
>替换掉一个页面，将要访问的页面载入主存。


##[练习3]释放某虚地址所在页并取消对应二级页表项的映射
根据提示， 先检查是否存在，找到相应页，减少引用，若引用次数为0，则释放页面。取消二级页表项的映射，然后flush TLB。

1.数据结构Page的全局变量的每一项与页表中的页目录项和页表项有无对应关系？若有，是啥？
>有！

2.如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？
>尚不知道。

