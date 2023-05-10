---
title: "Watch out for page faults"
date: 2023-05-10
---

# Intro

I am doing some work in the hope of being able to provide a real-time searchable index with very low compute resources.

Low disk usage / high compression can be achieved through [MOF](https://www.biorxiv.org/content/10.1101/2023.04.15.536996v2), but I am afraid is not great for real-time search because decompression itself takes time. An argument is to use better ways to compress/decompress, ways that are programmatic and we can decompress the relevant rows of the index on the fly, e.g. using a [Roaring Bitmap](https://github.com/RoaringBitmap/CRoaring). I haven't explored this much, but in essence decompression is CPU-bound and CPU is a very limited resource, so I am not sure this would give you real-time search with low compute resources.

I am still willing to explore a bit more with [COBS](https://github.com/iqbal-lab-org/cobs) to provide such resource. COBS provide a compact index, instead of compressed one. The compressed indexes, like MOF, make the workload to be CPU bound, and CPU time is expensive. Compact index, like COBS, make the workload to be disk bound, because the index is much larger, but is mmapped. The access pattern is random reads through the index due to hashing. COBS does not using a locality preserving hash like [LPHash](https://github.com/jermp/lphash), so similar kmers won't hash to close positions. For such random access patterns, some already available [very fast SSDs featuring 1.4 million random read IOPS](https://www.tomshardware.com/news/sk-hynix-pcie-4-ssd-record-breaking-random-speeds) might be required.

# A small test

I ran a small test in our EBI cluster using COBS and a subsampled index of [Grace's 661k dataset](https://journals.plos.org/plosbiology/article?id=10.1371/journal.pbio.3001421). This dataset was subsampled to the high-quality genomes (640k genomes), and the indexed kmers were also systematically subsampled. A COBS index was built and I searched a gene through it, and these are the relevant runtime results:
```
	User time (seconds): 3.89
	System time (seconds): 19.17
	Percent of CPU this job got: 2%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 14:39.19
	Maximum resident set size (kbytes): 2388024
	Major (requiring I/O) page faults: 485865
	Minor (reclaiming a frame) page faults: 68008
	Voluntary context switches: 486036
	Involuntary context switches: 17
	File system inputs: 68064
	Page size (bytes): 4096
```

This query took almost 15 mins, with 98% of the time waiting for IO. As COBS mmaps the index, it will incur into major page faults as it requests a page that is not in RAM yet. [Major Page faults seems to be the worst thing that can happen to the performance of a system](https://www.learnsteps.com/difference-between-minor-page-faults-vs-major-page-faults/). Reducing page faults is thus key for COBS performance. For some reason (?), COBS uses `sqrt(documents)` as the "page" size for the index (this basically means the size of a kmer row (a presence-absence row) in the compact index):
```
  -p, --page-size            the page size of the compact the index, default: 
                             sqrt(#documents)
```
This means that for our 640k dataset, page size will be `sqrt(640k)=800`. A kmer row of size 800 is represented as a 100-byte array in disk/RAM, and every kmer results in this small array being requested from disk to RAM. Page sizes are, however, 4096 bytes in size, so we could definitely use a much larger page size. Besides, COBS access pattern means that to search something in the 640k, we need to access `640000/800=800` pages.

To minimise the number of page access, which can cause major page faults, and is the issue with the highest impact on performance, let's increase COBS page size to 4096 bytes, 32768 bits. We rebuilt the COBS index using parameter `--page-size 32768` and rerun this query. It was much faster, under 2 mins:

```
	User time (seconds): 1.90
	System time (seconds): 1.87
	Percent of CPU this job got: 3%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 1:52.33
	Maximum resident set size (kbytes): 827628
	Major (requiring I/O) page faults: 75179
	Minor (reclaiming a frame) page faults: 44362
	Voluntary context switches: 75408
	Involuntary context switches: 14
	File system inputs: 73504
	Page size (bytes): 4096

```

It is correlated with us having just 75k major page faults, instead of 485k as before. Also note that every major page fault implies in a voluntary context switch, which incurs heavy slow downs...

Using a higher page size incurred in a much higher COBS index size: from 87GB (page size 800) to 195GB (page size 32768), but I think is worth it, disk space is not that expensive.

# TODO

* This was done in the EBI cluster, far from an isolated environment. Need to rebenchmark in an isolated environment. The issue is that all my machines use encrypted disks, so not great for this type of benchmark...
* This has also to be tested using a fast NVMe SSD. Without this type of hardware, I don't think COBS can get close to provide real-time searches;
* Review and correct typos;


# Conclusion

I am looking at providing real-time search for large microbial datasets in a frugal environment (think of the machines available [here](https://www.ovhcloud.com/en-gb/vps/compare/)). This might be possible using COBS with a page size of 4kb. Small number of cores are needed, as we've seen in the two test examples before, just ~3% of the runtime was using the CPU, the rest was waiting IO. This amount would surely increase with a fast NVMe SSD, but I think 4 or 8 cores can go a long way with just processing user requests, COBS searching and writing the response back. RAM is always the case of the higher, the better. The higher the RAM, the more pages we can keep in memory to avoid page faults. The bottleneck then becomes the disk, which has to be fast. However, [very fast SSDs are not that expensive.](https://www.tomshardware.com/news/sk-hynix-pcie-4-ssd-record-breaking-random-speeds)