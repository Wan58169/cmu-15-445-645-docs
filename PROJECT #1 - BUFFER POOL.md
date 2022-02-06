# PROJECT #1 - BUFFER POOL

## OVERVIEW

+ 实现一个buffer pool manager（简称`bpm`），用来管理`memory`
+ `bpm`主要的工作：
  + 支持**创建**、**淘汰**和**查找**`page`的功能
+ 简而言之，`bpm`实现的是**系统软件**中最核心的内存管理功能
  + `OS`中的虚拟内存

## TIPS

1. 每张`page`的`id`都是*unique*
2. 所有操作的实现都是`thread-safe`级别的

## TASK #1 - LRU REPLACEMENT POLICY

### SRC

+ `LRUReplacer` in `src/include/buffer/lru_replacer.h`和`src/buffer/lru_replacer.cpp`
+ `Replacer` in `src/include/buffer/replacer.h`

### HINT

+ `LRUReplacer`的大小和`bpm`的大小是一样的，实际上就是`LRUReplacer`管理着`pool`中那些可以被放回到`disk`中的`pages`
+ 一开始， `LRUReplacer`中并没有`frame`，只有当`Unpin(page)`之后，该`page`才会被加入到`LRUReplacer`的备选名单中
+ `page`和`frame`不是一一对应的关系，`page`是逻辑层级的编号，而`frame`是物理层级的编号，之间的关系：`frame_0`在A时刻存放着`page_1`，也可以在B时刻存放`page_3`，换言之，`frame`就是`page`的物理载体（铁打的`frame`，流水的`page`）

### METHOD

+ `Victim(T*)`：从备选名单中挑选出一张可以淘汰的`page`
+ `Pin(T)`：在`Pin(page)`之后，`LRUReplacer`从备选名单中移除该`page`
+ `Unpin(T)`：在`Unpin(page)`之后，`LRUReplacer`将该`page`加入备选名单
+ `Size()`：返回`LRUReplacer`中的`frame`数量

### IMPLEMENTATION

+ **data structures**

  1. `std::list<frame_id_t> list_`双向循环链表，`unpinned page`直接插入到队首，当需要`Victim`的时候直接删除队尾元素，这样的话**插入**和**删除**都是`O(1)`时间
  2. `std::unordered_map<frame_id_t, std::list<frame_id_t>::iterator> map_`哈希表，存放着所有`frame`在`list_`中的位置，保证能够在`O(1)`时间内查找到该`frame`

+ **member data**

  ```c++
  private:
    // TODO(student): implement me!
    std::recursive_mutex mtx_;
    std::list<frame_id_t> list_;
    std::unordered_map<frame_id_t, std::list<frame_id_t>::iterator> map_;
    size_t cap_;
  ```

  选用`std::recursive_mutex`的理由是递归锁虽然会牺牲一点效率的，但是编写`thread-safe`级别代码不易出错（`dead lock`）

+ **method**

  ```c++
  bool LRUReplacer::Victim(frame_id_t *frame_id) {
    std::lock_guard<std::recursive_mutex> lk(mtx_);
    if(Size() == 0) {
      return false;
    }
    frame_id_t tmpFrameId = list_.back();
    list_.pop_back();
    auto ite = map_.find(tmpFrameId);
    map_.erase(ite);
    *frame_id = tmpFrameId;
    return true;
  }
  ```

  1. 如果`LRUReplacer`的备选名单中没有`frame`，则直接返回`false`；
  2. 取出`list_`的队尾元素，然后在`map_`也做相应的删除操作，也就从备选名单中移除；
  3. 最后，返回`true`并将该`frame`返回给上层

  ```c++
  void LRUReplacer::Pin(frame_id_t frame_id) {
    std::lock_guard<std::recursive_mutex> lk(mtx_);
    auto ite = map_.find(frame_id);
    /* 如果该frame未曾出现过 */
    if(ite == map_.end()) {
      return;
    }
    list_.erase(ite->second);
    map_.erase(ite);
  }
  ```

  1. 先在`map_`中查找是否存在该`frame`；
  2. 如果该`frame`未曾出现过，则直接结束；
  3. 反之，从`list_`和`map_`中移除，也就是从备选名单中移除

  ```c++
  void LRUReplacer::Unpin(frame_id_t frame_id) {
    std::lock_guard<std::recursive_mutex> lk(mtx_);
    auto ite = map_.find(frame_id);
    /* 如果该frame曾经出现过 */
    if(ite != map_.end()) {
      return;
    }
    else {
      list_.push_front(frame_id);
      auto tmpKV = std::pair<frame_id_t, std::list<frame_id_t>::iterator>(frame_id, list_.begin());
      map_.insert(tmpKV);
    }
  }
  ```

  1. 先在`map_`中查找是否存在该`frame`；
  2. 如果该`frame`曾出现过，则直接结束；
  3. 加入备选名单，也就是把该`frame`压入`list_`和`map_`中

  ```c++
  size_t LRUReplacer::Size() {
    std::lock_guard<std::recursive_mutex> lk(mtx_);
    size_t size = list_.size();
    return size;
  }
  ```

  返回`list_`中`frame`个数

