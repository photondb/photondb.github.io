---
layout: post
title:  "The space reclamation of PhotonDB"
---

# Background

The page store of PhotonDB can be regarded as a log-structured page allocator.

The page store is divided into persistent and in-memory parts on the internals. The memory part consists of a list of write buffers. A write buffer is a continuous memory space, and new delta pages are allocated from the last write buffer. Each delta page has a unique logical address, which is increased in the allocation order. Other pages can access the delta page through this logical address. A delta page is active if it can be accessed from the root node. Once a page is updated, the relevant delta pages will no longer be accessed, so they will be deallocated and returned to the page store.

The active delta pages of a write buffer will be sealed and persisted to a new page file if the write buffer does not satisfy allocation requests. In addition to recording delta pages, the page file also records some metadata, including the mapping of delta pages to page ids, the addresses of deallocated delta pages that belong to other page files, and the offset of each delta page in the write buffer. Each page file maintains a data structure: FileMeta, which records the mapping of the logical address to the persisted offset in the page file.

Each write buffer has a unique and increasing ID, which is part of the logical address of a delta page: the logical address consists of the buffer ID and the offset of the delta page in the buffer:

```
logical address = (buffer id << 32) | offset
```

The ID of the page file is equal to the write buffer where the page file is generated, so for any logical address, you can directly locate the write buffer or page file and find the delta page (for the page file, you also need to find the file offset by querying FileMeta).

As mentioned earlier, the logical addresses of the deallocated delta pages are recorded in the page files. Although the data corresponding to the addresses will no longer be accessed, the disk space occupied by the delta pages is still preserved. We call this part of the space empty pages. To ensure that there is enough space for future write requests, the empty pages of these page files need to be free. Finding suitable page files and releasing the empty pages is the responsibility of space reclamation.

# Reclaiming framework

There are three questions to be answered about implementing space reclaiming: When is it triggered? What is the goal of optimization? What are the details of processing a page file?

These questions outline the contours of space reclamation.

1. trigger timing
2. page file selection strategy
3. Page file reclamation details

When certain metrics reach the trigger condition, PhotonDB uses the selection strategy to select candidate page files and process the page files, finally freeing up the free space. Later on, we will introduce PhotonDB's solutions to these three problems in turn.

## Trigger

The first discussion is about the trigger timing of space reclaiming. PhotonDB focuses on two metrics.

1. space usage
2. space amplification

When the space usage exceeds the high water mark or the space amplification exceeds the upper limit, PhotonDB will trigger the space reclaiming until the relevant metrics drop below the threshold. The purpose of setting the watermark of space usage is to ensure the proportion of remaining space; the purpose of setting the upper limit of space amplification is to spread the overall reclaiming cost to the whole running period of the program. Of course, free space can only be freed if there are deallocated delta pages; therefore, the space usage metric can only take effect if there is reclaimable space.

## Efficient strategy

Space reclamation takes up IO resources, it needs to relocate the active delta pages in the candidate page files. The cost of this process is related to the number of deallocated delta pages. The more deallocated delta pages, the smaller the IO cost of relocation.

The goal of the candidate file selection strategy is to find the most suitable page files for reclaiming so that the total IO overhead is minimized.

### Cost minimization

To find such a strategy, let's assume that at logical time $t_n$, the cost of a page file $i$ is $C_i$ and the decline rate of declaiming cost is $-\frac{dc_i(t_0)}{dt}$. At logical time $t_0$, if the reclaiming cost is $C_0$, then for any future time of $t$, the reclaiming cost of the page file satisfies:

$C_i(t) \approx C_i(t_0) - (-\frac{dc_i(t_0)}{dt}) (t - t_0)$

Assuming there are $k$ page files arranged in any order, reclaiming one at a time, the total cost is:

$Cost = \sum_{i=1}^{k} c_i(t_0) - \sum_{i=1}^{k}-\frac{dc_i(t_0)}{dt}(t_i-t_0)$

Now the problem is finding an order of page files that minimizes the total cost. $Cost$ is minimized when the second sum on the right-hand side is maximized. Given two sets of positive numbers X and Y, with $||X|| = ||Y||$, $\sum x y$ is maximized when $X = {x_i}$ and $Y = {y_i}$ are both ordered in the same way, i.e., $x_i ≥ x_j iff y_i ≥ y_j$. Since the sequence of $t_i-t_0$ is sorted incrementally, to maximize the second sum, the decline rates should also be sorted in increasing order. Therefore, to minimize the total cost, the page files with the smallest decline rate are prioritized and the page files with the largest decline rate are processed last.

