# Project1—BufferPoolManager LRU替换策略

## 任务描述
>This component is responsible for tracking page usage in the buffer pool. You will implement a new sub-class called `LRUReplacer` in **src/include/buffer/lru_replacer.h** and its corresponding implementation file in **src/buffer/lru_replacer.cpp**. `LRUReplacer` extends the abstract `Replacer` class (**src/include/buffer/replacer.h**), which contains the function specifications.\
\
>The size of the `LRUReplacer` is the same as buffer pool since it contains placeholders for all of the frames in the `BufferPoolManager`. However, not all the frames are considered as in the `LRUReplacer`. The `LRUReplacer` is initialized to have no frame in it. Then, only the newly unpinned ones will be considered in the `LRUReplacer`.\
\
>You will need to implement the LRU policy discussed in the class. You will need to implement the following methods:
1. `Victim(T*)` : Remove the object that was accessed the least recently compared to all the elements being tracked by the `Replacer`, store its contents in the output parameter and return `True`. If the `Replacer` is empty return `False`.
2. `Pin(T) `: This method should be called after a page is pinned to a frame in the `BufferPoolManager`. It should remove the frame containing the pinned page from the `LRUReplacer`.
3. `Unpin(T)` : This method should be called when the `pin_count` of a page becomes 0. This method should add the frame containing the unpinned page to the `LRUReplacer`.
4. `Size()` : This method returns the number of frames that are currently in the `LRUReplacer`.
>The implementation details are up to you. You are allowed to use built-in STL containers. You can assume that you will not run out of memory, but you must make sure that the operations are thread-safe.

这是**project 1**的第一个部分，根据上面的描述，我们的任务是实现LRU替换策略。

课程为我们提供的代码已经包括了一些必要的声明，我们需要在理解$LRU$的工作原理的基础上实现上述的四个函数。

## LRU原理及实现