### SUMMARY

1. Leetcode上的`146. LRU缓存`与之类似，可以练手

## TASK #2 - BUFFER POOL MANAGER

### SRC

+ `BufferPoolManager` in `src/include/buffer/buffer_pool_manager.h`和`src/include/buffer/buffer_pool_manager.cpp`

### HINT

+ `bpm`是一个中间角色，主要是与`disk`和`memory`打交道，其具体行为包括：
1. 能够从`disk`中调出`page`
  
2. 也能够将`page`放回`disk`中
  
+ 如果一张`page`里面没有实质内容，则它的`page_id`一定要设置成`INVALID_PAGE_ID`

+ `page`需要知道此时此刻到底还有多少个`client`在引用自己，如果`pin_count`>0的话，那么`bpm`是不能对其进行`Unpin`等操作的

### METHOD

+ `FetchPageImpl(page_id)`：从`buffer pool`中挑出`client`需要的`page`
+ `NewPageImpl(page_id)`：在`buffer pool`中新建一张`page`
+ `UnpinPageImpl(page_id, is_dirty)`：将`buffer pool`中目标`page`加入`LRUReplacer`的备选名单
+ `FlushPageImpl(page_id)`：将目标`page`内容写入`disk`
+ `DeletePageImpl(page_id)`：从`buffer pool`中删除目标`page`
+ `FlushAllPagesImpl()`：将`buffer pool`中所有`page`写入`disk`

### IMPLEMENTATION

+ **data structures**

  1. `std::list<frame_id_t> free_list_`双向链表用来存放一些空闲的`frame`
  2. `std::unordered_map<page_id_t, frame_id_t> page_table_`哈希表用来存放`page`和`frame`的位置关系，以便在`pages_`中快速定位`page`

  <img src="pics/Buffer Pool Manager.png" alt="Buffer Pool Manager" style="zoom:100%;" />

+ **member data**

  ```c++
   /** Number of pages in the buffer pool. */
    size_t pool_size_;
    /** Array of buffer pool pages. */
    Page *pages_;
    /** Pointer to the disk manager. */
    DiskManager *disk_manager_ __attribute__((__unused__));
    /** Pointer to the log manager. */
    LogManager *log_manager_ __attribute__((__unused__));
    /** Page table for keeping track of buffer pool pages. */
    std::unordered_map<page_id_t, frame_id_t> page_table_;
    /** Replacer to find unpinned pages for replacement. */
    Replacer *replacer_;
    /** List of free pages. */
    std::list<frame_id_t> free_list_;
    /** This latch protects shared data structures. We recommend updating this comment to describe what it protects. */
    std::recursive_mutex latch_;
  ```

