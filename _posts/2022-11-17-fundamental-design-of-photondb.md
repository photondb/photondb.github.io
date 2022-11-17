---
layout: post
title: "The Fundamental Design of PhotonDB"
---

Today, we are happy to announce [PhotonDB 0.0.2][photondb-0.0.2]!

While 0.0.1 only implements an in-memory latch-free B-tree, 0.0.2 comes with a persistent log-structured store and more. 0.0.2 is an important milestone because it is a proof-of-concept of our initial thoughts about PhotonDB.

Some of you may be interested in how PhotonDB works. In this post, we will describe the fundamental design of PhotonDB. It is important to note that PhotonDB is still under active development, and things are subject to change. So we are not going to give too many implementation details here. Instead, we will focus on interesting ideas and techniques that might be helpful to other similar projects.

## Architecture

![Figure 1](/images/2022-11-17-figure-1.drawio.svg)

Let's first introduce the overall architecture of PhotonDB. The architecture consists of three major modules:

- A latch-free B-tree
- A log-structured page store
- An environment abstraction of runtimes and platforms

The B-tree consists of many inner and leaf pages persisting in the page store. It interacts with the page store using a transactional interface that supports atomic operations across multiple pages.

The page store organizes pages into page files. It also runs background jobs to re-organize the files from time to time. To make PhotonDB runtime-agnostic, the page store interacts with the underlying runtime or platform via an environment abstraction.

## Latch-free B-tree

The latch-free B-tree implementation is inspired by the Bw-tree, with some differences and improvements.

The latch-free B-tree consists of logical pages. Each logical page has a unique id and stores data in a delta chain. In addition, the tree maintains a mapping table to translate a page id to the address of the delta chain. To access an entry, we first locate the corresponding leaf page starting from the root. Then, for reads, we merge the delta chain to get the desired data; for writes, we simply prepend a delta to the chain. This data structure might sound straightforward, but there are some challenges due to its latch-free nature.

The first challenge is ensuring that the page we are visiting is the one we expect. Let's explain the problem with an example in Figure 2. Two threads are concurrently accessing the tree. Thread 1 wants to write key 7 to the tree. It starts locating the leaf page for key 7 on page 1. Page 1 is an inner page, it points thread 1 to a child with page id 2. However, before thread 1 reaches page 2, thread 2 splits it and moves half of its data to page 3. Then when thread 1 visits the new page 2, it doesn't cover key 7 anymore. Ooops, if thread 1 doesn't know that and continues writing key 7 to the new page 2, it will screw up the page.

![Figure 2](/images/2022-11-17-figure-2.drawio.svg)

Surprisingly, [the Bw-tree paper][Bw-tree] doesn't mention the problem explicitly. Though some other related papers indicate that the Bw-tree solves the problem by embedding boundary keys on each page. Whenever a page is visited, its boundary keys are checked to ensure that it is the right page for the key. This solution works, but it comes with two non-trivial costs. The first cost is storing the boundary keys on each page. The second cost is checking the boundary keys on every page visit.

In PhotonDB, we have a more lightweight solution to reduce the costs. The observation is that the range of a page only changes when it splits or merges. So instead of checking if a page covers a key, we check if the page's range varies after we have visited its parent. We achieve this by adding an epoch number to each page. Whenever a page splits or merges, its epoch number also increases. In addition, we store every page's epoch number along with its page id in its parent. Whenever we visit a page, we compare the page's epoch number with the one from its parent to see if the range varies. Figure 3 shows an example of how the epoch number helps to detect conflicts. The current implementation uses a 48-bit integer to represent an epoch number, which is cheap to store and check compared to the boundary keys.

![Figure 3](/images/2022-11-17-figure-3.drawio.svg)

The second challenge is flushing multiple pages to the disk atomically. Atomicity is necessary for structure modification operations. For example, a page split first inserts a new page with the right half of the data and then updates the original page to shrink the range. Without atomicity, we may crash after inserting the new page but before updating the original page. In this case, the page inserted before the crash will become an orphan. We can't reach and clean orphan pages unless we scan all pages to find it out.

In PhotonDB, we implement a lightweight transactional interface on the page store. Every write to the tree is done within a transaction to avoid having an inconsistent state on restart. The key to atomicity is to avoid flushing the half-done transaction state to the disk. To achieve this, the page store maintains a list of write buffers to cache pending writes. When the size of a buffer reaches a threshold, the buffer is flushed to the disk. All pages in the same write buffer are persisted atomically. To prevent a write buffer from flushing, each transaction holds a reference of the current write buffer until the transaction finishes.

## Log-structured page store

The page store can be regarded as a sequential log of pages. All pages are allocated from the page store with increasing page addresses. Given a page address, the store can locate the physical position of the page on the disk.

