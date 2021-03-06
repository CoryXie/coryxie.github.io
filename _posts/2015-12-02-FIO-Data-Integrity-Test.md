---
layout: post
title: "FIO Data Integrity Test"
description: 
headline: 
modified: 2015-12-02
category: FileSysDev
tags: [Filesystem]
imagefeature: 
mathjax: 
chart: 
comments: true
featured: true
---

Data integrity is a fundamental aspect of storage security and reliability. With the advent of network storage and new technology trends that result in new failure modes for storage, interesting challenges arise in ensuring data integrity. A minor integrity violation, when not detected by the higher level software on time, could cause further loss of data. For example, a bit-flip while reading a file system inode bitmap could cause the file system to overwrite an important file. Therefore, prompt detection of integrity violations is vital for the reliability and safety of the stored data. The Flexible I/O Tester (`fio`) can be used to test data integrity/retention. This blog post tries to record some mailing list reading when I was studying the topic of *Data Integrity Verification*.

`fio` is an I/O tool meant to be used both for benchmark and stress/hardware verification. It has support for 19 different types of I/O engines (sync, mmap, libaio, posixaio, SG v3, splice, null, network, syslet, guasi, solarisaio, and more), I/O priorities (for newer Linux kernels), rate I/O, forked or threaded jobs, and much more. It can work on block devices as well as files. `fio` accepts job descriptions in a simple-to-understand text format. Several example job files are included. `fio` displays all sorts of I/O performance information, including complete IO latencies and percentiles. `fio` is in wide use in many places, for both benchmarking, QA, and verification purposes. It supports Linux, FreeBSD, NetBSD, OpenBSD, OS X, OpenSolaris, AIX, HP-UX, Android, and Windows.

# Problem