### LRU
**LRU**: `Least recently used`。LRU是一种缓存算法，通常也叫做缓存替换策略，用来管理内存中的缓存。根据**wikipedia**的描述[<sup>1</sup>](#refer-anchor-1)，存在将近二十种缓存替换策略。 LRU作为其中一种替换策略，中文表述是**最近最久未使用算法**。

![LRU.png](https://i.loli.net/2021/06/17/SGXYRKCU7bVy9ow.png)
<center id = "pic1" style="font-size:16px;text-decoration:underline">图1.LRU示意图</center> 

**基本思想**：因为内存是有限的，缓存区也是有限的，因此当缓存中 `page` 满了且我们需要载入一个新的 `page` 的时候，必须将缓存中的一个 `page` 替换出去，将需要的 `page` 再替换进来，才能满足我们的使用需求。为了解决上述的问题， `LRU` 的基本思想是，将缓存中最久未使用过的 `page` 替换出去。

**实现LRU**：根据上面的描述，我们需要维护每个 `page` 最近被使用的时间戳，每当我们需要从缓存中挑选一个 `page` 进行置换的时候，找到时间戳最小的 `page` 即可。

- 算法尝试：一个**看上去**比较正确的想法是，我们使用**小根堆**来保存每个 `page` 的时间戳和 `page_id` （在C++中可以使用pair来维护这两个信息），我们每次只需要将堆顶取出即可找到应该被移除的 `page`，而且时间复杂度也不高 $log(n)$。但是该算法忽略的部分是，每次我们访问缓存中存在的`page` 时，我们需要对该 `page` 的时间戳进行修改！如果我们在堆中按照时间戳来维护 `page` 的话，我们的修改操作时间复杂度将是$O(N)$的。如[图1](#pic1)所示（图源自**Wikipedia**[<sup>1</sup>](#refer-anchor-1)），我们在访问 `page D`之后，需要对 `D` 的时间戳进行修改。

- 正确算法：
    - 我们使用 **双链表 + 哈希表** 实现 `LRU`。首先，双链表用来维护所有的 `page`，并且按照时间戳进行维护，时间戳越小的放在双链表越靠前的位置，反之则放在双链表越靠后的位置，这样我们可以$O(1)$时间复杂度找到时间戳最小的`page`；然后，为了解决之前提到的需要对`page`进行修改的操作，我们使用哈希表维护每个`page`在链表中的位置，即指针，当我们要访问一个`page`的时候，只需要查找哈希表找到`page`的位置即可，每次对`page`进行访问之后，我们在链表中删除该`page`，并将该`page`重新添加到链表的尾部，同时更新哈希表。
    - 当然，我们不需要自己实现双链表和哈希表，使用C++中强大的STL可以极大地简化我们的工作。STL中 `std::list<T>`实现了双链表，`std::unordered_map<Key,T>`实现了哈希表。
    - 为了验证自己掌握了该算法，可以到[leetcode](https://leetcode-cn.com/problems/lru-cache-lcci/) 题库进行验证。

### 代码实现
根据课程在主页上的声明，独立完成实验的编写很重要，因此这里不会将代码完整放出，只介绍部分接口实现代码。

`LRU`管理的是那些没有正在被内存使用的`frame`，为了区分磁盘上的`page`和缓存中管理的`page`，在内存中用`frame`来表示`page`。

`bool Victim(frame_id_t *frame_id)`:
- 函数功能：该函数的作用是从缓存中置换一个`frame`出去，并将`frame_id`保存在参数中。
- 返回值：如果能够置换出一个`frame`，则返回`true`，否则返回`false`
- 实现：将`list`中时间戳最大的`frame`取出，记录该`frame_id`到参数中，并更新`list`和`unordered_map`
```C++
if (lst_.empty()) {
    *frame_id = INVALID_PAGE_ID;
    return false;
}
*frame_id = lst_.back();
lst_.pop_back();
m_.erase(*frame_id);
return true;
```

`void Pin(frame_id_t frame_id)`:
- 函数功能：`pin`表示的是我们将要固定住一个`frame`，接下来我们要对这个`frame`进行操作，那么我们在进行页面置换操作的时候，就不应该考虑该`frame`了，也就是将一个`frame`从`LRU`管理器中移除。
- 返回值：void
- 实现：从 `unordered_map` 中找到 `frame_id` 对应的`frame`，并从 `list` 中移除
```C++
auto iter_m = m_.find(frame_id);
if (iter_m == m_.end()) {
  return;
}
auto iter_lst = m_[frame_id];
lst_.erase(iter_lst);
m_.erase(iter_m);
```

`void Unpin(frame_id_t frame_id)`:
- 函数功能：`Unpin`一个`frame`表示我们对该`frame`的操作已经结束了，该`frame`在接下来的操作中可以被移除，那么把它交个 `LRU` 管理器进行管理。也就是将`frame_i`d 添加到 `list` 和 `unordered_map` 中
- 返回值：void
- 实现：将 `frame_id` 添加到 `list` 中，并在 `unordered_map` 中添加 `{frame_id, iterator_on_list}`。 这里需要判断`LRU`能够管理的`frame`个数，如果超过了，则需要先从`LRU`中替换出一个`frame`，即调用 `victim`。
```C++
  auto iter_m = m_.find(frame_id);
  if (iter_m != m_.end()) {
    return;
  }
  lst_.push_front(frame_id);
  m_[frame_id] = lst_.begin();
  if (lst_.size() > num_pages_) {
    frame_id_t frame_id;
    Victim(&frame_id);
  }
```

## 总结
刷 `leetcode` 算法题目和实际应用中实现该算法还是有一些区别的，在这里我们不仅要考虑从 `LRU` 管理器中插入一个 `frame` 和 删除一个 `frame` 需要怎么实现，更多的是需要考虑什么时候会执行这些操作。

**LRU在实际应用场景的作用是管理那些曾经被使用过但是现在没有被正在使用的frame，正在被使用的frame不应该被移除，没有被使用过的frame不需要被移除（因为当有新的page从磁盘读入的时候，直接获取没有被使用过的frame即可）。**

1. 在我们要对一个 `frame` 进行操作的时候，我们应该将该 `frame` 从 `LRU` 中进行移除，即 `Pin` 操作
2. 当我们对一个 `frame` 操作完毕，我们应该将该 `frame` 交给 `LRU` 进行管理，当内存中没有空闲的 `frame` 的时候，可以将该 `frame` 作为候选移除 `frame`, 即 `Unpin` 操作
3. `LRU` 的空间复杂度: $O(n)$, 时间复杂度: $O(1)$

**参考**：
<div id="refer-anchor-1"></div>

- [1] [Cache replacement policies](https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_recently_used_(LRU))

>请各位看官批评指正，接受任何**善意**的建议
