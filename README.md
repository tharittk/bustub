<img src="logo/bustub-whiteborder.svg" alt="BusTub Logo" height="200">

-----------------

[![Build Status](https://github.com/cmu-db/bustub/actions/workflows/cmake.yml/badge.svg)](https://github.com/cmu-db/bustub/actions/workflows/cmake.yml)

BusTub is a relational database management system built at [Carnegie Mellon University](https://db.cs.cmu.edu) for the [Introduction to Database Systems](https://15445.courses.cs.cmu.edu) (15-445/645) course. This system was developed for educational purposes and should not be used in production environments.

================================== Writing in progress ==================================
# Building a SQL Database From Scratch 🛢️

Building a relational database from the ground up was a monumental task, but it turned out also one of the most rewarding challenges for me as a software engineering apprentice. This project "sweated" my system programming. This post documents my journey of building BusTub, a disk-oriented SQL database management system in C++, as part of a project inspired by Carnegie Mellon University's renowned [15-445/645 Database Systems course](https://15445.courses.cs.cmu.edu/fall2024/).

⚠️Note: the course videos and assignments are made public. It is requested that the complete solutions should not be made publicly available. At the moment of exhaustion 😩, I certainly try googled the solution myself. With the benefit of hindsight, **I'm glad it did not exist**  -- and finishing the task feels totally worth it. I hope I use just-enough code to illustrate the point and not giving away too much for future students.  

There are 4 parts in building BusTub:

1.  **Buffer Pool Manager**: An abstraction that "caches" or acts as the bridge between memory and disk.
2.  **B+ Tree and Index**: The workhorse data structure that makes searching for data incredibly fast.
3.  **Query Execution Engine**: The brain that interprets SQL queries and executes them efficiently.
4.  **Concurrency Control**: The traffic cop that allows multiple users to access and modify data simultaneously without chaos.

## 💾 Part 1: The Buffer Pool Manager - The Gateway to Disk
The first component I built was the Buffer Pool Manager (BPM), the database's gateway to the disk. Since disk I/O is orders of magnitude slower than memory access, the BPM acts as a critical caching layer. Its primary responsibility is to keep frequently used database pages in memory, minimizing slow disk reads and writes. This abstraction is powerful: it allows the database to handle datasets much larger than the available RAM, while the rest of the system can request data pages by their ID without needing to know if they are currently in memory or on disk.

A key concept here is the distinction between a *page* and a *frame*. A page is a logical 4KB block of data, which can reside on disk or in memory. A frame is a physical 4KB slot in memory within the buffer pool. The BPM's job is to load logical pages into physical frames. This abstraction is also powerful: it allows the database to handle datasets that are much larger than the available RAM.

**The Challenge: Efficient Caching and Concurrency**

### LRU-K Replacement

When the buffer pool is full and a new page needs to be loaded from disk, a frame must be freed up. I implemented the LRU-K replacement algorithm, a sophisticated policy that improves upon simple LRU. It tracks the history of the last k accesses to each page, making it more robust against periodic scans that could otherwise flush out frequently used pages. An unpinned frame with fewer than k historical accesses is prioritized for eviction, preventing a single long scan from polluting the cache.

The logic for selecting a victim frame is captured in the `Evict()` method

```C++
auto LRUKReplacer::Evict() -> std::optional<frame_id_t> {
  latch_.lock();
  LRUKNode *evicted_frame = nullptr;
  uint32_t max_distance = 0;
  const uint32_t inf = std::numeric_limits<uint32_t>::max();

  for (auto &[frame_id, node] : node_store_) {
    if (!node.IsEvictable()) {
      continue;
    }

    uint32_t distance = (node.GetHistorySize() < k_) ? inf : current_timestamp_ - node.GetKTimeStamp();

    if (distance > max_distance) {
      max_distance = distance;
      evicted_frame = &node;
    } else {
      // evict the earlier timestamp
      bool both_inf = max_distance == inf && distance == inf;
      bool is_earlier = node.GetKTimeStamp() < evicted_frame->GetKTimeStamp();

      if (both_inf && is_earlier) evicted_frame = &node;
    }
  }

  if (evicted_frame != nullptr) {
    // to be re-used for other pages
    evicted_frame->Reset();
    --curr_size_;
  }
  latch_.unlock();

  return evicted_frame == nullptr ? std::nullopt : std::make_optional<frame_id_t>(evicted_frame->GetFrameID());
}
```

### Concurrency & Thread Safety 🔥

In a multi-threaded database, multiple execution threads will request pages simultaneously. A naive implementation with a single global lock would serialize all access and become a major performance bottleneck.

To solve this, my design employs a two-level locking strategy that balances performance and correctness. A coarse-grained mutex (`bpm_latch_`) protects the BPM's global metadata (the `page_table_` and `free_frames_` list). This lock is held only for the brief duration required to find or allocate a frame, minimizing contention.

Once a frame is identified, concurrency control shifts to a fine-grained, per-page read-write lock (`rwlatch_`). This lock is managed by RAII-style ReadPageGuard and WritePageGuard objects. The BPM releases its global latch before the page guard acquires its page-specific lock, allowing multiple threads to operate on different pages concurrently.

The CheckedWritePage function demonstrates this hand-off:

```C++
auto BufferPoolManager::CheckedWritePage(page_id_t page_id, AccessType access_type) -> std::optional<WritePageGuard> {
  // Global scope - whole buffer pool manager
  bpm_latch_->lock();

  // The requested page is not currently in a memory frame.
  if (page_table_.find(page_id) == page_table_.end()) {
    if (free_frames_.empty()) {
      // all frames are pinned
      if (!FreeFrameSlot()) {
        bpm_latch_->unlock();
        return std::nullopt;
      }
    }

    // Read the page from disk to frame i.e., "cache" it and make it discoverable through page table
    frame_id_t fid = ReadFromDiskToFrame(page_id);
    page_table_.insert({page_id, fid});
  }

  // The page is already in a frame. Get its frame_id.
  frame_id_t fid = page_table_.at(page_id);
  auto fheader = frames_[fid];

  // Create a WritePageGuard, an RAII object that manages the page's pin count
  // and its individual read-write latch.
  auto page_guard = WritePageGuard(page_id, fheader, replacer_, bpm_latch_, disk_scheduler_);

  // A page's pin_count tracks how many threads are actively using it.
  // A pinned page (pin_count > 0) cannot be evicted.
  int prev = page_guard.frame_->pin_count_.fetch_add(1);
  if (prev == 0) {
    replacer_->SetEvictable(page_guard.frame_->frame_id_, false);
  }

  // Notify the LRU-K replacer of this new access so it can update its history.
  replacer_->RecordAccess(fid);

  // The BPM's global latch can be released now. The returned page guard
  // holds a specific lock on the page itself, ensuring safe modification.
  bpm_latch_->unlock();
  page_guard.frame_->rwlatch_.lock(); // NOTE: ReadPage will use lock_shared()

  return std::make_optional<WritePageGuard>(std::move(page_guard));
}
```
The RAII PageGuard pattern ensures correctness by automatically managing the page's state. When a guard goes out of scope, its destructor (`Drop()` method) unpins the page and unlocks its latch, making it available for eviction if no other threads are using it. This design prevents common concurrency bugs like deadlocks and data races while maximizing parallelism.
 
```C++
void WritePageGuard::Drop() {
  if (!is_valid_) {
  } else {
    bpm_latch_->lock();
    int prev = frame_->pin_count_.fetch_sub(1);
    if (prev == 1) {
      replacer_->SetEvictable(frame_->frame_id_, true);
    }
    this->replacer_ = nullptr;
    this->disk_scheduler_ = nullptr;
    this->is_valid_ = false;
    bpm_latch_->unlock();
    this->frame_->rwlatch_.unlock();
  }
}
```

```bash
Running main() from gmock_main.cc

[       OK ] LRUKReplacerTest.SampleTest (0 ms)
[       OK ] DiskSchedulerTest.ScheduleWriteReadPageTest (0 ms)
[       OK ] BufferPoolManagerTest.VeryBasicTest (0 ms)
[       OK ] BufferPoolManagerTest.PagePinEasyTest (0 ms)
[       OK ] BufferPoolManagerTest.PagePinMediumTest (1 ms)
[       OK ] BufferPoolManagerTest.PageAccessTest (1006 ms)
[       OK ] BufferPoolManagerTest.ContentionTest (351 ms)
[       OK ] BufferPoolManagerTest.DeadlockTest (1000 ms)
[       OK ] BufferPoolManagerTest.EvictableTest (651 ms)
[  PASSED  ] 9 tests.
```

---

## 🌲 Part 2: The B+ Tree - The Foundation of Fast Queries (Indexing)

With a system for managing pages in memory, the next step is to organize the data on those pages to enable fast, efficient lookups. This is the job of the **B+ Tree**. 

Prof. Andy Pavlo describes it as the "greatest data structure of all-time" -- and I came to agree with that as I'm building it. It's a self-balancing tree data structure that maintains sorted data and allows for efficient insertion, deletion, and point and range queries—all in logarithmic time. In a database, B+ Trees are the primary data structure used for creating indexes, which allow the system to find a specific record by its key without scanning the entire table.

The B+ Tree has a specific structure:
*   **Internal Pages**: These pages act as **signpost**. They store an ordered list of keys and pointers (page IDs) to child pages. An internal page with *m* keys will have *m+1* children, guiding the search down the tree.
*   **Leaf Pages**: These pages are at the bottom-most level of the tree and store the actual index entries: the key and its corresponding value. In our database, this value is a `RID` (Record ID), which points to the exact location of the full tuple in the table heap. All leaf pages are also linked together in a doubly-linked list, which allows for highly efficient range queries.

### Complex Structural Modifications and Concurrency

Implementing a B+ Tree is a significant undertaking with several deep challenges:

1.  **Maintaining Tree Invariants**: The B+ Tree's performance guarantees rely on it being balanced and its nodes being at least half-full (except for the root). This requires complex logic for structural modifications:
    *   **Splitting**: When an insertion causes a page (leaf or internal) to overflow, it must be split into two pages. The median key is promoted up to the parent internal page to update its "signposts." This can cascade all the way up to the root, potentially increasing the tree's height by one.
    *   **Merging & Redistribution**: When a deletion causes a page to become less than half-full, it must be fixed. If an adjacent sibling page has extra keys, we can perform **redistribution**, borrowing a key to rebalance the two pages. If the sibling is also at the minimum size, we must **merge** the two pages into one. This deletion can also cascade up the tree, potentially shrinking its height.

    🔥 These mechanics by itself is hard to write in English. Let alone writing the correct code. I end up using this [textbook](https://www.db-book.com/) which spells out the psuedo-code for B+ Tree. Even in psuedo-code, it takes 3 pages.

2.  **Fine-Grained Concurrency Control**: Multiple threads might try to read and modify the tree at the same time. Locking the entire tree on every operation would make it a major performance bottleneck. The solution is a fine-grained locking protocol called **latch-crabbing** (or lock-coupling). The idea is to hold latches on as few pages as possible. As a thread traverses down the tree from the root, it acquires a latch on a child page *before* releasing the latch on its parent. Once the child is deemed "safe"—meaning it won't need to split or merge in a way that would require modifying the parent—the latch on the parent (and all its ancestors) can be released. This allows multiple threads to operate on different subtrees concurrently, dramatically improving throughput.

### Concurrent Tree Traversal with Latch-Crabbing 🦀

The latch-crabbing logic is central to the B+ Tree's concurrent performance. The snippet below from `GetWriteGuardKeyIn` shows how a thread safely traverses the tree to find the correct leaf node for an insertion or deletion, acquiring and releasing latches as it goes.

```cpp
// From: src/storage/index/b_plus_tree.cpp

INDEX_TEMPLATE_ARGUMENTS
auto BPLUSTREE_TYPE::GetWriteGuardKeyIn(Context &ctx, page_id_t root_page_id, const KeyType &key, OpType op)
    -> WritePageGuard {

  ctx.root_page_id_ = root_page_id;

  page_id_t current_page_id = root_page_id;
  // Start by acquiring a write latch on the root page.
  WritePageGuard page_guard = bpm_->WritePage(current_page_id);
  auto *page = page_guard.AsMut<BPlusTreePage>();

  // If the root is already a leaf, the traversal is complete.
  if (page->IsLeafPage()) {
    return page_guard;
  }

  // Traverse down the tree until we reach a leaf.
  while (!page->IsLeafPage()) {
    auto *internal = reinterpret_cast<InternalPage *>(page);
    // Find the child pointer to follow using binary search.
    int leftmost = BinarySearchInInternal(internal, key);
    current_page_id = internal->ValueAt(leftmost - 1);

    // Acquire a write latch on the child page. THIS IS THE "CRAB" STEP.
    WritePageGuard child_guard = bpm_->WritePage(current_page_id);
    auto *child_page = child_guard.AsMut<BPlusTreePage>();

    // Store the parent's guard in the context to track the path of latches held.
    ctx.write_set_.push_back(std::move(page_guard));

    // Now, check if the child node is "safe" for the current operation.
    bool is_safe = false;
    if (op == OpType::INSERT) {
      // For insertion, a node is safe if it's not full. No split will occur.
      is_safe = child_page->GetSize() < child_page->GetMaxSize() - 1;
    } else { // op == OpType::DELETE
      // For deletion, a node is safe if it has more than the minimum entries. No merge/redistribution needed.
      is_safe = child_page->GetSize() > child_page->GetMinSize();
    }

    // If the child is safe, we can release all latches held on its ancestors.
    // This maximizes concurrency by unlocking the upper parts of the tree.
    if (is_safe) {
      ReleaseLatchInContext(ctx);
    }

    // Move to the child node and continue the traversal.
    page_guard = std::move(child_guard);
    page = child_page;
  }
  return page_guard;
}
```

This function is a beautiful illustration of the latch-crabbing algorithm 🦀. By acquiring a latch on the child before releasing the parent's, we ensure no other thread can modify the link between them during traversal (it walks like a crab so people call it so). By opportunistically releasing latches on "safe" nodes as we descend, we maximize concurrency. 

```bash
Running main() from gmock_main.cc
[       OK ] BPlusTreeTests.BasicInsertTest (0 ms)
[       OK ] BPlusTreeTests.InsertTest1NoIterator (0 ms)
[       OK ] BPlusTreeTests.InsertTest2 (0 ms)
[       OK ] BPlusTreeTests.DeleteTestNoIterator (0 ms)
[       OK ] BPlusTreeTests.SequentialEdgeMixTest (2 ms)
[       OK ] BPlusTreeTests.BasicScaleTest (2060 ms)
[       OK ] BPlusTreeConcurrentTest.InsertTest1 (294 ms)
[       OK ] BPlusTreeConcurrentTest.InsertTest2 (281 ms)
[       OK ] BPlusTreeConcurrentTest.DeleteTest1 (12 ms)
[       OK ] BPlusTreeConcurrentTest.DeleteTest2 (15 ms)
[       OK ] BPlusTreeConcurrentTest.MixTest1 (5870 ms)
[       OK ] BPlusTreeConcurrentTest.MixTest2 (475 ms)
[  PASSED  ] 12 tests.
```
---

## Part 3: The Query Execution Engine
