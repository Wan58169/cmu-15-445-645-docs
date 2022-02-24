# PROJECT #2 - B+ TREE

## OVERVIEW

+ 实现一个基于`B+ Tree`的索引，使数据库能够实现快速检索功能
+ `index`主要的工作
  + 支持**插入**、**删除**和**查找**的功能
+ 简而言之，`index`实现的是类似于`STL`中`map`的功能

## TIPS

+ [PROJECT #1]()中的`page`在[PROJECT #2]()中被视为`BPlusTreePage`，用来存放`Tuples`。同时`BPlusTreePage`又分为两种：`BPlusTreeLeafPage`和`BPlusTreeInternalPage`
+ `BPlusTreePage`里有`size`个*slot*，每个*slot*里面存放一个`Tuple`
+ `B+ Tree`较为重要的是其`MaxSize`和`MinSize`，`Split`操作是以`MaxSize`为基准的，而`Redistribute`和`Coalesce`操作都是以`MinSize`为基准的

## TASK #1 - B+ TREE PAGES

### SRC

+ `BPlusTreePage` in `src/include/storage/page/b_plus_tree_page.h`和`src/page/b_plus_tree_page.cpp`
+ `BPlusTreeInternalPage` in `src/include/storage/page/b_plus_tree_internal_page.h`和`src/page/b_plus_tree_internal_page.cpp`
+ `BPlusTreeLeafPage` in `src/include/storage/page/b_plus_tree_leaf_page.h`和`src/storage/page/b_plus_tree_leaf_page.cpp`

### METHOD

1. 需要实现`BPlusTreePage`的辅助函数，例如

+ `IsLeafPage()`
+ `IsRootPage()`
+ `GetSize()`

```
* Header format (size in byte, 24 bytes in total):
* ----------------------------------------------------------------------------
* | PageType (4) | LSN (4) | CurrentSize (4) | MaxSize (4) |
* ----------------------------------------------------------------------------
* | ParentPageId (4) | PageId(4) |
* ----------------------------------------------------------------------------
```

2. 需要实现`BPlusTreeInternalPage`的辅助函数，例如

+ `Init()`
+ `ValueIndex(value)`：返回`value`所在的位置

+ `LookUp(key)`：在`Page`中找到`key`对应的`value`
+ `InsertNodeAfter(oldValue, newKey, newValue)`：在`value == oldValue`的*slot*后面插入`newKV`
+ `MoveHalfTo(recipient)`：将`page`中后半部分`KV`转移到`recipient`中
+ `Remove(index)`：删除`page[index]`上的`KV`
+ `MoveAllTo(recipient)`：将`page`中所有`KV`转移到`recipient`中
+ `MoveFirstToEndOf(recipient)`：将`page`中第一个`KV`转移到`recipient`尾部
+ `MoveLastToFrontOf(recipient)`：将`page`中最后一个`KV`插入到`recipient`头部

```
* Internal page format (keys are stored in increasing order):
*  --------------------------------------------------------------------------
* | HEADER | KEY(1)+PAGE_ID(1) | KEY(2)+PAGE_ID(2) | ... | KEY(n)+PAGE_ID(n) |
*  --------------------------------------------------------------------------
```

3. 需要实现`BPlusTreeLeafPage`的辅助函数，例如

+ `Init()`
+ `KeyIndex(key)`：返回`key`所在的位置
+ `Insert(key, value)`：将`KV`按序插入到`page`中
+ `MoveHalfTo(recipient)`
+ `LookUp(key, value)`
+ `MoveAllTo(recipient)`
+ `MoveFirstToEndOf(recipient)`
+ `MoveLastToFrontOf(recipient)`

```
* Leaf page format (keys are stored in order):
*  ----------------------------------------------------------------------
* | HEADER | KEY(1) + RID(1) | KEY(2) + RID(2) | ... | KEY(n) + RID(n)
*  ----------------------------------------------------------------------
```

### IMPLEMENTATION

1. 这是`BPlusTreePage`的辅助函数，较为简单

```C++
/*
 * Helper methods to get/set page type
 * Page type enum class is defined in b_plus_tree_page.h
 */
bool BPlusTreePage::IsLeafPage() const { return page_type_ == IndexPageType::LEAF_PAGE; }
bool BPlusTreePage::IsRootPage() const { return parent_page_id_ == INVALID_PAGE_ID; }
void BPlusTreePage::SetPageType(IndexPageType page_type) {
  page_type_ = page_type;
}

/*
 * Helper methods to get/set size (number of key/value pairs stored in that
 * page)
 */
int BPlusTreePage::GetSize() const { return size_; }
void BPlusTreePage::SetSize(int size) {
  size_ = size;
}
void BPlusTreePage::IncreaseSize(int amount) {
  size_ += amount;
}

/*
 * Helper methods to get/set max size (capacity) of the page
 */
int BPlusTreePage::GetMaxSize() const { return max_size_; }
void BPlusTreePage::SetMaxSize(int size) {
  max_size_ = size;
}

/*
 * Helper method to get min page size
 * Generally, min page size == max page size / 2
 */
int BPlusTreePage::GetMinSize() const { return max_size_/2; }

/*
 * Helper methods to get/set parent page id
 */
page_id_t BPlusTreePage::GetParentPageId() const { return parent_page_id_; }
void BPlusTreePage::SetParentPageId(page_id_t parent_page_id) {
  parent_page_id_ = parent_page_id;
}

/*
 * Helper methods to get/set self page id
 */
page_id_t BPlusTreePage::GetPageId() const { return page_id_; }
void BPlusTreePage::SetPageId(page_id_t page_id) {
  page_id_ = page_id;
}
```

2. 这是`BPlusTreeInternalPage`的辅助函数

```C++
/*****************************************************************************
 * HELPER METHODS AND UTILITIES
 *****************************************************************************/
/*
 * Init method after creating a new internal page
 * Including set page type, set current size, set page id, set parent id and set
 * max page size
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::Init(page_id_t page_id, page_id_t parent_id, int max_size) {
  SetPageType(IndexPageType::INTERNAL_PAGE);
  SetSize(1);
  SetPageId(page_id);
  SetParentPageId(parent_id);
  SetMaxSize(max_size);
}
/*
 * Helper method to get/set the key associated with input "index"(a.k.a
 * array offset)
 */
INDEX_TEMPLATE_ARGUMENTS
KeyType B_PLUS_TREE_INTERNAL_PAGE_TYPE::KeyAt(int index) const {
  // replace with your own code
  return array_[index].first;
}

INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::SetKeyAt(int index, const KeyType &key) {
  array_[index].first = key;
}

/*
 * Helper method to find and return array index(or offset), so that its value
 * equals to input "value"
 */
INDEX_TEMPLATE_ARGUMENTS
int B_PLUS_TREE_INTERNAL_PAGE_TYPE::ValueIndex(const ValueType &value) const {
  for(int i=0; i<GetSize(); i++) {
    if(array_[i].second == value) {
      return i;
    }
  }

  return INVALID_PAGE_ID;
}

/*
 * Helper method to get the value associated with input "index"(a.k.a array
 * offset)
 */
INDEX_TEMPLATE_ARGUMENTS
ValueType B_PLUS_TREE_INTERNAL_PAGE_TYPE::ValueAt(int index) const { 
  return array_[index].second; }
```

+ 需要注意，`Init()`函数将`size`设为1，即`BPlusTreeInternalPage`默认是有一个`KV`的，存放在`array[0]`

```C++
/*****************************************************************************
 * LOOKUP
 *****************************************************************************/
/*
 * Find and return the child pointer(page_id) which points to the child page
 * that contains input "key"
 * Start the search from the second key(the first key should always be invalid)
 */
INDEX_TEMPLATE_ARGUMENTS
ValueType B_PLUS_TREE_INTERNAL_PAGE_TYPE::Lookup(const KeyType &key, const KeyComparator &comparator) const {
  for(int i=1; i<GetSize(); i++) {
    /* 找到第一个比key大的 */
    if(comparator(KeyAt(i), key) > 0) {
      return ValueAt(i-1);
    }
  }

  /* the last key < key */
  return ValueAt(GetSize()-1);
}
```

+ 需要注意，在`LookUp`的过程中应该直接忽略`array[0]`，从`array[1]`开始查找。因为`array[0].key`并没有存放实际的数据，相反`array[0].value`非常重要，它存放的是一个泛化的指针，这对于构建索引来说非常重要！

```c++
/*****************************************************************************
 * INSERTION
 *****************************************************************************/
/*
 * Populate new root page with old_value + new_key & new_value
 * When the insertion cause overflow from leaf page all the way upto the root
 * page, you should create a new root page and populate its elements.
 * NOTE: This method is only called within InsertIntoParent()(b_plus_tree.cpp)
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::PopulateNewRoot(const ValueType &old_value, const KeyType &new_key,
                                                     const ValueType &new_value) {
  array_[0] = MappingType{new_key, old_value};
  array_[1] = MappingType{new_key, new_value};
  IncreaseSize(1);
}
/*
 * Insert new_key & new_value pair right after the pair with its value ==
 * old_value
 * @return:  new size after insertion
 */
INDEX_TEMPLATE_ARGUMENTS
int B_PLUS_TREE_INTERNAL_PAGE_TYPE::InsertNodeAfter(const ValueType &old_value, const KeyType &new_key,
                                                    const ValueType &new_value) {
  int idx = ValueIndex(old_value);

  idx++;  // new_key's idx
  IncreaseSize(1);

  /* shift one by one */
  for(int i=GetSize()-1; i>idx; i--) {
    array_[i] = array_[i-1];
  }
  array_[idx] = MappingType{new_key, new_value};

  return GetSize();
}
```

+ 在`InsertNodeAfter`中需要找出`oldValue`所在位置，然后其后的所有元素往后挪为`newKV`腾出空间

```C++
/*****************************************************************************
 * SPLIT
 *****************************************************************************/
/*
 * Remove half of key & value pairs from this page to "recipient" page
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::MoveHalfTo(BPlusTreeInternalPage *recipient,
                                                BufferPoolManager *buffer_pool_manager) {
  int idx = GetMinSize();
  int num = GetSize()-1 - idx;

  recipient->CopyNFrom(array_+1+idx, num, buffer_pool_manager);

  IncreaseSize(-num);
}

/* Copy entries into me, starting from {items} and copy {size} entries.
 * Since it is an internal page, for all entries (pages) moved, their parents page now changes to me.
 * So I need to 'adopt' them by changing their parent page id, which needs to be persisted with BufferPoolManger
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::CopyNFrom(MappingType *items, int size, BufferPoolManager *buffer_pool_manager) {
  std::copy(items, items+size, array_+1);
  IncreaseSize(size);

  for(int i=0; i<=size; i++) {
    Page *childPage = buffer_pool_manager->FetchPage(ValueAt(i));
    BPlusTreePage *childNode = reinterpret_cast<BPlusTreePage*>(childPage);

    childNode->SetParentPageId(GetPageId());

    buffer_pool_manager->UnpinPage(childPage->GetPageId(), true);
  }
}

INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::RemoveFirstKey() {
  for(int i=1; i<GetSize(); i++) {
    array_[i].first = array_[i+1].first;
  }
  for(int i=0; i<GetSize()-1; i++) {
    array_[i].second = array_[i+1].second;
  }
  IncreaseSize(-1);
}
```

+ `MoveHalfTo`函数就是将`page`的后半部分`KV`转移到`recipient`中，需要更新`size`

```c++
/*****************************************************************************
 * REMOVE
 *****************************************************************************/
/*
 * Remove the key & value pair in internal page according to input index(a.k.a
 * array offset)
 * NOTE: store key&value pair continuously after deletion
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::Remove(int index) {
  IncreaseSize(-1);
  for(int i=index; i<GetSize(); i++) {
    array_[i] = array_[i+1];
  }
}

/*
 * Remove the only key & value pair in internal page and return the value
 * NOTE: only call this method within AdjustRoot()(in b_plus_tree.cpp)
 */
INDEX_TEMPLATE_ARGUMENTS
ValueType B_PLUS_TREE_INTERNAL_PAGE_TYPE::RemoveAndReturnOnlyChild() {
  SetSize(0);
  return ValueAt(0);
}
```

+ `Remove`函数就是删除指定位置的`KV`

```c++
/*****************************************************************************
 * MERGE
 *****************************************************************************/
/*
 * Remove all of key & value pairs from this page to "recipient" page.
 * The middle_key is the separation key you should get from the parent. You need
 * to make sure the middle key is added to the recipient to maintain the invariant.
 * You also need to use BufferPoolManager to persist changes to the parent page id for those
 * pages that are moved to the recipient
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::MoveAllTo(BPlusTreeInternalPage *recipient, const KeyType &middle_key,
                                               BufferPoolManager *buffer_pool_manager) {
  SetKeyAt(0, middle_key);
  while(GetSize() != 0) {
    MoveFirstToEndOf(recipient, middle_key, buffer_pool_manager);
  }
  SetSize(0);
}
```

+ `MoveAllTo`函数就是将`page`中的所有`KV`转移到`recipient`中

```c++
/*****************************************************************************
 * REDISTRIBUTE
 *****************************************************************************/
/*
 * Remove the first key & value pair from this page to tail of "recipient" page.
 *
 * The middle_key is the separation key you should get from the parent. You need
 * to make sure the middle key is added to the recipient to maintain the invariant.
 * You also need to use BufferPoolManager to persist changes to the parent page id for those
 * pages that are moved to the recipient
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::MoveFirstToEndOf(BPlusTreeInternalPage *recipient, const KeyType &middle_key,
                                                      BufferPoolManager *buffer_pool_manager) {
  SetKeyAt(0, middle_key);
  recipient->CopyLastFrom(array_[0], buffer_pool_manager);

  IncreaseSize(-1);
  /* shift to head one by one */
  for(int i=0; i<GetSize(); i++) {
    array_[i] = array_[i+1];
  }
}

/* Append an entry at the end.
 * Since it is an internal page, the moved entry(page)'s parent needs to be updated.
 * So I need to 'adopt' it by changing its parent page id, which needs to be persisted with BufferPoolManger
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::CopyLastFrom(const MappingType &pair, BufferPoolManager *buffer_pool_manager) {
  IncreaseSize(1);
  array_[GetSize()-1] = pair;

  Page *childPage = buffer_pool_manager->FetchPage(ValueAt(GetSize()-1));
  BPlusTreePage *childNode = reinterpret_cast<BPlusTreePage*>(childPage);
  childNode->SetParentPageId(GetPageId());
  buffer_pool_manager->UnpinPage(childPage->GetPageId(), true);
}

/*
 * Remove the last key & value pair from this page to head of "recipient" page.
 * You need to handle the original dummy key properly, e.g. updating recipient’s array to position the middle_key at the
 * right place.
 * You also need to use BufferPoolManager to persist changes to the parent page id for those pages that are
 * moved to the recipient
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::MoveLastToFrontOf(BPlusTreeInternalPage *recipient, const KeyType &middle_key,
                                                       BufferPoolManager *buffer_pool_manager) {
  recipient->SetKeyAt(0, middle_key);

  recipient->CopyFirstFrom(array_[GetSize()-1], buffer_pool_manager);
  IncreaseSize(-1);
}

/* Append an entry at the beginning.
 * Since it is an internal page, the moved entry(page)'s parent needs to be updated.
 * So I need to 'adopt' it by changing its parent page id, which needs to be persisted with BufferPoolManger
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::CopyFirstFrom(const MappingType &pair, BufferPoolManager *buffer_pool_manager) {
  /* shift to tail one by one */
  for(int i=0; i<GetSize(); i++) {
    array_[i+1] = array_[i];
  }

  IncreaseSize(1);
  array_[0] = pair;

  Page *childPage = buffer_pool_manager->FetchPage(ValueAt(0));
  BPlusTreePage *childNode = reinterpret_cast<BPlusTreePage*>(childPage);
  childNode->SetParentPageId(GetPageId());
  buffer_pool_manager->UnpinPage(childPage->GetPageId(), true);
}
```

+ 主要是转移函数，需要注意下标以及数值拷贝问题

3. 这是`BPlusTreeLeafPage`的辅助函数

```c++
/*****************************************************************************
 * HELPER METHODS AND UTILITIES
 *****************************************************************************/

/**
 * Init method after creating a new leaf page
 * Including set page type, set current size to zero, set page id/parent id, set
 * next page id and set max size
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_LEAF_PAGE_TYPE::Init(page_id_t page_id, page_id_t parent_id, int max_size) {
  SetPageType(IndexPageType::LEAF_PAGE);
  SetSize(0);
  SetPageId(page_id);
  SetParentPageId(parent_id);
  SetMaxSize(max_size);
  SetNextPageId(INVALID_PAGE_ID);
}

/**
 * Helper methods to set/get next page id
 */
INDEX_TEMPLATE_ARGUMENTS
page_id_t B_PLUS_TREE_LEAF_PAGE_TYPE::GetNextPageId() const { return next_page_id_; }

INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_LEAF_PAGE_TYPE::SetNextPageId(page_id_t next_page_id) {
  next_page_id_ = next_page_id;
}

/**
 * Helper method to find the first index i so that array[i].first >= key
 * NOTE: This method is only used when generating index iterator
 */
INDEX_TEMPLATE_ARGUMENTS
int B_PLUS_TREE_LEAF_PAGE_TYPE::KeyIndex(const KeyType &key, const KeyComparator &comparator) const {
  for(int i=0; i<GetSize(); i++) {
    if(comparator(array_[i].first, key) >= 0) {
      return i;
    }
  }

  return INVALID_PAGE_ID;
}

/*
 * Helper method to find and return the key associated with input "index"(a.k.a
 * array offset)
 */
INDEX_TEMPLATE_ARGUMENTS
KeyType B_PLUS_TREE_LEAF_PAGE_TYPE::KeyAt(int index) const {
  // replace with your own code
  return array_[index].first;
}

/*
 * Helper method to find and return the key & value pair associated with input
 * "index"(a.k.a array offset)
 */
INDEX_TEMPLATE_ARGUMENTS
const MappingType &B_PLUS_TREE_LEAF_PAGE_TYPE::GetItem(int index) {
  // replace with your own code
  return array_[index];
}
```

+ 需要注意，`BPlusTreeLeafPage`的`array[0].key`是存放实际数据的，与`BPlusTreeInternalPage`恰恰相反！

```c++
/*****************************************************************************
 * INSERTION
 *****************************************************************************/
/*
 * Insert key & value pair into leaf page ordered by key
 * @return  page size after insertion
 */
INDEX_TEMPLATE_ARGUMENTS
int B_PLUS_TREE_LEAF_PAGE_TYPE::Insert(const KeyType &key, const ValueType &value, const KeyComparator &comparator) {
  int idx = KeyIndex(key, comparator);  // the first idx so that array[i].first >= key

  IncreaseSize(1);  // for new MappingType element

  if(idx == INVALID_PAGE_ID) {  // no found
    array_[GetSize()-1] = MappingType{key, value};
  }
  else {
    /* shift one by one */
    for(int i=GetSize()-1; i>idx; i--) {
      array_[i] = array_[i-1];
    }
    array_[idx] = MappingType{key, value};
  }

  return GetSize();
}
```

+ 需要注意，中间插入数据，后面的部分需要往后挪为其腾出空间

```c++
/*****************************************************************************
 * SPLIT
 *****************************************************************************/
/*
 * Remove half of key & value pairs from this page to "recipient" page
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_LEAF_PAGE_TYPE::MoveHalfTo(BPlusTreeLeafPage *recipient) {
  int idx = GetMinSize();
  int num = GetSize() - idx;

  recipient->CopyNFrom(array_+idx, num);

  IncreaseSize(-num);
}

/*
 * Copy starting from items, and copy {size} number of elements into me.
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_LEAF_PAGE_TYPE::CopyNFrom(MappingType *items, int size) {
  std::copy(items, items+size, array_);
  IncreaseSize(size);
}

/*****************************************************************************
 * LOOKUP
 *****************************************************************************/
/*
 * For the given key, check to see whether it exists in the leaf page. If it
 * does, then store its corresponding value in input "value" and return true.
 * If the key does not exist, then return false
 */
INDEX_TEMPLATE_ARGUMENTS
bool B_PLUS_TREE_LEAF_PAGE_TYPE::Lookup(const KeyType &key, ValueType *value, const KeyComparator &comparator) const {
  int idx = KeyIndex(key, comparator);

  if(idx==INVALID_PAGE_ID || comparator(array_[idx].first, key)!=0) {
    return false;
  }

  *value = array_[idx].second;
  return true;
}

/*****************************************************************************
 * REMOVE
 *****************************************************************************/
/*
 * First look through leaf page to see whether delete key exist or not. If
 * exist, perform deletion, otherwise return immediately.
 * NOTE: store key&value pair continuously after deletion
 * @return   page size after deletion
 */
INDEX_TEMPLATE_ARGUMENTS
int B_PLUS_TREE_LEAF_PAGE_TYPE::RemoveAndDeleteRecord(const KeyType &key, const KeyComparator &comparator) {
  /* step1. see whether delete key exist of not */
  int idx = KeyIndex(key, comparator);

  if(idx == INVALID_PAGE_ID) { return GetSize(); }

  /* step2. exist, deletion */
  if(comparator(array_[idx].first, key) == 0) {
    IncreaseSize(-1);
    /* shift one by one */
    for(int i=idx; i<GetSize(); i++) {
      array_[i] = array_[i + 1];
    }
  }

  return GetSize();
}

/*****************************************************************************
 * MERGE
 *****************************************************************************/
/*
 * Remove all of key & value pairs from this page to "recipient" page. Don't forget
 * to update the next_page id in the sibling page
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_LEAF_PAGE_TYPE::MoveAllTo(BPlusTreeLeafPage *recipient) {
  while(GetSize() != 0) {
    MoveFirstToEndOf(recipient);
  }
  SetSize(0);
}

/*****************************************************************************
 * REDISTRIBUTE
 *****************************************************************************/
/*
 * Remove the first key & value pair from this page to "recipient" page.
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_LEAF_PAGE_TYPE::MoveFirstToEndOf(BPlusTreeLeafPage *recipient) {
  recipient->CopyLastFrom(array_[0]);

  IncreaseSize(-1);
  /* shift to head one by one */
  for(int i=0; i<GetSize(); i++) {
    array_[i] = array_[i+1];
  }
}

/*
 * Copy the item into the end of my item list. (Append item to my array)
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_LEAF_PAGE_TYPE::CopyLastFrom(const MappingType &item) {
  IncreaseSize(1);
  array_[GetSize()-1] = item;
}

/*
 * Remove the last key & value pair from this page to "recipient" page.
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_LEAF_PAGE_TYPE::MoveLastToFrontOf(BPlusTreeLeafPage *recipient) {
  recipient->CopyFirstFrom(array_[GetSize()-1]);
  IncreaseSize(-1);
}

/*
 * Insert item at the front of my items. Move items accordingly.
 */
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_LEAF_PAGE_TYPE::CopyFirstFrom(const MappingType &item) {
  IncreaseSize(1);
  /* shift to tail one by one */
  for(int i=1; i<GetSize(); i++) {
    array_[i] = array_[i-1];
  }

  array_[0] = item;
}
```

+ 与`BPlusTreeInternalPage`类似

## TASK #2.A - B+TREE DATA STRUCTURE (INSERTION & POINT SEARCH)

### SRC

+ `src/include/index/b_plus_tree.h`和`src/index/b_plus_tree.cpp`

### HINT

+ 无论是`GetValue()`还是`Insert()`，我们都需要定位到`key`所在的`page`，即都需要用到`FindLeafPage()`
+ 在`Insert`一个`KV`后，如果`page`超出了`size`限制，则需要进行`Split`操作

### METHOD

+ `FindLeafPage(key)`：从`B+ Tree`根节点出发，定位到`key`所在的`page`
+ `GetValue(key)`：返回`key`对应的`value`
+ `Insert(key, value)`：向`B+ Tree`中插入`KV`
+ `Split(node)`：分裂函数

### IMPLEMENTATION

1. `FindLeafPage(key)`函数

```c++
/*****************************************************************************
 * UTILITIES AND DEBUG
 *****************************************************************************/
/*
 * Find leaf page containing particular key, if leftMost flag == true, find
 * the left most leaf page
 */
INDEX_TEMPLATE_ARGUMENTS
Page *BPLUSTREE_TYPE::FindLeafPage(const KeyType &key, bool leftMost) {
  if(IsEmpty()) {
    return nullptr;
  }

  /* get root page and cast to the BPlusTreePoage */
  Page *page = buffer_pool_manager_->FetchPage(root_page_id_);
  BPlusTreePage *node = reinterpret_cast<BPlusTreePage*>(page);

  /* loop util leaf page */
  while(!node->IsLeafPage()) {
    InternalPage *internalNode = reinterpret_cast<InternalPage*>(node);

    page_id_t childPageId = INVALID_PAGE_ID;
    if(leftMost) {
      childPageId = internalNode->ValueAt(0);
    }
    else {
      childPageId = internalNode->Lookup(key, comparator_);
    }

    buffer_pool_manager_->UnpinPage(page->GetPageId(), false);  // unpin now page
    page = buffer_pool_manager_->FetchPage(childPageId);                // pin child page
    node = reinterpret_cast<BPlusTreePage*>(page);
  }

  /* unpin the page */
  buffer_pool_manager_->UnpinPage(page->GetPageId(), false);

  return page;
}
```

+ 需要注意，`bpm`管理，即`Fetch`和`Unpin`；以及循环判断是否已到`leaf level`节点

2. `GetValue()`函数

```c++
/*****************************************************************************
 * SEARCH
 *****************************************************************************/
/*
 * Return the only value that associated with input key
 * This method is used for point query
 * @return : true means key exists
 */
INDEX_TEMPLATE_ARGUMENTS
bool BPLUSTREE_TYPE::GetValue(const KeyType &key, std::vector<ValueType> *result, Transaction *transaction) {
  std::lock_guard<std::mutex> lck(mtx_);
  /* step1. find the leaf page contained key */
  Page *leafPage = FindLeafPage(key, false);

  if(leafPage == nullptr) { return false; }

  /* step2. find the key in the leaf page */
  LeafPage *leafNode = reinterpret_cast<LeafPage*>(leafPage);

  ValueType value;
  bool isExist = leafNode->Lookup(key, &value, comparator_);

  if(isExist) {
    result->push_back(value);
  }

  return isExist;
}
```

3. `Insert(key, value)`函数

```c++
/*****************************************************************************
 * INSERTION
 *****************************************************************************/
/*
 * Insert constant key & value pair into b+ tree
 * if current tree is empty, start new tree, update root page id and insert
 * entry, otherwise insert into leaf page.
 * @return: since we only support unique key, if user try to insert duplicate
 * keys return false, otherwise return true.
 */
INDEX_TEMPLATE_ARGUMENTS
bool BPLUSTREE_TYPE::Insert(const KeyType &key, const ValueType &value, Transaction *transaction) {
  std::lock_guard<std::mutex> lck(mtx_);

  if(IsEmpty()) {
    StartNewTree(key, value);
    return true;
  }
  return InsertIntoLeaf(key, value, transaction);
}
/*
 * Insert constant key & value pair into an empty tree
 * User needs to first ask for new page from buffer pool manager(NOTICE: throw
 * an "out of memory" exception if returned value is nullptr), then update b+
 * tree's root page id and insert entry directly into leaf page.
 */
INDEX_TEMPLATE_ARGUMENTS
void BPLUSTREE_TYPE::StartNewTree(const KeyType &key, const ValueType &value) {
  /* step1. ask for new page from buffer pool manager as root page */
  Page *rootPage = buffer_pool_manager_->NewPage(&root_page_id_);

  /* step2. update b+ tree's root page id */
  UpdateRootPageId(1);

  /* step3. insert entry into leaf page */
  LeafPage *rootNode = reinterpret_cast<LeafPage *>(rootPage);
  rootNode->Init(root_page_id_, INVALID_PAGE_ID, leaf_max_size_);
  rootNode->Insert(key, value, comparator_);
  buffer_pool_manager_->UnpinPage(rootPage->GetPageId(), true);
}

/*
 * Insert constant key & value pair into leaf page
 * User needs to first find the right leaf page as insertion target, then look
 * through leaf page to see whether insert key exist or not. If exist, return
 * immdiately, otherwise insert entry. Remember to deal with split if necessary.
 * @return: since we only support unique key, if user try to insert duplicate
 * keys return false, otherwise return true.
 */
INDEX_TEMPLATE_ARGUMENTS
bool BPLUSTREE_TYPE::InsertIntoLeaf(const KeyType &key, const ValueType &value, Transaction *transaction) {
  /* step1. find the right leaf page */
  Page *leafPage = FindLeafPage(key, false);
  LeafPage *leafNode = reinterpret_cast<LeafPage*>(leafPage);

  /* step2. look through leaf page to see whether insert key exist or not */
  ValueType tmpValue;
  bool isExist = leafNode->Lookup(key, &tmpValue, comparator_);

  /* step3. If exist, return immdiately */
  if(isExist) {
    return false;
  }

  /* step4. otherwise insert entry */
  int size = leafNode->Insert(key, value, comparator_);

  /* Attention. remeber to deal with split if necessary */
  if(size >= leafNode->GetMaxSize()) {
    LeafPage *newLeafNode = Split(leafNode);
    InsertIntoParent(leafNode, newLeafNode->KeyAt(0), newLeafNode, transaction);
  }

  return true;
}
```

+ 需要注意，在向`leaf page`插入`KV`后，如果超出`size`限制，则需要进行`Split`操作

4. `Split(node)`函数

```c++
/*
 * Split input page and return newly created page.
 * Using template N to represent either internal page or leaf page.
 * User needs to first ask for new page from buffer pool manager(NOTICE: throw
 * an "out of memory" exception if returned value is nullptr), then move half
 * of key & value pairs from input page to newly created page
 */
INDEX_TEMPLATE_ARGUMENTS
template <typename N>
N *BPLUSTREE_TYPE::Split(N *node) {
  /* step1. first ask for new page from buffer pool manager */
  page_id_t newPageId;
  Page *newPage = buffer_pool_manager_->NewPage(&newPageId);

  if(newPage == nullptr) {
    throw Exception(ExceptionType::OUT_OF_MEMORY, "out of memory");
  }

  /* step2. move half of key & value pairs from input page to newly created page */
  if(node->IsLeafPage()) {
    LeafPage *oldLeafNode = reinterpret_cast<LeafPage*>(node);
    LeafPage *newLeafNode = reinterpret_cast<LeafPage*>(newPage);
    /* shift half pairs to newLeafNode */
    newLeafNode->Init(newPageId, oldLeafNode->GetParentPageId(), leaf_max_size_);

    oldLeafNode->MoveHalfTo(newLeafNode);
    newLeafNode->SetNextPageId(oldLeafNode->GetNextPageId());
    oldLeafNode->SetNextPageId(newLeafNode->GetPageId());

    buffer_pool_manager_->UnpinPage(newPage->GetPageId(), true);
    /* return newly created page */
    return reinterpret_cast<N*>(newLeafNode);
  }

  InternalPage *oldInternalNode = reinterpret_cast<InternalPage*>(node);
  InternalPage *newInternalNode = reinterpret_cast<InternalPage*>(newPage);
  /* shift half pairs to newInternalNode */
  newInternalNode->Init(newPageId, oldInternalNode->GetParentPageId(), internal_max_size_);

  oldInternalNode->MoveHalfTo(newInternalNode, buffer_pool_manager_);

  buffer_pool_manager_->UnpinPage(newPage->GetPageId(), true);
  /* return newly created page */
  return reinterpret_cast<N*>(newInternalNode);
}

/*
 * Insert key & value pair into internal page after split
 * @param   old_node      input page from split() method
 * @param   key
 * @param   new_node      returned page from split() method
 * User needs to first find the parent page of old_node, parent node must be
 * adjusted to take info of new_node into account. Remember to deal with split
 * recursively if necessary.
 */
INDEX_TEMPLATE_ARGUMENTS
void BPLUSTREE_TYPE::InsertIntoParent(BPlusTreePage *old_node, const KeyType &key, BPlusTreePage *new_node,
                                      Transaction *transaction) {
  /* step1. first find the parent page of old_node */
  if(old_node->IsRootPage()) {
    page_id_t newPageId;
    Page *newPage = buffer_pool_manager_->NewPage(&newPageId);
    root_page_id_ = newPageId;

    UpdateRootPageId(0);
    InternalPage *newRootNode = reinterpret_cast<InternalPage*>(newPage);
    newRootNode->Init(newPageId, INVALID_PAGE_ID, internal_max_size_);
    newRootNode->PopulateNewRoot(old_node->GetPageId(), key, new_node->GetPageId());

    /* less keys in internal page */
    if(!old_node->IsLeafPage()) {
      InternalPage *newNode = reinterpret_cast<InternalPage*>(new_node);
      newNode->RemoveFirstKey();
    }

    old_node->SetParentPageId(newRootNode->GetPageId());
    new_node->SetParentPageId(newRootNode->GetPageId());
    buffer_pool_manager_->UnpinPage(newPage->GetPageId(), true);

    return;
  }

  /* step2. insert into parent */
  Page *parentPage = buffer_pool_manager_->FetchPage(old_node->GetParentPageId());
  InternalPage *parentNode = reinterpret_cast<InternalPage*>(parentPage);

  parentNode->InsertNodeAfter(old_node->GetPageId(), key, new_node->GetPageId());

  /* less keys in internal page */
  if(!old_node->IsLeafPage()) {
    InternalPage *newNode = reinterpret_cast<InternalPage*>(new_node);
    newNode->RemoveFirstKey();
  }

  new_node->SetParentPageId(parentNode->GetPageId());
  buffer_pool_manager_->UnpinPage(parentPage->GetPageId(), true);

  /* Attention. remeber to deal with split recursively if necessary */
  if(parentNode->GetSize() > parentNode->GetMaxSize()) {
    InternalPage *newParentNode = Split(parentNode);
    InsertIntoParent(parentNode, newParentNode->KeyAt(1), newParentNode, transaction);
  }
}
```

+ 需要注意，在分裂成两半后，如果旧节点是根节点的话，则创建一个新的父节点用来串联新旧两叶子节点；反之，则将新节点的`array[0].key`插入到旧节点的父节点中。
+ 如果父节点在其`level`上依然超出`size`限制，则继续递归，继续分裂

## TASK #2.B - B+TREE DATA STRUCTURE (DELETION)

### SRC

+ `src/index/b_plus_tree.h`和`src/index/b_plus_tree.cpp`

### HINT

+ 与[TASK #2.A]()一样，在执行`Remove`操作之前同样需要定位到`key`所在的`page`，即`FindLeafPage()`
+ 先`RemoveAndDeleteRecord`再说，如果删除`Tuple`后导致`size`小于`minSize`，则需要进行`Redistribute`或`Coalesce`补救
+ 如果邻亲节点的`size`大于`minSize`，则进行`Redistribute`操作；反之进行`Coalesce`操作。邻亲节点的`size`大于`minSize`意思就是邻亲节点有多余的`Tuple`，即有条件向节点伸出援手
+ 在`Coalesce()`里可能需要递归处理父节点

### METHOD

+ `Remove()`
+ `CoalesceOrRedistribute()`
+ `Coalesce()`
+ `Redistribute()`
+ `AdjustRoot()`：`CoalesceOrRedistribute()`的辅助函数

### IMPLEMENTATION

```c++
/*****************************************************************************
 * REMOVE
 *****************************************************************************/
/*
 * Delete key & value pair associated with input key
 * If current tree is empty, return immdiately.
 * If not, User needs to first find the right leaf page as deletion target, then
 * delete entry from leaf page. Remember to deal with redistribute or merge if
 * necessary.
 */
INDEX_TEMPLATE_ARGUMENTS
void BPLUSTREE_TYPE::Remove(const KeyType &key, Transaction *transaction) {
  std::lock_guard<std::mutex> lck(mtx_);
  /* step1. is empty? */
  if(IsEmpty()) {
    return;
  }

  /* step2. find the right leaf page */
  Page *leafPage = FindLeafPage(key, false);

  if(leafPage == nullptr) { return; }

  LeafPage *leafNode = reinterpret_cast<LeafPage*>(leafPage);

  /* step3. delete entry from leaf page */
  int delSize = leafNode->RemoveAndDeleteRecord(key, comparator_);

  /* step4. redistribute or merge */
  if(delSize < leafNode->GetMinSize()) {
    CoalesceOrRedistribute(leafNode, transaction);
  }
}

/*
 * User needs to first find the sibling of input page. If sibling's size + input
 * page's size > page's max size, then redistribute. Otherwise, merge.
 * Using template N to represent either internal page or leaf page.
 * @return: true means target leaf page should be deleted, false means no
 * deletion happens
 */
INDEX_TEMPLATE_ARGUMENTS
template <typename N>
bool BPLUSTREE_TYPE::CoalesceOrRedistribute(N *node, Transaction *transaction) {
  /* step1. except root */
  if(node->IsRootPage()) {
    return AdjustRoot(node);
  }

  /* step2. first find the sibling */
  Page *parentPage = buffer_pool_manager_->FetchPage(node->GetParentPageId());
  InternalPage *parentNode = reinterpret_cast<InternalPage*>(parentPage);
  int idx = parentNode->ValueIndex(node->GetPageId());

  page_id_t siblingPageId = parentNode->ValueAt(idx==0 ? 1 : idx-1);
  Page *siblingPage = buffer_pool_manager_->FetchPage(siblingPageId);
  N* siblingNode = reinterpret_cast<N*>(siblingPage);

  /* step3. redistribute or merge */
  bool isDeleted;
  if(siblingNode->GetSize() > siblingNode->GetMinSize()) {
    Redistribute(siblingNode, node, idx);
    isDeleted = false;
  }
  else {
    isDeleted = Coalesce(&siblingNode, &node, &parentNode, idx, transaction);
  }

  buffer_pool_manager_->UnpinPage(parentPage->GetPageId(), true);
  buffer_pool_manager_->UnpinPage(siblingPage->GetPageId(), true);

  return isDeleted;
}

/*
 * Move all the key & value pairs from one page to its sibling page, and notify
 * buffer pool manager to delete this page. Parent page must be adjusted to
 * take info of deletion into account. Remember to deal with coalesce or
 * redistribute recursively if necessary.
 * Using template N to represent either internal page or leaf page.
 * @param   neighbor_node      sibling page of input "node"
 * @param   node               input from method coalesceOrRedistribute()
 * @param   parent             parent page of input "node"
 * @return  true means parent node should be deleted, false means no deletion
 * happend
 */
INDEX_TEMPLATE_ARGUMENTS
template <typename N>
bool BPLUSTREE_TYPE::Coalesce(N **neighbor_node, N **node,
                              BPlusTreeInternalPage<KeyType, page_id_t, KeyComparator> **parent, int index,
                              Transaction *transaction) {
  /* step1. pick up sibling and node */
  int keyIdx = index;
  if(index == 0) {
    std::swap(neighbor_node,node);
    keyIdx = 1;
  }
  auto midKey = (*parent)->KeyAt(keyIdx);

  /* step2. move all the key & value pairs from one page to its sibling page */
  if((*node)->IsLeafPage()) {
    LeafPage *siblingLeafNode = reinterpret_cast<LeafPage*>(*neighbor_node);
    LeafPage *leafNode = reinterpret_cast<LeafPage*>(*node);
    leafNode->MoveAllTo(siblingLeafNode);
    siblingLeafNode->SetNextPageId(leafNode->GetNextPageId());
  }
  else {
    InternalPage *siblingInternalNode = reinterpret_cast<InternalPage*>(*neighbor_node);
    InternalPage *internalNode = reinterpret_cast<InternalPage*>(*node);
    internalNode->MoveAllTo(siblingInternalNode, midKey, buffer_pool_manager_);
  }

  /* step3. notify bpm to delete this page */
  buffer_pool_manager_->DeletePage((*node)->GetPageId());

  /* step4. parent page update */
  (*parent)->Remove(keyIdx);

  /* step5. coalesce or redistribute */
  if((*parent)->GetSize() < (*parent)->GetMinSize()) {
    return CoalesceOrRedistribute(*parent, transaction);
  }

  return true;
}

/*
 * Redistribute key & value pairs from one page to its sibling page. If index ==
 * 0, move sibling page's first key & value pair into end of input "node",
 * otherwise move sibling page's last key & value pair into head of input
 * "node".
 * Using template N to represent either internal page or leaf page.
 * @param   neighbor_node      sibling page of input "node"
 * @param   node               input from method coalesceOrRedistribute()
 */
INDEX_TEMPLATE_ARGUMENTS
template <typename N>
void BPLUSTREE_TYPE::Redistribute(N *neighbor_node, N *node, int index) {
  Page *parentPage = buffer_pool_manager_->FetchPage(node->GetParentPageId());
  InternalPage *parentNode = reinterpret_cast<InternalPage*>(parentPage);

  if(node->IsLeafPage()) {
    LeafPage *leafNode = reinterpret_cast<LeafPage*>(node);
    LeafPage *siblingLeafNode = reinterpret_cast<LeafPage*>(neighbor_node);
    if(index == 0) {  // node -> sibling
      siblingLeafNode->MoveFirstToEndOf(leafNode);
      parentNode->SetKeyAt(1, siblingLeafNode->KeyAt(0));
    }
    else {  // sibling -> node
      siblingLeafNode->MoveLastToFrontOf(leafNode);
      parentNode->SetKeyAt(index, leafNode->KeyAt(0));
    }
  }
  else {
    InternalPage *internalNode = reinterpret_cast<InternalPage*>(node);
    InternalPage *siblingInternalNode = reinterpret_cast<InternalPage*>(neighbor_node);
    if(index == 0) {  // node -> sibling
      siblingInternalNode->MoveFirstToEndOf(internalNode, parentNode->KeyAt(1), buffer_pool_manager_);
      parentNode->SetKeyAt(1, siblingInternalNode->KeyAt(0));
    }
    else {  // sibling -> node
      siblingInternalNode->MoveLastToFrontOf(internalNode, parentNode->KeyAt(index), buffer_pool_manager_);
      parentNode->SetKeyAt(index, internalNode->KeyAt(0));
    }
  }

  buffer_pool_manager_->UnpinPage(parentPage->GetPageId(), true);
}
/*
 * Update root page if necessary
 * NOTE: size of root page can be less than min size and this method is only
 * called within coalesceOrRedistribute() method
 * case 1: when you delete the last element in root page, but root page still
 * has one last child
 * case 2: when you delete the last element in whole b+ tree
 * @return : true means root page should be deleted, false means no deletion
 * happend
 */
INDEX_TEMPLATE_ARGUMENTS
bool BPLUSTREE_TYPE::AdjustRoot(BPlusTreePage *old_root_node) {
  /* case 1: still has one last child */
  if(!old_root_node->IsLeafPage() && old_root_node->GetSize()==1) {
    InternalPage *internalNode = reinterpret_cast<InternalPage*>(old_root_node);
    page_id_t childPageId = internalNode->RemoveAndReturnOnlyChild();

    /* update root */
    root_page_id_ = childPageId;
    UpdateRootPageId(0);
    Page *newRootPage = buffer_pool_manager_->FetchPage(root_page_id_);
    InternalPage *newRootNode = reinterpret_cast<InternalPage*>(newRootPage);
    newRootNode->SetParentPageId(INVALID_PAGE_ID);
    buffer_pool_manager_->UnpinPage(newRootPage->GetPageId(), true);
    return true;
  }

  /* case 2: empty in whole b+ tree */
  if(old_root_node->IsLeafPage() && old_root_node->GetSize()==0) {
    root_page_id_ = INVALID_PAGE_ID;
    UpdateRootPageId(0);
    return true;
  }

  return false;
}
```

## TASK #3 - INDEX ITERATOR

### SRC

+ `src/include/index/index_iterator.h` and `src/index/index_iterator.cpp`

### HINT

+ 实现类似`STL`迭代器，主要用于遍历`B+ Tree`叶子节点

### METHOD

+ `IsEnd()`
+ `operator*()`
+ `operator++()`
+ `operator==()`

### IMPLEMENTATION

```C++
INDEX_TEMPLATE_ARGUMENTS
bool INDEXITERATOR_TYPE::IsEnd() { return leaf_->GetNextPageId()==INVALID_PAGE_ID && index_==leaf_->GetSize(); }

INDEX_TEMPLATE_ARGUMENTS
const MappingType &INDEXITERATOR_TYPE::operator*() { return leaf_->GetItem(index_); }

INDEX_TEMPLATE_ARGUMENTS
INDEXITERATOR_TYPE &INDEXITERATOR_TYPE::operator++() {
  index_++;

  if(index_==leaf_->GetSize() && leaf_->GetNextPageId()!=INVALID_PAGE_ID) {
    auto nextPageId = leaf_->GetNextPageId();
    buffer_pool_manager_->UnpinPage(leaf_->GetPageId(), false);
    Page *nextPage = buffer_pool_manager_->FetchPage(nextPageId);
    leaf_ = reinterpret_cast<LeafPage*>(nextPage);
    index_ = 0;
  }

  return *this;
}

INDEX_TEMPLATE_ARGUMENTS
bool INDEXITERATOR_TYPE::operator==(const IndexIterator &itr) const {
  return leaf_->GetPageId()==itr.leaf_->GetPageId() && index_==itr.index_;
}

INDEX_TEMPLATE_ARGUMENTS
bool INDEXITERATOR_TYPE::operator!=(const IndexIterator &itr) const { return !(*this == itr); }
```

## TASK #4 - CONCURRENT INDEX

### HINT

+ 实现并发操作，无非就是让`GetValue()`、`Insert()`和`Remove()`操作是原子化的，很容易想到互斥量
+ 也可以用`lab`提供的事务的读写锁