We call the selection strategy given by the above formula the Min Decline Rate strategy. It is intuitive: if the cost can decline significantly over time in the future, then it is worthwhile to wait for some time before processing.

### Decline rate

With the theoretical guidance, the next step is to calculate the decline rate for each page file. Assuming that the proportion of empty pages in a page file is $E$, then reclaiming the space of a page file requires reclaiming $1/E$ page files with empty pages. The external write is $1/E(1-E)$. Then, the cost of writing a page file is:

$Cost = \frac{1}{E} reads + \frac{1}{E} (1-E) writes + 1 = \frac{2}{E}$

Further, the IO cost decline rate is:

$\frac{d(Cost)}{dt} = (\frac{-2}{E^2})(\frac{dF}{du}) \approx \frac{−2(1 − E)}{E^2}f\Delta E$

Where $f$ is the update frequency of delta pages and $\delta E$ is the rate of change of $E$ with each update. The update frequency of the file is $f$ multiplied by the number of active pages.

The update frequency of each page can be estimated as follows:

$f = \frac{2}{t_{now}-t_{up2}}$

Where $t_{up2}$ represents the logical time when a page is updated for the penultimate time.

The Min Decline Rate strategy comes from the paper: Efficiently Reclaiming Space in a Log Structured Store. If you are interested in the derivation process, you can refer to the original paper.

PhotonDB uses the Min Decline Rate strategy as the candidate page files selection strategy. The above formula is used to calculate and rank the decline rate of each page file each time space reclaiming is performed. In addition to the cost calculation, the original paper also points out that the reclaiming process can use the update frequency to classify delta pages to further reduce the reclaiming cost.

## Process candidate files

When processing candidate page files, it should be ensured that the active delta pages are still accessible after processing is completed. Since in-place updates are not considered, reclaiming a page file requires the implementation to copy the active delta pages to other locations.

The straightforward way is to copy the delta page into a newly created space while updating the reference to the delta page to point it to the copied location. This process, called relocation, has one obvious flaw: it changes the logical address of the delta page. Each active delta page can be accessed from the root node, and the reference to it may exist in the page table or in the predecessor node on the same delta chain. For the latter, changing the logical address means traversing the entire delta chain to find the corresponding predecessor node to replace. For immutable data structures, replacement means introducing a new delta page, which is responsible for mapping the original logical address to the new logical address.

To avoid introducing additional complexity, version 0.2 of PhotonDB uses a page rewriting mechanism to avoid the above problems.

### Page rewriting issue

The page rewriting of PhotonDB is similar to consolidation operations. So they can reuse some logic. Consolidation merges the delta chain and replaces the delta chain with the resulting delta page.

Page rewriting and consolidation are slightly different, and divided into two aspects:

1. Page rewriting still has the effect of relocation, even if the delta chain length is 1, it needs to generate a new delta page and replace it.
2. Page rewriting may generate a delta chain instead of a delta page. For example, a split delta cannot be merged into a new delta page until it is applied to the parent node.

The downside of the page rewriting is also obvious. The process of merging the delta chain has a lot of IO fan-in and fan-out. Another problem with page rewriting is that it does not track the update frequency of these resulting delta pages. Unlike the practice in the paper, PhotonDB only tracks the update frequency of the page file, not the update frequency of the delta page, for performance reasons. The delta pages generated by these rewrites are treated as new writes and mixed with the delta pages written by the user, resulting in distorted update frequency.

## Solution

It is possible to reclaim a page file without modifying the logical address of delta pages, simply by introducing a  global translation layer that maps the page file id from the logical address to the physical address `(file, offset)`.

However, as more and more delta pages are deallocated, new files will become smaller and more fragmented, requiring greater overhead to maintain metadata for these small files, while io prefetching and batch processing will become less efficient. To this end, we introduce a new file format: mapping file. Mapping files package the active delta pages in multiple page files into one file and record the mapping relationship between logical addresses and physical addresses.

In addition to reducing fragmentation and avoiding the IO amplification introduced by page rewriting, the mapping file also provides the previously mentioned ability to classify delta pages according to the update frequency. Putting delta pages with similar update frequencies together can achieve the effect of hot and cold separation. From the experiment, the introduction of the mapping file makes the 0.3 version of PhotonDB reduce the write amplification by~ 5 times and~ 2.5 times under zipfan and uniform workload, respectively, compared with the previous version.

