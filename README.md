## Copy from [你所不知道的 C 語言: linked list 和非連續記憶體--Linked list 在 Linux 核心原始程式碼](https://hackmd.io/@sysprog/c-linked-list#%E4%BD%A0%E6%89%80%E4%B8%8D%E7%9F%A5%E9%81%93%E7%9A%84-C-%E8%AA%9E%E8%A8%80-linked-list-%E5%92%8C%E9%9D%9E%E9%80%A3%E7%BA%8C%E8%A8%98%E6%86%B6%E9%AB%94)

### 簡化的實作

Linux 核心原始程式碼變動快，為了後續探討方便，我們自 Linux 核心抽離出關鍵程式碼，建立 [sysprog21/linux-list](https://github.com/sysprog21/linux-list) 專案，讓學員得以理解箇中奧秘並進行相關實驗。

#### `container_of()`

為了提升程式碼的可攜性 (portability)，C89/C99 提供 [offsetof](https://man7.org/linux/man-pages/man3/offsetof.3.html) 巨集，可接受給定成員的型態及成員的名稱，傳回「成員的位址減去 struct 的起始位址」。示意如下:
![](https://hackmd.io/_uploads/SkfkiRJ2K.png)

[offsetof](https://man7.org/linux/man-pages/man3/offsetof.3.html) 定義於 `<stddef.h>`。

巨集 `container_of` 則在 [offsetof](https://man7.org/linux/man-pages/man3/offsetof.3.html) 的基礎上，擴充為「給定成員的位址、struct 的型態，及成員的名稱，傳回此 struct 的位址」，示意如下圖:
![](https://hackmd.io/_uploads/rJcei0k2F.png)

在 `container_of` 巨集出現前，程式設計的思維往往是:
1. 給定結構體起始地址
2. 求出結構體特定成員的記憶體內容
3. 傳回結構體成員的地址，作日後存取使用

`container_of` 巨集則逆轉上述流程，例如 `list_entry` 巨集利用 `container_of` 巨集，從 `struct list_head` 這個公開介面，「反向」去存取到自行定義的結構體開頭地址。

> 延伸閱讀: [Linux 核心原始程式碼巨集: `container_of`](https://hackmd.io/@sysprog/linux-macro-containerof)

#### `LIST_HEAD()` / `INIT_LIST_HEAD()`
```cpp
#define LIST_HEAD(head) struct list_head head = {&(head), &(head)}

static inline void INIT_LIST_HEAD(struct list_head *head) {
    head->next = head;
    head->prev = head;
}
```

初始化 `struct list_head`，先將 `next` 和 `prev` 都指向自身。`head` 指向的結構體之中的 `next` 成員表示 linked list 結構的開頭，而 `prev` 則指向結構體的結尾。

#### `list_add` / `list_add_tail`
```cpp
static inline void list_add(struct list_head *node, struct list_head *head) {
    struct list_head *next = head->next;

    next->prev = node;
    node->next = next;
    node->prev = head;
    head->next = node;
}

static inline void list_add_tail(struct list_head *node, struct list_head *head) {
    struct list_head *prev = head->prev;

    prev->next = node;
    node->next = head;
    node->prev = prev;
    head->prev = node;
}
```

將指定的 `node` 插入 linked list `head` 的開頭或者結尾。

#### `list_del`
```cpp
static inline void list_del(struct list_head *node) {
    struct list_head *next = node->next;
    struct list_head *prev = node->prev;

    next->prev = prev;
    prev->next = next;

#ifdef LIST_POISONING
    node->prev = (struct list_head *) (0x00100100);
    node->next = (struct list_head *) (0x00200200);
#endif
}
```

本函式將 `node` 從其所屬的 linked list 結構中移走。注意 `node` 本體甚至是包含 `node` 的結構所分配的記憶體，在此皆未被釋放，僅僅是將 `node` 從其原本的 linked list 連接移除。亦即，使用者需自行管理記憶體。

`LIST_POISONING` 巨集一旦定義，將讓已被移走的 `node` 節點在存取時，觸發作業系統的例外狀況：對於 `next` 或者 `prev` 的存取，會觸發執行時期的 invalid memory access (若系統禁止 predefined memory access)。

#### `list_del_init`
```cpp
static inline void list_del_init(struct list_head *node) {
    list_del(node);
    INIT_LIST_HEAD(node);
}
```

以 `list_del` 為基礎，但移除的 `node` 會額外呼叫 `INIT_LIST_HEAD` 把 `prev` 和 `next` 指向自身。

#### `list_empty`
```cpp
static inline int list_empty(const struct list_head *head) {
    return (head->next == head);
}
```

檢查 `head` 的 `next` 是否指向自身，確認 list 是否為 empty 狀態。

#### `list_singular`
```cpp
static inline int list_is_singular(const struct list_head *head) {
    return (!list_empty(head) && head->prev == head->next);
}
```

若 `head` 非 empty 狀態且 `prev` 和 `next` 是同一個節點，表示 linked list 只有一個節點。

#### `list_splice` / `list_splice_tail`
```cpp
static inline void list_splice(struct list_head *list, struct list_head *head) {
    struct list_head *head_first = head->next;
    struct list_head *list_first = list->next;
    struct list_head *list_last = list->prev;

    if (list_empty(list))
        return;

    head->next = list_first;
    list_first->prev = head;

    list_last->next = head_first;
    head_first->prev = list_last;
}

static inline void list_splice_tail(struct list_head *list,
                                    struct list_head *head) {
    struct list_head *head_last = head->prev;
    struct list_head *list_first = list->next;
    struct list_head *list_last = list->prev;

    if (list_empty(list))
        return;

    head->prev = list_last;
    list_last->next = head;

    list_first->prev = head_last;
    head_last->next = list_first;
}
```

將 `list` 的所有 node 都插入到 `head` 的開始 / 結束位置中。注意 `list` 本身仍維持原貌。

#### `list_spice_init` / `list_splice_tail_init`
```cpp
static inline void list_splice_init(struct list_head *list,
                                    struct list_head *head) {
    list_splice(list, head);
    INIT_LIST_HEAD(list);
}

static inline void list_splice_tail_init(struct list_head *list,
                                         struct list_head *head) {
    list_splice_tail(list, head);
    INIT_LIST_HEAD(list);
}
```

這二個函式類似 `list_splice` 及 `list_splice_tail`，但移除的 `list` 會額外呼叫 `INIT_LIST_HEAD` 把 `prev` 和 `next` 指向自身。

#### `list_cut_position`
```cpp
static inline void list_cut_position(struct list_head *head_to,
                                     struct list_head *head_from,
                                     struct list_head *node) {
    struct list_head *head_from_first = head_from->next;

    if (list_empty(head_from))
        return;

    if (head_from == node) {
        INIT_LIST_HEAD(head_to);
        return;
    }

    head_from->next = node->next;
    head_from->next->prev = head_from;

    head_to->prev = node;
    node->next = head_to;
    head_to->next = head_from_first;
    head_to->next->prev = head_to;
}
```

將從 `head_from` 的第一個節點到 `node` 間的一系列節點都移動到 `head_to` 上。`head_to` 必須是 empty 狀態 (`next` 和 `prev` 都指向自己)，否則可能發生 memory leak。

#### `list_move` / `list_move_tail`
```cpp
static inline void list_move(struct list_head *node, struct list_head *head) {
    list_del(node);
    list_add(node, head);
}

static inline void list_move_tail(struct list_head *node,
                                  struct list_head *head) {
    list_del(node);
    list_add_tail(node, head);
}
```

將 `node` 從原本的 linked list 移動到另一個 linked list `head` 的開頭或尾端。

#### `list_entry`
```cpp
#define list_entry(node, type, member) container_of(node, type, member)
```

`container_of` 等價的包裝，符合以 `list_` 開頭的命名慣例，此處的 entry 就是 list 內部的節點。

#### `list_first_entry` / `list_last_entry`
```cpp
#define list_first_entry(head, type, member) \
    list_entry((head)->next, type, member)

#define list_last_entry(head, type, member) \
    list_entry((head)->prev, type, member)
```

取得 linked list 的開頭或者結尾的 entry。

#### `list_for_each`
```cpp
#define list_for_each(node, head) \
    for (node = (head)->next; node != (head); node = node->next)
```

走訪整個 linked list。注意: `node` 和 `head` 不能在迴圈中被更改 (可能在多工環境中出現)，否則行為不可預期。

#### `list_for_each_entry`
```cpp
#ifdef __LIST_HAVE_TYPEOF
#define list_for_each_entry(entry, head, member)                       \
    for (entry = list_entry((head)->next, __typeof__(*entry), member); \
         &entry->member != (head);                                     \
         entry = list_entry(entry->member.next, __typeof__(*entry), member))
#endif
```

走訪包含 `struct list_head` 的另外一個結構之 entry。`node` 和 `head` 不能在迴圈中被更改，否則行為不可預期。
* 因為 `typeof` 之限制，只能在 GNUC 下使用

#### `list_for_each_safe` / `list_for_each_entry_safe`
```cpp
#define list_for_each_safe(node, safe, head)                     \
    for (node = (head)->next, safe = node->next; node != (head); \
         node = safe, safe = node->next)
         
#define list_for_each_entry_safe(entry, safe, head, member)                \
    for (entry = list_entry((head)->next, __typeof__(*entry), member),     \
        safe = list_entry(entry->member.next, __typeof__(*entry), member); \
         &entry->member != (head); entry = safe,                           \
        safe = list_entry(safe->member.next, __typeof__(*entry), member))
```

透過額外的 `safe` 紀錄每個疊代 (iteration) 所操作的節點的下一個節點，因此目前的節點可允許被移走，其他操作則同為不可預期行為。

### Linux 核心風格 Linked List 應用案例

從一些範例來看 Linux 核心風格 linked list 實際的使用方式。

#### Quick sort (遞迴版本)
詳見 [sysprog21/linux-list](https://github.com/sysprog21/linux-list) 專案中的  [quick-sort.c](https://github.com/sysprog21/linux-list/blob/master/examples/quick-sort.c) 程式碼。

```cpp
static void list_qsort(struct list_head *head) {
    struct list_head list_less, list_greater;
    struct listitem *pivot;
    struct listitem *item = NULL, *is = NULL;

    if (list_empty(head) || list_is_singular(head))
        return;

    INIT_LIST_HEAD(&list_less);
    INIT_LIST_HEAD(&list_greater);
```

先確認 `head` 所承載的 linked list 有兩個以上 entry，否則就返回，不用排序。以 `INIT_LIST_HEAD` 初始化另外兩個 list 結構，它們分別是用來插入 entry 中比 pivot 小或者其他的節點。

```cpp
    pivot = list_first_entry(head, struct listitem, list);
    list_del(&pivot->list);

    list_for_each_entry_safe (item, is, head, list) {
        if (cmpint(&item->i, &pivot->i) < 0)
            list_move_tail(&item->list, &list_less);
        else
            list_move(&item->list, &list_greater);
    }   
```

藉由 `list_first_entry` 取得第一個 entry 選為 pivot:
* `list_del` 將該 pivot entry 從 linked list 中移除
* 走訪整個 linked list，`cmpint` 回傳兩個指標中的值相減的數值，因此小於 `0` 意味著 `item->i` 的值比 `pivot` 的值小，加入 `list_less`，反之則同理

```cpp
    list_qsort(&list_less);
    list_qsort(&list_greater);

    list_add(&pivot->list, head);
    list_splice(&list_less, head);
    list_splice_tail(&list_greater, head);
}
```

藉由遞迴呼叫將 `list_less` 和 `list_greater` 排序。在 `list_for_each_entry_safe` 中，`list_move_tail` 和 `list_move` 會將所有原本在 `head` 中的節點移出，因此首先 `list_add` 加入 `pivot`，再把已經排好的 `list_less` 放在 pivot 前 `list_greater` 放在 pivot 後，完成排序。

#### Quick sort (非遞迴)

參考 [Optimized QuickSort: C Implementation (Non-Recursive)](https://alienryderflex.com/quicksort/)，嘗試實作非遞迴的 quick sort。

```cpp
static void list_qsort_no_recursive(struct list_head *head) {
    struct listitem *begin[MAX_LEN], *end[MAX_LEN], *L, *R;
    struct listitem pivot;
    int i = 0;

    begin[0] = list_first_entry(head, struct listitem, list);
    end[0] = list_last_entry(head, struct listitem, list);
```

`begin` 和 `end` 表示在 linked list 中的排序目標的開頭和結尾，因此最初是整個 linked list 的頭至尾。

```cpp
    while (i >= 0) {
        L = begin[i];
        R = end[i];
```

`begin` 和 `end` 的效果類似 stack，會填入每一輪要處理的節點開頭至結尾 ，因此先取出該輪的頭尾至 `L` 和 `R`。

```cpp
        if (L != R && &begin[i]->list != head) {
            // pivot is the actual address of L
            pivot = *begin[i];
            if (i == MAX_LEN - 1) {
                assert(-1);
                return;
            }
```

接著，以最開頭的節點作為 pivot。
* `i == MAX_LEN - 1` 的目的是額外檢查這輪如果填入 `begin` 和 `end` 是否會超出原本給定的陣列大小，因為我們所給予的空間是有限的

```cpp
            while (L != R) {
                while (R->i >= pivot.i && L != R)
                    R = list_entry(R->list.prev, struct listitem, list);
                if (L != R) {
                    L->i = R->i;
                    L = list_entry(L->list.next, struct listitem, list);
                }

                while (L->i <= pivot.i && L != R)
                    L = list_entry(L->list.next, struct listitem, list);
                if (L != R) {
                    R->i = L->i;
                    R = list_entry(R->list.prev, struct listitem, list);
                }
            }
```

否則的話，從結尾的點(`R`)一直往 `prev` 前進，找到比 `pivot` 值更小的節點的話就將其值移到開頭的 `L` 去。同理，從開頭的點(`L`)一直往 `next` 前進，找到比 `pivot` 值更大的節點的話，就將其值移到結尾的 `R` 去
* `L != R` 則負責判斷當 `L` 往 `next` 而 `R` 往 `prev` 移動碰在一起時，當 `L == R` 時，不再做上述的操作，離開迴圈

```cpp
            L->i = pivot.i;
            begin[i + 1] = list_entry(L->list.next, struct listitem, list);
            end[i + 1] = end[i];
            end[i++] = L;
```

此時 `L` 所在地方是 `pivot` 的值應在的正確位置，因此將 `pivot` 的值填入 `L`。此時需要被處理的排序是 `pivot` 往後到結尾的一段，兩個點分別是 `L` 的 `next`，和這輪的 `end[i]`
* 另一段則是 `pivot` 以前從原本的 `begin[i]` 到 `L` 一段

```cpp
        } else
            i--;
    }
}
```

如果 `L == R` 或者  `&begin[i]->list == head`，表示此段 linked list 已經不需要再做處理，`i--` 類似於 pop stack 的操作。