Someone called *Ruvinsky Konstantin* was [wondering](http://www.spinics.net/lists/fio/msg01933.html) about the correct way to perform a data integrity check while using a high stress scenario with `fio`.

* Scenario 1: File system, could the following command be used for data integrity check of IO on device with FS?

```console
	
	fio --name=dedup --ioengine=libaio --rw=randwrite --bs=16k --numjobs=64 --thread  --direct=1 --write_iolog=C --size=400m --directory=/mnt --do_verify=1 --verify=md5

```

He got the message that "Multiple writers may overwrite blocks that belong to other jobs. This can cause verification failures."

In the fio man pages, it is written: 

	filename=str 	Fio normally makes up a filename based on the job name,
	            	thread number, and file number. If you want to share
	            	files between threads in a job or several jobs, specify
	            	a filename for each of them to override the default

Actually, `fio` creates 64 files and according to man pages every thread is writing to its file. Since `-iodepth=1` (default) why should there be a problem with multiple writers even in the case of async ioengine?

He then asked, "If this is not correct, can someone suggest a data integrity check command in multithreaded `fio` run for devices with FS?".

* Scenario 2: RAW device, could the flag `--lockfile=readwrite` in the following fio command solve the problem of "Multiple writers may overwrite blocks that belong to other jobs"?

```console

	fio --name=dedup --ioengine=posixaio --rw=randwrite --lockfile=readwrite --bs=32k --numjobs=64 --thread --size=100m --verify=md5 --do_verify=1 --direct=1 --filename=/dev/hdiskpower2 --verify_fatal=1 --invalidate=0

```

In the `fio` man pages, it is written: 

	lockfile=readwrite      Read-write locking on the file. Many
	                        readers may access the file at the
	                        same time, but writes get exclusive
	                        access.

Taking into account that `iodepth=1` and only one thread can write in a given time, one can suppose that it solves the problem of blocks overwrite since it looks similar to one threaded writer, isn't it?

He then asked once again, "If this is not correct, can someone explain why and suggest a data integrity check command in multithreaded fio run for raw devices?". 

The above email was not answered by anyone in the `fio` mailing list, but it does not mean no one cared about this issue. *Grant Grundler* who has been working on IO subsystems since the mid 1980s, initiated some enhancments to Storage Data Integrity Testing on `fio`. He instructed his intern called *Juan Casse* to work on this topic.

# Design

First, *Grant Grundler* sent out an email to the `fio` maintainer *Jens Axboe* and started the *Desigh Phase* of this enhancment project. 

According to the email threads, *Grant Grundler* summarized the requirements for using `fio` to do Storage Data Integrity Testing as below:

* Verify data written to storage is available and correct.

In fact, `fio` had some basic verify features before 2013 when *Grant Grundler* initiated this enhancment project, which should have matched this requirement.

* Log writes (save a map) so we can repeatedly verify writes at a later date (weeks or months later).

In fact `fio` has `axmap` to log writes, but *Jens Axboe* had preference to `lfsr` (Linear Feedback Shift Register) to reproduce data for verification; `fio` supports this with `random_generator=lfsr`. *Jens Axboe* made a note that `lfsr` (Linear Feedback Shift Register) is basically a way to generate a "random" sequence of numbers that are guaranteed not to repeat until the cycle is repeated. With this you never have to do on-the-side tracking to avoid overlaps or overwrites.

* Provide some "bread crumbs" for debugging when data is NOT correct. (Not available typically will result in reported errors). 

*Grant Grundler* stated four pieces of data for this "bread crumbs":

 >- timestamp
 >- LBA written (e.g. if it's a partition, that means the offset into the partition)
 >- magic number for that test run (think of it as a GUID - verifies the block was written by `fio`)
 >- generation number in the case that we rewrite an LBA - so we can detect stale data

Note that we want `fio` to check for stale data between multiple writes, to do this we would like to add a generation number, which counts the number of times the same block has been written. In fact, fio has `unsigned short numberio` in its `struct vhdr_meta` for `verify=meta` mode at the time of that email thread, but it was not initially used as a strict generation number. 

Also, in fio `meta` verify mode, there was actually already a timestamp there (sec + usec) in `struct vhdr_meta`, besides the notion of a generation number in `unsigned short numberio` (it includes what number write this was).

The `struct vhdr_meta` structure looked like this at that time of the discussion:

```c

	struct vhdr_meta
	{
	        uint64_t offset;
	        unsigned char thread;
	        unsigned short numberio;
	        unsigned long time_sec;
	        unsigned long time_usec;
	};

```

Note that the above `struct vhdr_meta` was moved into generic verify header structure in latest `fio` source tree. But the same constructs are still the same.

```c

	/*
	 * A header structure associated with each checksummed data block. It is
	 * followed by a checksum specific header that contains the verification
	 * data.
	 */
	struct verify_header {
		uint16_t magic;
		uint16_t verify_type;
		uint32_t len;
		uint64_t rand_seed;
		uint64_t offset;
		uint32_t time_sec;
		uint32_t time_usec;
		uint16_t thread;
		uint16_t numberio;
		uint32_t crc32;
	};

```

* Verify data retention: we wanted the "verify" to be an option to the "read" workload mix. 
 
So not necessarily all data that gets written will get verified "during" the write workload. The reason is that performance statistics needs to be as consistent as possible without "verify" in a mixed read/write workload. *Jens Axboe* pointed out that `fio` already supported this kind of usage at that time of the discussion. Simply do the write workload with `do_verify=0`, then do a similar read workload with `do_verify=1` and the same verify checksum settings, etc. 

To verify everything that was written or trimmed, we can invoke `fio` again (think autotest invoking `fio` twice per test run) to check for retention. And then invoke fio many more times while the device is getting baked in a thermal chamber. *Jens Axboe* pointed out once again, that in fact, trim verification can be done if the device supports persistent and guaranteed zero return on a completed trim by `trim_verify_zero` (Verify that trim/discarded blocks are returned as zeroes). If that isn't set, trimmed regions are simply ignored for a verify.

This should be sufficient for data retention testing as well: If you write with `do_verify=0` and then 3 months later read with `do_verify=1`.

* Work on any block storage device (ie no knowledge of specific device geometry or flash vs magnetic vs optical or removable vs built-in - some 'workloads' might be geared for specific types of storage).

The original `fio` should have matched this requirement.

* Specify workload the same way `fio` does (multi-threaded, async, random vs seq, read vs write mix, etc).

The original `fio` should have matched this requirement.

* Collect same performance statistics that fio does (latency histograms in particular).

The original `fio` should have matched this requirement.

Since these are destructive tests, we can expect the primary target "audience" is anyone working on Storage HW or wants to confirm Storage HW is operating correctly before deploying $$$ worth of HW.

BTW, to eventually support adding "trim/discard command" testing into the mix, we would need to know when a block is explicitly unmapped and should be all zeros if we attempt to read it.

# Implementation

*Juan Casse* followed *Grant Grundler* by sending out another email thread, giving out more details about their intended implementation.

## Phase 1: Write data to storage device

Run `fio` with a job file specifying the following options.

* `readwrite`=

 - `randwrite` (does random writes)

 - `randrw` (does random reads and writes)

* `size=64k` or any other size (specifies the total amount of data that will be written/read)

* `bs=8192` or any other size (specifies the size of each data block)

* `runtime=1` or any other number of seconds (specifies the duration of the fio run)

* `time_based` (tells `fio` to run based on time rather than iops)

* `verify=meta` (adds `block number`, `numberio` and `timestamp` to the block verify header)

* `verify_pattern=0xffffffffffffffff` or any other pattern (fills the rest of the block with the specified pattern)

* `verify_dump=1` (writes to a file the data read from disk and the data that was expected, in case of data corruption)

* `continue_on_error=verify` (allows `fio` to continue even if corrupted blocks are found, otherwise
`fio` will stop execution on the first corrupted block)

* `random_generator=lfsr` (use `lfsr` as the random number generator)

## Phase 2: Verify data

The `fio` verification modes checksum both the stored header, as well as the actual contents (in fact `fio` checksums the verify header separately with the contents). If the verify header is good, we can check the actual data. If the actual data is not good, we can recreate the original content (based on `verify_pattern`) and compare with what is on disk. That gives us a pretty good idea of what was destroyed and how. But as *Jens Axboe* stated, it does not have the required bits for real retention testing, like timestamp and/or sequence. That could be added to the `verify_header` structure, or it could be a specific part of e.g the meta verify. The latter has the offset written already, for instance.

Recall that we mentioned adding a "generation number" to the block header data? Well, *Juan Casse* updated the `fio` source code to use the existing `numberio` instead.

Run fio with the same options as before plus specifying a new option:

 - `data_integrity_check`

When this option is given in the job file, fio will "replay" the workload (without actually writing data to storage). Once we have done this, we can read each block back and compare its `numberio` with the one obtained by running `lfsr` in reverse. We run `lfsr` in reverse because the `numberio` that was last written to a block will be found toward the end of the `lfsr` sequence when the data is written multiple times, which is what we want.

The `numberio` is incremented each time we read or write, so it can easily be computed going backwards by decrementing its value.

*Juan Casse* asked two questions to *Jens Axboe* but got no reply from *Jens Axboe*: How is the block offset computed from the `lfsr`? Do you see any problems trying to compute the offset going backwards with the `lfsr`?

One way to perform the data integrity check is to verify each block in order of block number (offset). For each block number, run the `lfsr` backwards starting from the end until we hit the block number. We then compare the `numberio` obtained by running the `lfsr` backwards with the one read from storage.

In the `fio` code, the function `do_io()` performs the workload specified, whether it be writes, reads and writes, or just reads. The function `do_verify()` is executed after `do_io()` only when the workload does any writes. If the workload does only reads, `do_verify()` is not executed. This function reads the blocks back and compares the
offset (block number). I already have code in place that checks for `numberio` as well.

However, if the job file specifies to run based on time rather than total number of bytes (setting `runtime=int` and `time_based`), then `do_verify()` is not performed. We would also need to run `do_verify()` in this case to make sure that the correct data was indeed written to storage.

Note: `fio` needs to be run based on time if we want `numberio` incremented when a block is rewritten. If we set fio to run a number of iterations instead (by specifying `loops=int`), the same `numberio` will be written every time the block is rewritten.

# References

This blog entry is mostly an edit based on the following sources, credits should go to these authors!

* http://www.spinics.net/lists/fio/msg02256.html
* http://www.spinics.net/lists/fio/msg02341.html
* http://www.spinics.net/lists/fio/msg01933.html
* Also [*Ensuring Data Integrity in Storage: Techniques and Applications*](https://www.fsl.cs.sunysb.edu/docs/integrity-storagess05/integrity.html) provides extensive analysis for Data Integrity issues.