+ **method**

  ```c++
  /**
   * Fetch the requested page from the buffer pool.
   * @param page_id id of page to be fetched
   * @return the requested page
   */
  Page *BufferPoolManager::FetchPageImpl(page_id_t page_id) {
    // 1.     Search the page table for the requested page (P).
    // 1.1    If P exists, pin it and return it immediately.
    // 1.2    If P does not exist, find a replacement page (R) from either the free list or the replacer.
    //        Note that pages are always found from the free list first.
    // 2.     If R is dirty, write it back to the disk.
    // 3.     Delete R from the page table and insert P.
    // 4.     Update P's metadata, read in the page content from disk, and then return a pointer to P.
  
    std::lock_guard<std::recursive_mutex> lk(latch_);
    frame_id_t frameId;
  
    // step0
    size_t pinnedCnt = 0;
    for(size_t i=0; i<pool_size_; i++) {
      if(pages_[i].pin_count_ >= 1) {
        pinnedCnt++;
      }
    }
    if(pinnedCnt == pool_size_) { return nullptr; }
  
    // step1
    auto ite = page_table_.find(page_id);
    // step1.1
    if(ite != page_table_.end()) {
      frameId = ite->second;
      replacer_->Pin(frameId);
      Page *p = pages_+frameId;
      return p;
    }
    // step1.2
    bool existFrame = true;
    if(free_list_.size() == 0) {
      existFrame = replacer_->Victim(&frameId);
    }
    else {
      frameId = free_list_.front();
      free_list_.pop_front();
    }
  
    // step2
    if(!existFrame) return nullptr;
  
    Page *r = pages_+frameId;
    if(r->is_dirty_) {
      FlushPageImpl(pages_[frameId].page_id_);
    }
  
    // step3
    auto tmpKV = std::pair<page_id_t, frame_id_t>(page_id, frameId);
    page_table_.insert(tmpKV);
  
    // step4
    char data[PAGE_SIZE];
    disk_manager_->ReadPage(page_id, data);
    pages_[frameId].page_id_ = page_id;
    pages_[frameId].pin_count_ = 1;
    pages_[frameId].is_dirty_ = false;
    strcpy(pages_[frameId].data_, data);
    Page *p = pages_+frameId;
    return p;
  }
  ```

  1. 如果`pages_`中的所有`page`都被引用，则直接返回`nullptr`
  2. 如果该`page`存在于`pages_`中，则直接返回之；反之从`disk`中调取`page`
  3. 先选取可以`Victim`的`frame`，然后进行放回`disk`操作，为`fetched page`腾出空间
  4. 然后再将`fetched page`导入`memory`中，最后返回之

  ```c++
  /**
   * Creates a new page in the buffer pool.
   * @param[out] page_id id of created page
   * @return nullptr if no new pages could be created, otherwise pointer to new page
   */
  Page *BufferPoolManager::NewPageImpl(page_id_t *page_id) {
    // 0.   Make sure you call AllocatePage!
    // 1.   If all the pages in the buffer pool are pinned, return nullptr.
    // 2.   Pick a victim page P from either the free list or the replacer. Always pick from the free list first.
    // 3.   Update P's metadata, zero out memory and add P to the page table.
    // 4.   Set the page ID output parameter. Return a pointer to P.
  
    std::lock_guard<std::recursive_mutex> lk(latch_);
  
    // step1
    size_t pinnedCnt = 0;
    for(size_t i=0; i<pool_size_; i++) {
      if(pages_[i].pin_count_ >= 1) {
        pinnedCnt++;
      }
    }
    if(pinnedCnt == pool_size_) { return nullptr; }
  
    // step2
    frame_id_t frameId;
    if(free_list_.size() == 0) {
      replacer_->Victim(&frameId);
      FlushPageImpl(pages_[frameId].page_id_);
    }
    else {
      frameId = free_list_.front();
      free_list_.pop_front();
    }
  
    // step3
    *page_id = disk_manager_->AllocatePage();
    pages_[frameId].page_id_ = *page_id;
    pages_[frameId].pin_count_ = 1;
    pages_[frameId].is_dirty_ = false;
    auto tmpKV = std::pair<page_id_t, frame_id_t>(pages_[frameId].page_id_, frameId);
    page_table_.insert(tmpKV);
  
    // step4
    Page *p = pages_+frameId;
    return p;
  }
  ```

  1. 如果`pages_`中的所有`page`都被引用，则直接返回`nullptr`
  2. 先看看`free_list_`中有没有空闲`frame`，如果没有则`Victim`一个`frame`，为`new page`腾出空间

  ```c++
  /**
   * Unpin the target page from the buffer pool.
   * @param page_id id of page to be unpinned
   * @param is_dirty true if the page should be marked as dirty, false otherwise
   * @return false if the page pin count is <= 0 before this call, true otherwise
   */
  bool BufferPoolManager::UnpinPageImpl(page_id_t page_id, bool is_dirty) {
    std::lock_guard<std::recursive_mutex> lk(latch_);
    /* 在page_table中定位到需要Unpin的page */
    auto ite = page_table_.find(page_id);
    frame_id_t frameId = ite->second;
  
    if(is_dirty) {
      pages_[frameId].is_dirty_ = true;
    }
  
    replacer_->Unpin(frameId);
  
    if(pages_[frameId].pin_count_ <= 0) {
      return false;
    }
    else {
      pages_[frameId].pin_count_ = 0;
      return true;
    }
  }
  ```

  该方法主要还是调用`LRUReplacer`的`Unpin`方法

  ```c++
  /**
   * Deletes a page from the buffer pool.
   * @param page_id id of page to be deleted
   * @return false if the page exists but could not be deleted, true if the page didn't exist or deletion succeeded
   */
  bool BufferPoolManager::DeletePageImpl(page_id_t page_id) {
    // 0.   Make sure you call DeallocatePage!
    // 1.   Search the page table for the requested page (P).
    // 2.   If P does not exist, return true.
    // 3.   If P exists, but has a non-zero pin-count, return false. Someone is using the page.
    // 4.   Otherwise, P can be deleted. Remove P from the page table, reset its metadata and return it to the free list.
  
    std::lock_guard<std::recursive_mutex> lk(latch_);
  
    disk_manager_->DeallocatePage(page_id);
  
    // step1
    auto ite = page_table_.find(page_id);
  
    // step2
    if(ite == page_table_.end()) {
      return true;
    }
  
    // step3
    frame_id_t frameId = ite->second;
    if(pages_[frameId].pin_count_ != 0) {
      return false;
    }
  
    // step4
    page_table_.erase(ite);
    pages_[frameId].page_id_ = INVALID_PAGE_ID;
    pages_[frameId].pin_count_ = 0;
    pages_[frameId].is_dirty_ = false;
    free_list_.push_back(frameId);
    return true;
  }
  ```

  该方法和`NewPageImpl`思路相同，还有`Flush...`方法较为简单，不作赘述

### SUMMARY

1. 需要好好阅读`src/test/buffer/buffer_pool_manager_test.cpp`，才能深刻理解`bpm`的种种行为







