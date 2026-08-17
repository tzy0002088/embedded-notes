---
title: "copyin 逐页拷贝：为什么每次只算到页尾的字节数"
date: 2026-08-18
tags: [xv6, vm, copyin, page-table, mmu]
status: notes
---

## 背景

阅读 `kernel/vm.c` 的 `copyin()` 时，看到下面两行：

```c
n = PGSIZE - (srcva - va0);
if (n > len)
  n = len;
```

不理解为什么拷贝前要先算“当前页还剩多少字节”，所以要求举例说明。

## 提问

这部分是在干啥，请举例说明下。

## 理解

这两行在 `copyin()` 中，用来计算本轮最多可以拷贝多少字节：

1. 不能超过当前用户物理页的末尾。
2. 不能超过这次还剩下的总长度 `len`。

完整代码：

```c
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  uint64 n, va0, pa0;

  while (len > 0) {
    va0 = PGROUNDDOWN(srcva);
    pa0 = walkaddr(pagetable, va0);

    if (pa0 == 0) {
      if ((pa0 = vmfault(pagetable, va0, 1)) == 0) {
        return -1;
      }
    }

    n = PGSIZE - (srcva - va0);
    if (n > len)
      n = len;

    memmove(dst, (void *)(pa0 + (srcva - va0)), n);

    len -= n;
    dst += n;
    srcva = va0 + PGSIZE;
  }

  return 0;
}
```

`va0 = PGROUNDDOWN(srcva)` 得到 `srcva` 所在页的起始地址。

`srcva - va0` 是页内偏移。

`PGSIZE - (srcva - va0)` 就是从当前地址到这一页末尾还剩多少字节。

`if (n > len) n = len;` 则是把本轮拷贝量限制在剩余需求以内。

所以本质是：

```text
n = min(到当前页末尾的字节数, 剩余需要拷贝的字节数)
```

### 例子 1：len 很短，不需要跨页

```text
PGSIZE = 4096
srcva  = 0x1205
len    = 100
```

```text
va0 = 0x1000
offset = srcva - va0 = 0x205 = 517
到页尾的字节数 = 4096 - 517 = 3579
```

因为：

```text
100 < 3579
```

所以：

```text
n = 100
```

本轮只拷贝 100 字节，从物理地址：

```text
pa0 + 0x205
```

拷到内核缓冲区 `dst`。

### 例子 2：len 很长，需要跨页

```text
PGSIZE = 4096
srcva  = 0x1205
len    = 10000
```

第一轮：

```text
va0 = 0x1000
offset = 0x205
n = 4096 - 517 = 3579
```

因为：

```text
3579 < 10000
```

所以本轮只拷贝到当前页末尾，共 3579 字节。

之后：

```text
len = 10000 - 3579 = 6421
srcva = 0x1000 + 0x1000 = 0x2000
```

下一轮从下一页继续处理。

## 要点

- 用户虚拟地址连续，不代表底层物理地址连续。
- `walkaddr()` 一次只翻译一个虚拟页，所以 `copyin()` 必须逐页处理。
- `n = PGSIZE - (srcva - va0)` 计算当前页内还能拷贝的字节数。
- `if (n > len) n = len;` 保证不会超过剩余总长度。
- 循环更新 `srcva = va0 + PGSIZE`，进入下一页。

## 参考

- `/home/tzy/inspiration/my_xv6/kernel/vm.c`