![Figure 4](/images/2022-11-17-figure-4.drawio.svg)

Internally, the page store is divided into a persistent part and an in-memory part.

The persistent part consists of a set of page files. Each file has a unique and increasing id. Each file stores a set of pages and some metadata for crash recovery and space reclamation. We will talk more about recovery later but leave the space reclamation for future posts.

The in-memory part consists of a list of write buffers. Besides providing atomicity as described earlier, the write buffers also serve as a latch-free, log-structured allocator for new pages. This allocator allows fast allocation with no fragmentation. Each write buffer is also identified by an increasing id, which will become the file id when the buffer is flushed. Given a page address, we can extract the id of the file or the buffer containing the page, and the page's offset in the file or the buffer.

One challenge of the page store is allowing concurrent access to the page files and write buffers, while having background jobs to flush write buffers and clean up obsoleted page files from time to time. The Bw-tree uses an epoch-based reclamation mechanism to address the problem. In Rust, we have a [`crossbeam-epoch`][crossbeam-epoch] crate for the job. However, `crossbeam-epoch` is not designed for async Rust. If a thread runs multiple async tasks, each of which pins the local epoch in turns, the local epoch may never be advanced.

Fortunately, we have found another way out using a concurrency control mechanism similar to LevelDB/RocksDB. We organize page files and write buffers into versions. Every access to the tree takes a reference of the latest version first. The version guarantees that all accessible page files and write buffers remain valid until there is no reference to it anymore.

Another challenge of the page store is crash recovery. We use a manifest file to record all changes to the persistent state. So recovering the page files is straightforward. The hard part is recovering the page table, which is latch-free and constantly updated by multiple threads. We must ensure that the page files and the page table are consistent after recovery.

[LLAMA][LLAMA] uses a simple tactic to checkpoint the complete page table from time to time. Each checkpoint also records the offset of the sequential log when the checkpoint starts. On restart, the store recovers to a consistent state up to the offset in the last checkpoint. However, we think it is not economic writing out the whole page table when only some of the entries changes between checkpoints.

In PhotonDB, we figure out a way to persist updated entries in the page table incrementally. The key lies in that each update to the page table is also associated with a new page written to the write buffer. So if we store the corresponding page id with the new page, then the mapping from the page id to the new page address will be flushed along with the buffer. In addition, we aggregate all mappings in the buffer to a metadata block in the page file. This way, on a restart, we only need to read a small block from each file to construct the complete page table.

## Environment

Being runtime-agnostic has both pros and cons in Rust. The pros is that we can choose whatever runtime we want. The cons is that once we choose a runtime, we may be tied to it and its ecosystem. This is because a lot of async crates depend on a specific runtime to work. If we are not using the same runtime as these crates, we have to say goodbye to them.

Since PhotonDB is also an async crate, we have to decide a runtime to depend on too. The simple answer is to choose one of the most popular runtimes. However, we will be upset if people can't use PhotonDB simply because they prefer another runtime.

So, in order to make PhotonDB runtime-agnostic, we introduce an environment abstraction. We build a trait called `Env`, which abstracts all interactions between PhotonDB and a specific runtime or platform. Most of the interactions are about file IO and task management used by the page store. The raw APIs of PhotonDB depend on a generic parameter that implements `Env`. Since `Env` is a public trait, users can implement the trait for whatever runtime they want. If everything goes well, users can plug their `Env` implementation in the raw APIs to run PhotonDB with their runtime.

PhotonDB currently provides two built-in `Env` implementations: `Std` and `Photon`. The `Std` implementation is based on the standard library in Rust, and the `Photon` implementation is based on our new async runtime [`PhotonIO`][photonio-0.0.5]. These two environments enable PhotonDB to provide both synchronous and asynchronous APIs. For more details about the APIs, please check the document of [PhotonDB 0.0.2][photondb-0.0.2].

## Conclusion

While the current version may not be stable or performant, the fundamental design of PhotonDB is already working. We have landed and improved the ideas from different papers and projects along with the development. We are proud of what we have done and will keep evolving [the project][photondb].

[photondb]: https://github.com/photondb/photondb/
[photondb-0.0.2]: https://docs.rs/photondb/latest/photondb/
[photonio-0.0.5]: https://docs.rs/photonio/latest/photonio/
[crossbeam-epoch]: https://crates.io/crates/crossbeam-epoch
[Bw-tree]: https://www.microsoft.com/en-us/research/publication/the-bw-tree-a-b-tree-for-new-hardware/
[LLAMA]: https://www.microsoft.com/en-us/research/publication/llama-a-cachestorage-subsystem-for-modern-hardware/