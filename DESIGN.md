## The design of littlefs

A little fail-safe filesystem designed for microcontrollers.

```
   | | |     .---._____
  .-----.   |          |
--|o    |---| littlefs |
--|     |---|          |
  '-----'   '----------'
   | | |
```

littlefs was originally built as an experiment to learn about filesystem design
in the context of microcontrollers. Specifically, How would you build a
filesystem that is resilient to power-loss and flash wear without using
unbounded memory?

**TODO add some segway?**

## The problem

The embedded systems littlefs targets are usually 32-bit microcontrollers with
around 32KiB of RAM and 512KiB of ROM. These are often paired with SPI NOR
flash chips with about 4MiB of flash storage. These devices are too small for
Linux and most filesystems, requiring code written specifically with size in
mind.

Flash itself is an interesting piece of technology with its own quirks and
nuance. Unlike other forms of storage, writing to flash requires two
operations: erasing and programming. Programming (setting bits to 0) is
relatively cheap and can be very granular. Erasing however (setting bits to 1),
requires an expensive and destructive operation that gives flash its name.
[Wikipedia](https://en.wikipedia.org/wiki/Flash_memory) has more information
about this if you are interested in how flash works.

To make things more annoying, it's common for these embedded systems to lose
power at any time. Usually, microcontroller code is simple and reactive, with
no concept of a shutdown routine. This presents a big challenge for persistent
storage, where an unlucky power loss can corrupt the storage and leave a device
unrecoverable.

This leaves us with three major requirements for an embedded filesystem.

1. **Power-loss resilience** - On these systems, power can be lost at any time.
   If a power loss corrupts any persistent data structures, this can cause the
   device to become unrecoverable. An embedded filesystem must be designed to
   recover from a power loss during any write operation.

1. **Wear leveling** - Writing to flash is destructive. If a filesystem
   repeatedly writes to the same block, eventually that block will wear out.
   Filesystems that don't take wear into account can easily burn through blocks
   used to store frequently updated metadata and cause a device's early death.

1. **Bounded RAM/ROM** - If the above requirements weren't enough, these
   systems also have very limited amounts of memory. This prevents many
   existing filesystem designs, which can lean on relatively large amounts of
   RAM to temporarily store filesystem metadata.

   For ROM, this means we need to keep our design simple and reuse code paths
   were possible. For RAM we have a stronger requirement, all RAM usage is
   bounded. This means RAM usage does not grow as the filesystem changes in
   size or number of files. This creates a unique challenge as even presumably
   simple operations, such as traversing the filesystem, become surprisingly
   difficult.

**---------- below is old ---------**

For a bit of backstory, the littlefs was developed with the goal of learning
more about filesystem design by tackling the relative unsolved problem of
managing a robust filesystem resilient to power loss on devices
with limited RAM and ROM.

The embedded systems the littlefs is targeting are usually 32 bit
microcontrollers with around 32KB of RAM and 512KB of ROM. These are
often paired with SPI NOR flash chips with about 4MB of flash storage.

Flash itself is a very interesting piece of technology with quite a bit of
nuance. Unlike most other forms of storage, writing to flash requires two
operations: erasing and programming. The programming operation is relatively
cheap, and can be very granular. For NOR flash specifically, byte-level
programs are quite common. Erasing, however, requires an expensive operation
that forces the state of large blocks of memory to reset in a destructive
reaction that gives flash its name. The [Wikipedia entry](https://en.wikipedia.org/wiki/Flash_memory)
has more information if you are interested in how this works.

This leaves us with an interesting set of limitations that can be simplified
to three strong requirements:

1. **Power-loss resilient** - This is the main goal of the littlefs and the
   focus of this project.

   Embedded systems are usually designed without a shutdown routine and a
   notable lack of user interface for recovery, so filesystems targeting
   embedded systems must be prepared to lose power at any given time.

   Despite this state of things, there are very few embedded filesystems that
   handle power loss in a reasonable manner, and most can become corrupted if
   the user is unlucky enough.

2. **Wear leveling** - Due to the destructive nature of flash, most flash
   chips have a limited number of erase cycles, usually in the order of around
   100,000 erases per block for NOR flash. Filesystems that don't take wear
   into account can easily burn through blocks used to store frequently updated
   metadata.

   Consider the [FAT filesystem](https://en.wikipedia.org/wiki/Design_of_the_FAT_file_system),
   which stores a file allocation table (FAT) at a specific offset from the
   beginning of disk. Every block allocation will update this table, and after
   100,000 updates, the block will likely go bad, rendering the filesystem
   unusable even if there are many more erase cycles available on the storage
   as a whole.

3. **Bounded RAM/ROM** - Even with the design difficulties presented by the
   previous two limitations, we have already seen several flash filesystems
   developed on PCs that handle power loss just fine, such as the
   logging filesystems. However, these filesystems take advantage of the
   relatively cheap access to RAM, and use some rather... opportunistic...
   techniques, such as reconstructing the entire directory structure in RAM.
   These operations make perfect sense when the filesystem's only concern is
   erase cycles, but the idea is a bit silly on embedded systems.

   To cater to embedded systems, the littlefs has the simple limitation of
   using only a bounded amount of RAM and ROM. That is, no matter what is
   written to the filesystem, and no matter how large the underlying storage
   is, the littlefs will always use the same amount of RAM and ROM. This
   presents a very unique challenge, and makes presumably simple operations,
   such as iterating through the directory tree, surprisingly difficult.

## Existing designs?

There are of course, many different existing filesystem. Here is a very rough
summary of the general ideas behind some of them.

Most of the existing filesystems fall into the one big category of filesystem
designed in the early days of spinny magnet disks. While there is a vast amount
of interesting technology and ideas in this area, the nature of spinny magnet
disks encourage properties, such as grouping writes near each other, that don't
make as much sense on recent storage types. For instance, on flash, write
locality is not important and can actually increase wear.

One of the most popular designs for flash filesystems is called the
[logging filesystem](https://en.wikipedia.org/wiki/Log-structured_file_system).
The flash filesystems [jffs](https://en.wikipedia.org/wiki/JFFS)
and [yaffs](https://en.wikipedia.org/wiki/YAFFS) are good examples. In a
logging filesystem, data is not stored in a data structure on disk, but instead
the changes to the files are stored on disk. This has several neat advantages,
such as the fact that the data is written in a cyclic log format and naturally
wear levels as a side effect. And, with a bit of error detection, the entire
filesystem can easily be designed to be resilient to power loss. The
journaling component of most modern day filesystems is actually a reduced
form of a logging filesystem. However, logging filesystems have a difficulty
scaling as the size of storage increases. And most filesystems compensate by
caching large parts of the filesystem in RAM, a strategy that is inappropriate
for embedded systems.

Another interesting filesystem design technique is that of [copy-on-write (COW)](https://en.wikipedia.org/wiki/Copy-on-write).
A good example of this is the [btrfs](https://en.wikipedia.org/wiki/Btrfs)
filesystem. COW filesystems can easily recover from corrupted blocks and have
natural protection against power loss. However, if they are not designed with
wear in mind, a COW filesystem could unintentionally wear down the root block
where the COW data structures are synchronized.

**-- new below --**

## Existing designs?

So what's out there? There are, of course, many different filesystems, however
they often share and borrow feature from each other. If we only look at
power-loss resilience and wear leveling, we can narrow these down to only a
handful of designs.

<!-- pic here? -->

1. First we have the non-resilient block based filesystems, such as FAT and
   ext2. These are the the earliest filesystem designs and often the most
   simple. Here storage is divided into blocks, with each file being stored
   in a collection of blocks.  Without modifications, these filesystems are
   not power-loss resilient, so updating a file is a simple as rewriting the
   blcoks in place.

   Because of their simplicity, these filesystems are usually both the fastest
   and smallest. However the lack of power resilience is, of course, bad, and
   the binding relationship of storage location and data removes the
   filesystem's ability to manage wear.

<!-- pic here? -->

2. In a completely different direction, we have logging filesystems, such as
   JFFS, YAFFS, and SPIFFS. In a logging filesystem, storage location is not
   bound to a piece of data, instead the entire storage is used for a circular
   log which is appended with every change made to the filesystem. Writing
   appends new changes, while reading requires traversing the log to
   reconstruct a file. Some logging filesystems cache files to avoid the read
   cost, but this comes at a tradeoff of RAM.

   Logging filesystem are beautifully elegant. With a checksum, we can easily
   detect power-loss and fall back to the previous state by ignoring failed
   appends. And if that wasn't good enough, their cyclic nature means that
   logging filesystems distribute wear across storage perfectly.

   The main downside is performance. If we look at garbage collection, the
   process of cleaning up outdated data from the end of the log, I've yet to
   see a pure logging filesystem that does not have one of these two costs:

   1. O(n^2) runtime
   2. O(n) RAM

   SPIFFS is a very interesting case here, as it uses the fact that repeated
   programs to NOR flash is both atomic and masking. This is a very neat
   solution, however it limits the type of storage you can support.

<!-- pic here? -->

3. Perhaps the most common type of filesystem, a journaling filesystem, such
   as ext4 and NTFS, is what happens when you try to mate a block based
   filesystem with a logging filesystem. Here, we take a normal block based
   filesystem and add a bounded log where we note every change before it
   occurs.

   This sort of filesystem takes the best from both worlds. Performance can be
   as fast as a block based filesystem (though updating the journal does slow
   it down a bit), and atomic updates to the journal allow the filesystem to
   recover in the event of a power loss.

   Unfortunately, journaling filesystems have two problems. They are fairly
   complex, since there are effectively two filesystems running in parallel,
   which comes with a code size cost. They also offer no protection against
   wear because of the strong relationship between storage location and data.

<!-- pic here? -->

4. Last but not least we have copy-on-write (COW) filesystems, such as btrfs
   and ZFS. These are very similar to other block based filesystems, but
   instead of updating block inplace, all updates are performed by creating
   a copy and then updating the reference to the block. This effectively
   pushes all of our problems upwards until we reach the root of our
   filesystem, which is often stored in a very small log.

   COW filesystems are very enticing. They offer very similar performance to
   block based filesystems while managing to pull off atomic updates without
   storing data changes directly in a log. They even disassociate the storage
   location of data, which creates the opportunity for wear leveling.

   Well, almost. The unbounded upwards movement of updates causes some
   problems. Because updates to a COW filesystem don't stop until they've
   reached the root, an update can cascade into a larger set of writes than
   the original data. On top of this, the upward motion focuses these writes
   into the block, which can wear out much earlier than the rest of our
   filesystem.

## littlefs

So what does littlefs do?

If we look at existing filesystems, there are two specific design patterns that
stand out, but each have their own set of problems. Logging, which provides
independent atomicity, has poor runtime performance, and COW structures, which
perform well, push the atomicity problem upwards.

Can we work around these limitations?

Consider logging. It has either a O(n^2) runtime or O(n) RAM cost. We can't
avoid these costs, _but_ if we put an upper bound on the size we can at least
prevent the theoretical cost from becoming problem. This is sort of like the
old computer science trick where you can reduce any algorithmic complexity to
O(1) by simply bounding the input.

In the case of COW structures, we can try twisting the definition a bit. Lets
say that our COW structure doesn't copy after a single write, but instead
copies after n writes. This doesn't change most COW properties (assuming you
can write atomically!), but what it does do is prevents the upward motion of
wear. This sort of copy-on-bounded-writes (COBW) still focuses wear, but at
each level we divide the propogation of wear by n. With a sufficiently
large n wear propogation is no longer a problem.

Do you see where this is going? Seperate, logging and COW are imperfect
solutions and have weaknesses that limit their usefulness. But if we merge
the two they can mutually solve each other's limitations.

This is the idea behind littlefs. At the sub-block level, littlefs is built
out of small, two blocks logs that provide atomic updates to metadata anywhere
on the filesystem. At the super-block level, littlefs is a COW tree of blocks
that can be evicted on demand.

<!-- pic here? -->

There are still some minor issues. Small logs can be expensive in terms of
storage, in the worst case a small log costs 4x the size of the original data.
COBW structures require an efficient block allocator since allocation occurs
every n writes. And there is still the challenge of keeping the RAM usuage
constant.

**- scratch below -**








COW updates still focus wear as it moves upwards, but at each level the wear is
divided by n. With a sufficiently large n we can completely remove the focus of
wear.






but we can use an old computer science trick where putting
an upper bound on our input reduces the complexity to O(1).


 If n is a constant



We can actually work around some of these limitations. 




We can actually improve log performance with a sort of computer science hack. 





At a high level, littlefs attempts to merge these into two layers ordered by
scale.


 by considering the filesystem
to be split into two orders of scale.


storage to be
split 


 by splitting the filesystem into two layers
of scale

littlefs tries to attack the problem by splitting it
into two different orders of scale.

**- TODO move this to readme? -**
## High level

At a high level, littlefs is a block based filesystem that uses small logs to
store metadata and larger copy-on-write (COW) structures to store file data.

In littlefs, these ingredients form a sort of two-layered cake, with the small
logs (called metadata pairs) providing fast updates to metadata anywhere on
storage, while the COW structures store file data compactly and without any
wear amplification cost.

Both of these data structures are built out of blocks, which are fed by a
common block allocator. By limiting the number of erases allowed on a block
per allocation, the allocator provides dynamic wear leveling over the entire
filesystem.

```
            .--------.                                      \
            |root dir|-.                                    |
            | pair 0 | |                                    |     
   .--------|        |-'                                    |
   |        '--------'                                      |
   |        .-'    '-------------------------.              |
   |       v                                  v             +- metadata
   |  .--------.        .--------.        .--------.        |
   '->| dir A  |------->| dir A  |------->| dir B  |        |
      | pair 0 |        | pair 1 |        | pair 0 |        |
      |        |        |        |        |        |        |
      '--------'        '--------'        '--------'        |
      .-'    '-.            |             .-'    '-.        /
     v          v           v            v          v
.--------.  .--------.  .--------.  .--------.  .--------.  \
| file C |  | file D |  | file E |  | file F |  | file G |  |
| blck 0 |  | blck 0 |  | blck 0 |  | blck 0 |  | blck 0 |  |
|        |  |        |  |        |  |        |  |        |  |
'--------'  '--------'  '--------'  '--------'  '--------'  |
    |                      | |          |                   |
    v                      v |          v                   |
.--------.              .--------.  .--------.              |
| file C |              | file E |  | file F |              |
| blck 0 |              | blck 0 |  | blck 0 |              +- file data
|        |              |        |  |        |              |
'--------'              '--------'  '--------'              |
                           | |                              |
                           v v                              |
                        .--------.                          |
                        | file E |                          |
                        | blck 0 |                          |
                        |        |                          |
                        '--------'                          /
     
    ^                                               |       \
    | block allocator                               v       |
.--------.  .--------.  .--------.  .--------.  .--------.  |
|        |  |        |  |        |  |        |  |        |  +- free blocks
|        |<-|        |<-|        |<-|        |<-|        |  |
|        |  |        |  |        |  |        |  |        |  |
'--------'  '--------'  '--------'  '--------'  '--------'  /
```

The key to making littlefs efficient is the implementation of these components.


**- TODO move this to readme? -**



**-- scratch below --**

Fortunately for us, data locality has very little
impact on flash, so all blocks are treated equal









This limits the cost of garbage collection while creating a balance
between the upward cost of COW operations and the distribution of write
operations.



If we look at littlefs as a whole, these ingredients form a sort of two-layered
cake, with 

```
```



**-- new below? --**

## Metadata pairs

Metadata pairs are the backbone of littlefs. These are small, two block logs
that allow atomic updates anywhere in the filesystem.

Why two blocks? Well, logs work by appending entries to a circular buffer
stored on disk. But remember that flash has limited write granularity. We can
incrementally program new data onto erased blocks, but we need to erase a full
block at a time. This means that in order for our circular buffer to work, we
need more than one block.

We could make our logs larger than two blocks, but the next challenge is how
do we store references to these logs. Because the blocks themselves are erased
during operation, using a data structure to track these blocks is complicated.
The simple solution here is to store a two block addresses for every metadata
pair. This has the added advantage that we can change out blocks in the
metadata pair independently, and we don't reduce our block granularity for
other operations.

In order to determine which metadata block is the most recent, we store a
revision count that we compare using sequence arithmetic (very handy for
avoiding problems with integer overflow). Conventiently, this revision count
also gives us a rough idea of how many erases have occured on the block.

--- <!-- picture of metadata pair tree? -->

So how do atomically update our metadata pairs? Atomicity (a type of power-loss
resilience) requires two parts, redundancy and error detection.  Error
detection can be provided with a checksum, in littlefs's case we use a 32-bit
CRC. Maintaining redundancy, on the other hand, requires multiple stages.

1. If our block is not full and the program size is small enough to let us
   append more entries, we can simply append the entries to the log. Because
   we don't overwrite the original entries (remember rewriting flash requires
   an erase), we still have the original entries if we lose power during the
   append.
   
   Note that littlefs doesn't maintain a checksum for each entry. Many logging
   filesystems do this, but it limits what you can update in a single atomic
   operation. What we can do instead is group multiple entries into a commit
   that shares a single checksum. This lets us update multiple unrelated pieces
   of metadata as long as they reside on the same metadata pair.

2. If our block _is_ full of entries, we need to somehow remove outdated
   entries to make space for new ones. This process is called garbage
   collection, but because littlefs has multiple garbage collectors we refer
   to it as compaction when talking about metadata pairs.

   Compared to other filesystems, littlefs's garbage collector is relatively
   simple. We want to avoid RAM consumption, so we use a sort of brute force
   solution. For each entry we check to see if a newer entry has been written,
   if the entry is the most recent we append it to our new block. This is where
   having two blocks becomes important, if we lose power we still have
   everything in our original block.

   This compaction step is also when we erase the metadata block and increment
   the revision count. Because we can commit multiple entries at once, we can
   write all of these changes to the second block without worrying about power
   loss. It's only when the commit's CRC is written that the entries and
   revision count become committed and readable.


3. If our block size is full of entries _and_ we can't find any garbage, then
   what?



 to
   clean, then what?
3.

    

   Note that this is also when we erase our increment our revision count
   
   




 the extra block in our metadata pair. This is where
   having two blocks becomes important because if we lose power we still have
   the data in our original block. To determine which block is the most 


littlefs's garbage
   collector is relatively simple. Because 
   


but since littlefs has multiple garbage collectors we call
   this compaction.


  is called garbage collection
 



 we need to collect any outdated entries. 
   perform garbage collection in what littlefs calls
   the "compact" step. Here we use a RAM conservative approach where we check
   that each entry is the most recent in the metadata pair, if it is we write
   it to our new block. 


iterate
over each entry for each entry 


Many logs use a checksum for each entry, but this means that each entry is
its own synchronization point if power is lost. In littlefs's case 

So what is stored in metadata pairs?



**-- scratch below --**



are a circular buffer of updates


 work by appending entries to a circular buffer
stored in disk. But remember that flash is limited to performing block sized
erases. We can incrementally program new data, but in order to write over old
data we need to erase that block at some point. This means that in order for
logs to function, we need to 


 Well, the way logs work is that we append entries to the log
until the log is full. But once the log is full we can't just become
unresponsive, we need to clean up outdated entries. This process is called
garbage collection. We can do this by iterating through the oldest entries and
move any entries that are still valid to the end of the log.


## Metadata pairs

So, 



Metadata pairs form the backbone of littlefs and provide a system for
distributed atomic updates. A metadata pair is a small, two block log that can
be updated atomically.








Metadata pairs provide the backbone for littlefs. These are small, two block
logs that can be updated atomically.




The backbone of littlefs 

The cornerstone of littlefs are the metadata pairs 

**-- old below --**

## Metadata pairs

The core piece of technology that provides the backbone for the littlefs is
the concept of metadata pairs. The key idea here is that any metadata that
needs to be updated atomically is stored on a pair of blocks tagged with
a revision count and checksum. Every update alternates between these two
pairs, so that at any time there is always a backup containing the previous
state of the metadata.

Consider a small example where each metadata pair has a revision count,
a number as data, and the XOR of the block as a quick checksum. If
we update the data to a value of 9, and then to a value of 5, here is
what the pair of blocks may look like after each update:
```
  block 1   block 2        block 1   block 2        block 1   block 2
.---------.---------.    .---------.---------.    .---------.---------.
| rev: 1  | rev: 0  |    | rev: 1  | rev: 2  |    | rev: 3  | rev: 2  |
| data: 3 | data: 0 | -> | data: 3 | data: 9 | -> | data: 5 | data: 9 |
| xor: 2  | xor: 0  |    | xor: 2  | xor: 11 |    | xor: 6  | xor: 11 |
'---------'---------'    '---------'---------'    '---------'---------'
                 let data = 9             let data = 5
```

After each update, we can find the most up to date value of data by looking
at the revision count.

Now consider what the blocks may look like if we suddenly lose power while
changing the value of data to 5:
```
  block 1   block 2        block 1   block 2        block 1   block 2
.---------.---------.    .---------.---------.    .---------.---------.
| rev: 1  | rev: 0  |    | rev: 1  | rev: 2  |    | rev: 3  | rev: 2  |
| data: 3 | data: 0 | -> | data: 3 | data: 9 | -x | data: 3 | data: 9 |
| xor: 2  | xor: 0  |    | xor: 2  | xor: 11 |    | xor: 2  | xor: 11 |
'---------'---------'    '---------'---------'    '---------'---------'
                 let data = 9             let data = 5
                                          powerloss!!!
```

In this case, block 1 was partially written with a new revision count, but
the littlefs hadn't made it to updating the value of data. However, if we
check our checksum we notice that block 1 was corrupted. So we fall back to
block 2 and use the value 9.

Using this concept, the littlefs is able to update metadata blocks atomically.
There are a few other tweaks, such as using a 32 bit CRC and using sequence
arithmetic to handle revision count overflow, but the basic concept
is the same. These metadata pairs define the backbone of the littlefs, and the
rest of the filesystem is built on top of these atomic updates.

## Non-meta data

Now, the metadata pairs do come with some drawbacks. Most notably, each pair
requires two blocks for each block of data. I'm sure users would be very
unhappy if their storage was suddenly cut in half! Instead of storing
everything in these metadata blocks, the littlefs uses a COW data structure
for files which is in turn pointed to by a metadata block. When
we update a file, we create copies of any blocks that are modified until
the metadata blocks are updated with the new copy. Once the metadata block
points to the new copy, we deallocate the old blocks that are no longer in use.

Here is what updating a one-block file may look like:
```
  block 1   block 2        block 1   block 2        block 1   block 2
.---------.---------.    .---------.---------.    .---------.---------.
| rev: 1  | rev: 0  |    | rev: 1  | rev: 0  |    | rev: 1  | rev: 2  |
| file: 4 | file: 0 | -> | file: 4 | file: 0 | -> | file: 4 | file: 5 |
| xor: 5  | xor: 0  |    | xor: 5  | xor: 0  |    | xor: 5  | xor: 7  |
'---------'---------'    '---------'---------'    '---------'---------'
    |                        |                                  |
    v                        v                                  v
 block 4                  block 4    block 5       block 4    block 5
.--------.               .--------. .--------.    .--------. .--------.
| old    |               | old    | | new    |    | old    | | new    |
| data   |               | data   | | data   |    | data   | | data   |
|        |               |        | |        |    |        | |        |
'--------'               '--------' '--------'    '--------' '--------'
            update data in file        update metadata pair
```

It doesn't matter if we lose power while writing new data to block 5,
since the old data remains unmodified in block 4. This example also
highlights how the atomic updates of the metadata blocks provide a
synchronization barrier for the rest of the littlefs.

At this point, it may look like we are wasting an awfully large amount
of space on the metadata. Just looking at that example, we are using
three blocks to represent a file that fits comfortably in one! So instead
of giving each file a metadata pair, we actually store the metadata for
all files contained in a single directory in a single metadata block.

Now we could just leave files here, copying the entire file on write
provides the synchronization without the duplicated memory requirements
of the metadata blocks. However, we can do a bit better.

## CTZ skip-lists

There are many different data structures for representing the actual
files in filesystems. Of these, the littlefs uses a rather unique [COW](https://upload.wikimedia.org/wikipedia/commons/0/0c/Cow_female_black_white.jpg)
data structure that allows the filesystem to reuse unmodified parts of the
file without additional metadata pairs.

First lets consider storing files in a simple linked-list. What happens when we
append a block? We have to change the last block in the linked-list to point
to this new block, which means we have to copy out the last block, and change
the second-to-last block, and then the third-to-last, and so on until we've
copied out the entire file.

```
Exhibit A: A linked-list
.--------.  .--------.  .--------.  .--------.  .--------.  .--------.
| data 0 |->| data 1 |->| data 2 |->| data 4 |->| data 5 |->| data 6 |
|        |  |        |  |        |  |        |  |        |  |        |
|        |  |        |  |        |  |        |  |        |  |        |
'--------'  '--------'  '--------'  '--------'  '--------'  '--------'
```

To get around this, the littlefs, at its heart, stores files backwards. Each
block points to its predecessor, with the first block containing no pointers.
If you think about for a while, it starts to make a bit of sense. Appending
blocks just point to their predecessor and no other blocks need to be updated.
If we update a block in the middle, we will need to copy out the blocks that
follow, but can reuse the blocks before the modified block. Since most file
operations either reset the file each write or append to files, this design
avoids copying the file in the most common cases.

```
Exhibit B: A backwards linked-list
.--------.  .--------.  .--------.  .--------.  .--------.  .--------.
| data 0 |<-| data 1 |<-| data 2 |<-| data 4 |<-| data 5 |<-| data 6 |
|        |  |        |  |        |  |        |  |        |  |        |
|        |  |        |  |        |  |        |  |        |  |        |
'--------'  '--------'  '--------'  '--------'  '--------'  '--------'
```

However, a backwards linked-list does come with a rather glaring problem.
Iterating over a file _in order_ has a runtime cost of O(n^2). Gah! A quadratic
runtime to just _read_ a file? That's awful. Keep in mind reading files is
usually the most common filesystem operation.

To avoid this problem, the littlefs uses a multilayered linked-list. For
every nth block where n is divisible by 2^x, the block contains a pointer
to block n-2^x. So each block contains anywhere from 1 to log2(n) pointers
that skip to various sections of the preceding list. If you're familiar with
data-structures, you may have recognized that this is a type of deterministic
skip-list.

The name comes from the use of the
[count trailing zeros (CTZ)](https://en.wikipedia.org/wiki/Count_trailing_zeros)
instruction, which allows us to calculate the power-of-two factors efficiently.
For a given block n, the block contains ctz(n)+1 pointers.

```
Exhibit C: A backwards CTZ skip-list
.--------.  .--------.  .--------.  .--------.  .--------.  .--------.
| data 0 |<-| data 1 |<-| data 2 |<-| data 3 |<-| data 4 |<-| data 5 |
|        |<-|        |--|        |<-|        |--|        |  |        |
|        |<-|        |--|        |--|        |--|        |  |        |
'--------'  '--------'  '--------'  '--------'  '--------'  '--------'
```

The additional pointers allow us to navigate the data-structure on disk
much more efficiently than in a singly linked-list.

Taking exhibit C for example, here is the path from data block 5 to data
block 1. You can see how data block 3 was completely skipped:
```
.--------.  .--------.  .--------.  .--------.  .--------.  .--------.
| data 0 |  | data 1 |<-| data 2 |  | data 3 |  | data 4 |<-| data 5 |
|        |  |        |  |        |<-|        |--|        |  |        |
|        |  |        |  |        |  |        |  |        |  |        |
'--------'  '--------'  '--------'  '--------'  '--------'  '--------'
```

The path to data block 0 is even more quick, requiring only two jumps:
```
.--------.  .--------.  .--------.  .--------.  .--------.  .--------.
| data 0 |  | data 1 |  | data 2 |  | data 3 |  | data 4 |<-| data 5 |
|        |  |        |  |        |  |        |  |        |  |        |
|        |<-|        |--|        |--|        |--|        |  |        |
'--------'  '--------'  '--------'  '--------'  '--------'  '--------'
```

We can find the runtime complexity by looking at the path to any block from
the block containing the most pointers. Every step along the path divides
the search space for the block in half. This gives us a runtime of O(log n).
To get to the block with the most pointers, we can perform the same steps
backwards, which puts the runtime at O(2 log n) = O(log n). The interesting
part about this data structure is that this optimal path occurs naturally
if we greedily choose the pointer that covers the most distance without passing
our target block.

So now we have a representation of files that can be appended trivially with
a runtime of O(1), and can be read with a worst case runtime of O(n log n).
Given that the the runtime is also divided by the amount of data we can store
in a block, this is pretty reasonable.

Unfortunately, the CTZ skip-list comes with a few questions that aren't
straightforward to answer. What is the overhead? How do we handle more
pointers than we can store in a block? How do we store the skip-list in
a directory entry?

One way to find the overhead per block is to look at the data structure as
multiple layers of linked-lists. Each linked-list skips twice as many blocks
as the previous linked-list. Another way of looking at it is that each 
linked-list uses half as much storage per block as the previous linked-list.
As we approach infinity, the number of pointers per block forms a geometric
series. Solving this geometric series gives us an average of only 2 pointers
per block.

![overhead_per_block](https://latex.codecogs.com/svg.latex?%5Clim_%7Bn%5Cto%5Cinfty%7D%5Cfrac%7B1%7D%7Bn%7D%5Csum_%7Bi%3D0%7D%5E%7Bn%7D%5Cleft%28%5Ctext%7Bctz%7D%28i%29&plus;1%5Cright%29%20%3D%20%5Csum_%7Bi%3D0%7D%5Cfrac%7B1%7D%7B2%5Ei%7D%20%3D%202)

Finding the maximum number of pointers in a block is a bit more complicated,
but since our file size is limited by the integer width we use to store the
size, we can solve for it. Setting the overhead of the maximum pointers equal
to the block size we get the following equation. Note that a smaller block size
results in more pointers, and a larger word width results in larger pointers.

![maximum overhead](https://latex.codecogs.com/svg.latex?B%20%3D%20%5Cfrac%7Bw%7D%7B8%7D%5Cleft%5Clceil%5Clog_2%5Cleft%28%5Cfrac%7B2%5Ew%7D%7BB-2%5Cfrac%7Bw%7D%7B8%7D%7D%5Cright%29%5Cright%5Crceil)

where:  
B = block size in bytes  
w = word width in bits  

Solving the equation for B gives us the minimum block size for various word
widths:  
32 bit CTZ skip-list = minimum block size of 104 bytes  
64 bit CTZ skip-list = minimum block size of 448 bytes  

Since littlefs uses a 32 bit word size, we are limited to a minimum block
size of 104 bytes. This is a perfectly reasonable minimum block size, with most
block sizes starting around 512 bytes. So we can avoid additional logic to
avoid overflowing our block's capacity in the CTZ skip-list.

So, how do we store the skip-list in a directory entry? A naive approach would
be to store a pointer to the head of the skip-list, the length of the file
in bytes, the index of the head block in the skip-list, and the offset in the
head block in bytes. However this is a lot of information, and we can observe
that a file size maps to only one block index + offset pair. So it should be
sufficient to store only the pointer and file size.

But there is one problem, calculating the block index + offset pair from a
file size doesn't have an obvious implementation.

We can start by just writing down an equation. The first idea that comes to
mind is to just use a for loop to sum together blocks until we reach our
file size. We can write this equation as a summation:

![summation1](https://latex.codecogs.com/svg.latex?N%20%3D%20%5Csum_i%5En%5Cleft%5BB-%5Cfrac%7Bw%7D%7B8%7D%5Cleft%28%5Ctext%7Bctz%7D%28i%29&plus;1%5Cright%29%5Cright%5D)

where:  
B = block size in bytes  
w = word width in bits  
n = block index in skip-list  
N = file size in bytes  

And this works quite well, but is not trivial to calculate. This equation
requires O(n) to compute, which brings the entire runtime of reading a file
to O(n^2 log n). Fortunately, the additional O(n) does not need to touch disk,
so it is not completely unreasonable. But if we could solve this equation into
a form that is easily computable, we can avoid a big slowdown.

Unfortunately, the summation of the CTZ instruction presents a big challenge.
How would you even begin to reason about integrating a bitwise instruction?
Fortunately, there is a powerful tool I've found useful in these situations:
The [On-Line Encyclopedia of Integer Sequences (OEIS)](https://oeis.org/).
If we work out the first couple of values in our summation, we find that CTZ
maps to [A001511](https://oeis.org/A001511), and its partial summation maps
to [A005187](https://oeis.org/A005187), and surprisingly, both of these
sequences have relatively trivial equations! This leads us to a rather
unintuitive property:

![mindblown](https://latex.codecogs.com/svg.latex?%5Csum_i%5En%5Cleft%28%5Ctext%7Bctz%7D%28i%29&plus;1%5Cright%29%20%3D%202n-%5Ctext%7Bpopcount%7D%28n%29)

where:  
ctz(x) = the number of trailing bits that are 0 in x  
popcount(x) = the number of bits that are 1 in x  

It's a bit bewildering that these two seemingly unrelated bitwise instructions
are related by this property. But if we start to dissect this equation we can
see that it does hold. As n approaches infinity, we do end up with an average
overhead of 2 pointers as we find earlier. And popcount seems to handle the
error from this average as it accumulates in the CTZ skip-list.

Now we can substitute into the original equation to get a trivial equation
for a file size:

![summation2](https://latex.codecogs.com/svg.latex?N%20%3D%20Bn%20-%20%5Cfrac%7Bw%7D%7B8%7D%5Cleft%282n-%5Ctext%7Bpopcount%7D%28n%29%5Cright%29)

Unfortunately, we're not quite done. The popcount function is non-injective,
so we can only find the file size from the block index, not the other way
around. However, we can solve for an n' block index that is greater than n
with an error bounded by the range of the popcount function. We can then
repeatedly substitute this n' into the original equation until the error
is smaller than the integer division. As it turns out, we only need to
perform this substitution once. Now we directly calculate our block index:

![formulaforn](https://latex.codecogs.com/svg.latex?n%20%3D%20%5Cleft%5Clfloor%5Cfrac%7BN-%5Cfrac%7Bw%7D%7B8%7D%5Cleft%28%5Ctext%7Bpopcount%7D%5Cleft%28%5Cfrac%7BN%7D%7BB-2%5Cfrac%7Bw%7D%7B8%7D%7D-1%5Cright%29&plus;2%5Cright%29%7D%7BB-2%5Cfrac%7Bw%7D%7B8%7D%7D%5Cright%5Crfloor)

Now that we have our block index n, we can just plug it back into the above
equation to find the offset. However, we do need to rearrange the equation
a bit to avoid integer overflow:

![formulaforoff](https://latex.codecogs.com/svg.latex?%5Cmathit%7Boff%7D%20%3D%20N%20-%20%5Cleft%28B-2%5Cfrac%7Bw%7D%7B8%7D%5Cright%29n%20-%20%5Cfrac%7Bw%7D%7B8%7D%5Ctext%7Bpopcount%7D%28n%29)

The solution involves quite a bit of math, but computers are very good at math.
Now we can solve for both the block index and offset from the file size in O(1).

Here is what it might look like to update a file stored with a CTZ skip-list:
```
                                      block 1   block 2
                                    .---------.---------.
                                    | rev: 1  | rev: 0  |
                                    | file: 6 | file: 0 |
                                    | size: 4 | size: 0 |
                                    | xor: 3  | xor: 0  |
                                    '---------'---------'
                                        |
                                        v
  block 3     block 4     block 5     block 6
.--------.  .--------.  .--------.  .--------.
| data 0 |<-| data 1 |<-| data 2 |<-| data 3 |
|        |<-|        |--|        |  |        |
|        |  |        |  |        |  |        |
'--------'  '--------'  '--------'  '--------'

|  update data in file
v

                                      block 1   block 2
                                    .---------.---------.
                                    | rev: 1  | rev: 0  |
                                    | file: 6 | file: 0 |
                                    | size: 4 | size: 0 |
                                    | xor: 3  | xor: 0  |
                                    '---------'---------'
                                        |
                                        v
  block 3     block 4     block 5     block 6
.--------.  .--------.  .--------.  .--------.
| data 0 |<-| data 1 |<-| old    |<-| old    |
|        |<-|        |--| data 2 |  | data 3 |
|        |  |        |  |        |  |        |
'--------'  '--------'  '--------'  '--------'
     ^ ^           ^
     | |           |      block 7     block 8     block 9    block 10
     | |           |    .--------.  .--------.  .--------.  .--------.
     | |           '----| new    |<-| new    |<-| new    |<-| new    |
     | '----------------| data 2 |<-| data 3 |--| data 4 |  | data 5 |
     '------------------|        |--|        |--|        |  |        |
                        '--------'  '--------'  '--------'  '--------'

|  update metadata pair
v

                                                   block 1   block 2
                                                 .---------.---------.
                                                 | rev: 1  | rev: 2  |
                                                 | file: 6 | file: 10|
                                                 | size: 4 | size: 6 |
                                                 | xor: 3  | xor: 14 |
                                                 '---------'---------'
                                                                |
                                                                |
  block 3     block 4     block 5     block 6                   |
.--------.  .--------.  .--------.  .--------.                  |
| data 0 |<-| data 1 |<-| old    |<-| old    |                  |
|        |<-|        |--| data 2 |  | data 3 |                  |
|        |  |        |  |        |  |        |                  |
'--------'  '--------'  '--------'  '--------'                  |
     ^ ^           ^                                            v
     | |           |      block 7     block 8     block 9    block 10
     | |           |    .--------.  .--------.  .--------.  .--------.
     | |           '----| new    |<-| new    |<-| new    |<-| new    |
     | '----------------| data 2 |<-| data 3 |--| data 4 |  | data 5 |
     '------------------|        |--|        |--|        |  |        |
                        '--------'  '--------'  '--------'  '--------'
```

## Block allocation

So those two ideas provide the grounds for the filesystem. The metadata pairs
give us directories, and the CTZ skip-lists give us files. But this leaves
one big [elephant](https://upload.wikimedia.org/wikipedia/commons/3/37/African_Bush_Elephant.jpg)
of a question. How do we get those blocks in the first place?

One common strategy is to store unallocated blocks in a big free list, and
initially the littlefs was designed with this in mind. By storing a reference
to the free list in every single metadata pair, additions to the free list
could be updated atomically at the same time the replacement blocks were
stored in the metadata pair. During boot, every metadata pair had to be
scanned to find the most recent free list, but once the list was found the
state of all free blocks becomes known.

However, this approach had several issues:

- There was a lot of nuanced logic for adding blocks to the free list without
  modifying the blocks, since the blocks remain active until the metadata is
  updated.
- The free list had to support both additions and removals in FIFO order while
  minimizing block erases.
- The free list had to handle the case where the file system completely ran
  out of blocks and may no longer be able to add blocks to the free list.
- If we used a revision count to track the most recently updated free list,
  metadata blocks that were left unmodified were ticking time bombs that would
  cause the system to go haywire if the revision count overflowed.
- Every single metadata block wasted space to store these free list references.

Actually, to simplify, this approach had one massive glaring issue: complexity.

> Complexity leads to fallibility.  
> Fallibility leads to unmaintainability.  
> Unmaintainability leads to suffering.  

Or at least, complexity leads to increased code size, which is a problem
for embedded systems.

In the end, the littlefs adopted more of a "drop it on the floor" strategy.
That is, the littlefs doesn't actually store information about which blocks
are free on the storage. The littlefs already stores which files _are_ in
use, so to find a free block, the littlefs just takes all of the blocks that
exist and subtract the blocks that are in use.

Of course, it's not quite that simple. Most filesystems that adopt this "drop
it on the floor" strategy either rely on some properties inherent to the
filesystem, such as the cyclic-buffer structure of logging filesystems,
or use a bitmap or table stored in RAM to track free blocks, which scales
with the size of storage and is problematic when you have limited RAM. You
could iterate through every single block in storage and check it against
every single block in the filesystem on every single allocation, but that
would have an abhorrent runtime.

So the littlefs compromises. It doesn't store a bitmap the size of the storage,
but it does store a little bit-vector that contains a fixed set lookahead
for block allocations. During a block allocation, the lookahead vector is
checked for any free blocks. If there are none, the lookahead region jumps
forward and the entire filesystem is scanned for free blocks.

Here's what it might look like to allocate 4 blocks on a decently busy
filesystem with a 32bit lookahead and a total of
128 blocks (512Kbytes of storage if blocks are 4Kbyte):
```
boot...         lookahead:
                fs blocks: fffff9fffffffffeffffffffffff0000
scanning...     lookahead: fffff9ff
                fs blocks: fffff9fffffffffeffffffffffff0000
alloc = 21      lookahead: fffffdff
                fs blocks: fffffdfffffffffeffffffffffff0000
alloc = 22      lookahead: ffffffff
                fs blocks: fffffffffffffffeffffffffffff0000
scanning...     lookahead:         fffffffe
                fs blocks: fffffffffffffffeffffffffffff0000
alloc = 63      lookahead:         ffffffff
                fs blocks: ffffffffffffffffffffffffffff0000
scanning...     lookahead:         ffffffff
                fs blocks: ffffffffffffffffffffffffffff0000
scanning...     lookahead:                 ffffffff
                fs blocks: ffffffffffffffffffffffffffff0000
scanning...     lookahead:                         ffff0000
                fs blocks: ffffffffffffffffffffffffffff0000
alloc = 112     lookahead:                         ffff8000
                fs blocks: ffffffffffffffffffffffffffff8000
```

While this lookahead approach still has an asymptotic runtime of O(n^2) to
scan all of storage, the lookahead reduces the practical runtime to a
reasonable amount. Bit-vectors are surprisingly compact, given only 16 bytes,
the lookahead could track 128 blocks. For a 4Mbyte flash chip with 4Kbyte
blocks, the littlefs would only need 8 passes to scan the entire storage.

The real benefit of this approach is just how much it simplified the design
of the littlefs. Deallocating blocks is as simple as simply forgetting they
exist, and there is absolutely no concern of bugs in the deallocation code
causing difficult to detect memory leaks.

## Directories

Now we just need directories to store our files. Since we already have
metadata blocks that store information about files, lets just use these
metadata blocks as the directories. Maybe turn the directories into linked
lists of metadata blocks so it isn't limited by the number of files that fit
in a single block. Add entries that represent other nested directories.
Drop "." and ".." entries, cause who needs them. Dust off our hands and
we now have a directory tree.

```
            .--------.
            |root dir|
            | pair 0 |
            |        |
            '--------'
            .-'    '-------------------------.
           v                                  v
      .--------.        .--------.        .--------.
      | dir A  |------->| dir A  |        | dir B  |
      | pair 0 |        | pair 1 |        | pair 0 |
      |        |        |        |        |        |
      '--------'        '--------'        '--------'
      .-'    '-.            |             .-'    '-.
     v          v           v            v          v
.--------.  .--------.  .--------.  .--------.  .--------.
| file C |  | file D |  | file E |  | file F |  | file G |
|        |  |        |  |        |  |        |  |        |
|        |  |        |  |        |  |        |  |        |
'--------'  '--------'  '--------'  '--------'  '--------'
```

Unfortunately it turns out it's not that simple. See, iterating over a
directory tree isn't actually all that easy, especially when you're trying
to fit in a bounded amount of RAM, which rules out any recursive solution.
And since our block allocator involves iterating over the entire filesystem
tree, possibly multiple times in a single allocation, iteration needs to be
efficient.

So, as a solution, the littlefs adopted a sort of threaded tree. Each
directory not only contains pointers to all of its children, but also a
pointer to the next directory. These pointers create a linked-list that
is threaded through all of the directories in the filesystem. Since we
only use this linked list to check for existence, the order doesn't actually
matter. As an added plus, we can repurpose the pointer for the individual
directory linked-lists and avoid using any additional space.

```
            .--------.
            |root dir|-.
            | pair 0 | |
   .--------|        |-'
   |        '--------'
   |        .-'    '-------------------------.
   |       v                                  v
   |  .--------.        .--------.        .--------.
   '->| dir A  |------->| dir A  |------->| dir B  |
      | pair 0 |        | pair 1 |        | pair 0 |
      |        |        |        |        |        |
      '--------'        '--------'        '--------'
      .-'    '-.            |             .-'    '-.
     v          v           v            v          v
.--------.  .--------.  .--------.  .--------.  .--------.
| file C |  | file D |  | file E |  | file F |  | file G |
|        |  |        |  |        |  |        |  |        |
|        |  |        |  |        |  |        |  |        |
'--------'  '--------'  '--------'  '--------'  '--------'
```

This threaded tree approach does come with a few tradeoffs. Now, anytime we
want to manipulate the directory tree, we find ourselves having to update two
pointers instead of one. For anyone familiar with creating atomic data
structures this should set off a whole bunch of red flags.

But unlike the data structure guys, we can update a whole block atomically! So
as long as we're really careful (and cheat a little bit), we can still
manipulate the directory tree in a way that is resilient to power loss.

Consider how we might add a new directory. Since both pointers that reference
it can come from the same directory, we only need a single atomic update to
finagle the directory into the filesystem:
```
   .--------.
   |root dir|-.
   | pair 0 | |
.--|        |-'
|  '--------'
|      |
|      v
|  .--------.
'->| dir A  |
   | pair 0 |
   |        |
   '--------'

|  create the new directory block
v

               .--------.
               |root dir|-.
               | pair 0 | |
            .--|        |-'
            |  '--------'
            |      |
            |      v
            |  .--------.
.--------.  '->| dir A  |
| dir B  |---->| pair 0 |
| pair 0 |     |        |
|        |     '--------'
'--------'

|  update root to point to directory B
v

         .--------.
         |root dir|-.
         | pair 0 | |
.--------|        |-'
|        '--------'
|        .-'    '-.
|       v          v
|  .--------.  .--------.
'->| dir B  |->| dir A  |
   | pair 0 |  | pair 0 |
   |        |  |        |
   '--------'  '--------'
```

Note that even though directory B was added after directory A, we insert
directory B before directory A in the linked-list because it is convenient.

Now how about removal:
```
         .--------.        .--------.
         |root dir|------->|root dir|-.
         | pair 0 |        | pair 1 | |
.--------|        |--------|        |-'
|        '--------'        '--------'
|        .-'    '-.            |
|       v          v           v
|  .--------.  .--------.  .--------.
'->| dir A  |->| dir B  |->| dir C  |
   | pair 0 |  | pair 0 |  | pair 0 |
   |        |  |        |  |        |
   '--------'  '--------'  '--------'

|  update root to no longer contain directory B
v

   .--------.              .--------.
   |root dir|------------->|root dir|-.
   | pair 0 |              | pair 1 | |
.--|        |--------------|        |-'
|  '--------'              '--------'
|      |                       |
|      v                       v
|  .--------.  .--------.  .--------.
'->| dir A  |->| dir B  |->| dir C  |
   | pair 0 |  | pair 0 |  | pair 0 |
   |        |  |        |  |        |
   '--------'  '--------'  '--------'

|  remove directory B from the linked-list
v

   .--------.  .--------.
   |root dir|->|root dir|-.
   | pair 0 |  | pair 1 | |
.--|        |--|        |-'
|  '--------'  '--------'
|      |           |
|      v           v
|  .--------.  .--------.
'->| dir A  |->| dir C  |
   | pair 0 |  | pair 0 |
   |        |  |        |
   '--------'  '--------'
```

Wait, wait, wait, that's not atomic at all! If power is lost after removing
directory B from the root, directory B is still in the linked-list. We've
just created a memory leak!

And to be honest, I don't have a clever solution for this case. As a
side-effect of using multiple pointers in the threaded tree, the littlefs
can end up with orphan blocks that have no parents and should have been
removed.

To keep these orphan blocks from becoming a problem, the littlefs has a
deorphan step that simply iterates through every directory in the linked-list
and checks it against every directory entry in the filesystem to see if it
has a parent. The deorphan step occurs on the first block allocation after
boot, so orphans should never cause the littlefs to run out of storage
prematurely. Note that the deorphan step never needs to run in a read-only
filesystem.

## The move problem

Now we have a real problem. How do we move things between directories while
remaining power resilient? Even looking at the problem from a high level,
it seems impossible. We can update directory blocks atomically, but atomically
updating two independent directory blocks is not an atomic operation.

Here's the steps the filesystem may go through to move a directory:
```
         .--------.
         |root dir|-.
         | pair 0 | |
.--------|        |-'
|        '--------'
|        .-'    '-.
|       v          v
|  .--------.  .--------.
'->| dir A  |->| dir B  |
   | pair 0 |  | pair 0 |
   |        |  |        |
   '--------'  '--------'

|  update directory B to point to directory A
v

         .--------.
         |root dir|-.
         | pair 0 | |
.--------|        |-'
|        '--------'
|    .-----'    '-.
|    |             v
|    |           .--------.
|    |        .->| dir B  |
|    |        |  | pair 0 |
|    |        |  |        |
|    |        |  '--------'
|    |     .-------'
|    v    v   |
|  .--------. |
'->| dir A  |-'
   | pair 0 |
   |        |
   '--------'

|  update root to no longer contain directory A
v
     .--------.
     |root dir|-.
     | pair 0 | |
.----|        |-'
|    '--------'
|        |
|        v
|    .--------.
| .->| dir B  |
| |  | pair 0 |
| '--|        |-.
|    '--------' |
|        |      |
|        v      |
|    .--------. |
'--->| dir A  |-'
     | pair 0 |
     |        |
     '--------'
```

We can leave any orphans up to the deorphan step to collect, but that doesn't
help the case where dir A has both dir B and the root dir as parents if we
lose power inconveniently.

Initially, you might think this is fine. Dir A _might_ end up with two parents,
but the filesystem will still work as intended. But then this raises the
question of what do we do when the dir A wears out? For other directory blocks
we can update the parent pointer, but for a dir with two parents we would need
work out how to update both parents. And the check for multiple parents would
need to be carried out for every directory, even if the directory has never
been moved.

It also presents a bad user-experience, since the condition of ending up with
two parents is rare, it's unlikely user-level code will be prepared. Just think
about how a user would recover from a multi-parented directory. They can't just
remove one directory, since remove would report the directory as "not empty".

Other atomic filesystems simple COW the entire directory tree. But this
introduces a significant bit of complexity, which leads to code size, along
with a surprisingly expensive runtime cost during what most users assume is
a single pointer update.

Another option is to update the directory block we're moving from to point
to the destination with a sort of predicate that we have moved if the
destination exists. Unfortunately, the omnipresent concern of wear could
cause any of these directory entries to change blocks, and changing the
entry size before a move introduces complications if it spills out of
the current directory block.

So how do we go about moving a directory atomically?

We rely on the improbableness of power loss.

Power loss during a move is certainly possible, but it's actually relatively
rare. Unless a device is writing to a filesystem constantly, it's unlikely that
a power loss will occur during filesystem activity. We still need to handle
the condition, but runtime during a power loss takes a back seat to the runtime
during normal operations.

So what littlefs does is inelegantly simple. When littlefs moves a file, it
marks the file as "moving". This is stored as a single bit in the directory
entry and doesn't take up much space. Then littlefs moves the directory,
finishing with the complete remove of the "moving" directory entry.

```
         .--------.
         |root dir|-.
         | pair 0 | |
.--------|        |-'
|        '--------'
|        .-'    '-.
|       v          v
|  .--------.  .--------.
'->| dir A  |->| dir B  |
   | pair 0 |  | pair 0 |
   |        |  |        |
   '--------'  '--------'

|  update root directory to mark directory A as moving
v

        .----------.
        |root dir  |-.
        | pair 0   | |
.-------| moving A!|-'
|       '----------'
|        .-'    '-.
|       v          v
|  .--------.  .--------.
'->| dir A  |->| dir B  |
   | pair 0 |  | pair 0 |
   |        |  |        |
   '--------'  '--------'

|  update directory B to point to directory A
v

        .----------.
        |root dir  |-.
        | pair 0   | |
.-------| moving A!|-'
|       '----------'
|    .-----'    '-.
|    |             v
|    |           .--------.
|    |        .->| dir B  |
|    |        |  | pair 0 |
|    |        |  |        |
|    |        |  '--------'
|    |     .-------'
|    v    v   |
|  .--------. |
'->| dir A  |-'
   | pair 0 |
   |        |
   '--------'

|  update root to no longer contain directory A
v
     .--------.
     |root dir|-.
     | pair 0 | |
.----|        |-'
|    '--------'
|        |
|        v
|    .--------.
| .->| dir B  |
| |  | pair 0 |
| '--|        |-.
|    '--------' |
|        |      |
|        v      |
|    .--------. |
'--->| dir A  |-'
     | pair 0 |
     |        |
     '--------'
```

Now, if we run into a directory entry that has been marked as "moved", one
of two things is possible. Either the directory entry exists elsewhere in the
filesystem, or it doesn't. This is a O(n) operation, but only occurs in the
unlikely case we lost power during a move.

And we can easily fix the "moved" directory entry. Since we're already scanning
the filesystem during the deorphan step, we can also check for moved entries.
If we find one, we either remove the "moved" marking or remove the whole entry
if it exists elsewhere in the filesystem.

## Wear awareness

So now that we have all of the pieces of a filesystem, we can look at a more
subtle attribute of embedded storage: The wear down of flash blocks.

The first concern for the littlefs, is that perfectly valid blocks can suddenly
become unusable. As a nice side-effect of using a COW data-structure for files,
we can simply move on to a different block when a file write fails. All
modifications to files are performed in copies, so we will only replace the
old file when we are sure none of the new file has errors. Directories, on
the other hand, need a different strategy.

The solution to directory corruption in the littlefs relies on the redundant
nature of the metadata pairs. If an error is detected during a write to one
of the metadata pairs, we seek out a new block to take its place. Once we find
a block without errors, we iterate through the directory tree, updating any
references to the corrupted metadata pair to point to the new metadata block.
Just like when we remove directories, we can lose power during this operation
and end up with a desynchronized metadata pair in our filesystem. And just like
when we remove directories, we leave the possibility of a desynchronized
metadata pair up to the deorphan step to clean up.

Here's what encountering a directory error may look like with all of
the directories and directory pointers fully expanded:
```
         root dir
         block 1   block 2
       .---------.---------.
       | rev: 1  | rev: 0  |--.
       |         |         |-.|
.------|         |         |-|'
|.-----|         |         |-'
||     '---------'---------'
||       |||||'--------------------------------------------------.
||       ||||'-----------------------------------------.         |
||       |||'-----------------------------.            |         |
||       ||'--------------------.         |            |         |
||       |'-------.             |         |            |         |
||       v         v            v         v            v         v
||    dir A                  dir B                  dir C
||    block 3   block 4      block 5   block 6      block 7   block 8
||  .---------.---------.  .---------.---------.  .---------.---------.
|'->| rev: 1  | rev: 0  |->| rev: 1  | rev: 0  |->| rev: 1  | rev: 0  |
'-->|         |         |->|         |         |->|         |         |
    |         |         |  |         |         |  |
    |         |         |  |         |         |  |         |         |
    '---------'---------'  '---------'---------'  '---------'---------'

|  update directory B
v

         root dir
         block 1   block 2
       .---------.---------.
       | rev: 1  | rev: 0  |--.
       |         |         |-.|
.------|         |         |-|'
|.-----|         |         |-'
||     '---------'---------'
||       |||||'--------------------------------------------------.
||       ||||'-----------------------------------------.         |
||       |||'-----------------------------.            |         |
||       ||'--------------------.         |            |         |
||       |'-------.             |         |            |         |
||       v         v            v         v            v         v
||    dir A                  dir B                  dir C
||    block 3   block 4      block 5   block 6      block 7   block 8
||  .---------.---------.  .---------.---------.  .---------.---------.
|'->| rev: 1  | rev: 0  |->| rev: 1  | rev: 2  |->| rev: 1  | rev: 0  |
'-->|         |         |->|         | corrupt!|->|         |         |
    |         |         |  |         | corrupt!|  |         |         |
    |         |         |  |         | corrupt!|  |         |         |
    '---------'---------'  '---------'---------'  '---------'---------'

|  oh no! corruption detected
v  allocate a replacement block

         root dir
         block 1   block 2
       .---------.---------.
       | rev: 1  | rev: 0  |--.
       |         |         |-.|
.------|         |         |-|'
|.-----|         |         |-'
||     '---------'---------'
||       |||||'----------------------------------------------------.
||       ||||'-------------------------------------------.         |
||       |||'-----------------------------.              |         |
||       ||'--------------------.         |              |         |
||       |'-------.             |         |              |         |
||       v         v            v         v              v         v
||    dir A                  dir B                    dir C
||    block 3   block 4      block 5   block 6        block 7   block 8
||  .---------.---------.  .---------.---------.    .---------.---------.
|'->| rev: 1  | rev: 0  |->| rev: 1  | rev: 2  |--->| rev: 1  | rev: 0  |
'-->|         |         |->|         | corrupt!|--->|         |         |
    |         |         |  |         | corrupt!| .->|         |         |
    |         |         |  |         | corrupt!| |  |         |         |
    '---------'---------'  '---------'---------' |  '---------'---------'
                                       block 9   |
                                     .---------. |
                                     | rev: 2  |-'
                                     |         |
                                     |         |
                                     |         |
                                     '---------'

|  update root directory to contain block 9
v

        root dir
        block 1   block 2
      .---------.---------.
      | rev: 1  | rev: 2  |--.
      |         |         |-.|
.-----|         |         |-|'
|.----|         |         |-'
||    '---------'---------'
||       .--------'||||'----------------------------------------------.
||       |         |||'-------------------------------------.         |
||       |         ||'-----------------------.              |         |
||       |         |'------------.           |              |         |
||       |         |             |           |              |         |
||       v         v             v           v              v         v
||    dir A                   dir B                      dir C
||    block 3   block 4       block 5     block 9        block 7   block 8
||  .---------.---------.   .---------. .---------.    .---------.---------.
|'->| rev: 1  | rev: 0  |-->| rev: 1  |-| rev: 2  |--->| rev: 1  | rev: 0  |
'-->|         |         |-. |         | |         |--->|         |         |
    |         |         | | |         | |         | .->|         |         |
    |         |         | | |         | |         | |  |         |         |
    '---------'---------' | '---------' '---------' |  '---------'---------'
                          |               block 6   |
                          |             .---------. |
                          '------------>| rev: 2  |-'
                                        | corrupt!|
                                        | corrupt!|
                                        | corrupt!|
                                        '---------'

|  remove corrupted block from linked-list
v

        root dir
        block 1   block 2
      .---------.---------.
      | rev: 1  | rev: 2  |--.
      |         |         |-.|
.-----|         |         |-|'
|.----|         |         |-'
||    '---------'---------'
||       .--------'||||'-----------------------------------------.
||       |         |||'--------------------------------.         |
||       |         ||'--------------------.            |         |
||       |         |'-----------.         |            |         |
||       |         |            |         |            |         |
||       v         v            v         v            v         v
||    dir A                  dir B                  dir C
||    block 3   block 4      block 5   block 9      block 7   block 8
||  .---------.---------.  .---------.---------.  .---------.---------.
|'->| rev: 1  | rev: 2  |->| rev: 1  | rev: 2  |->| rev: 1  | rev: 0  |
'-->|         |         |->|         |         |->|         |         |
    |         |         |  |         |         |  |         |         |
    |         |         |  |         |         |  |         |         |
    '---------'---------'  '---------'---------'  '---------'---------'
```

Also one question I've been getting is, what about the root directory?
It can't move so wouldn't the filesystem die as soon as the root blocks
develop errors? And you would be correct. So instead of storing the root
in the first few blocks of the storage, the root is actually pointed to
by the superblock. The superblock contains a few bits of static data, but
outside of when the filesystem is formatted, it is only updated when the root
develops errors and needs to be moved.

## Wear leveling

The second concern for the littlefs is that blocks in the filesystem may wear
unevenly. In this situation, a filesystem may meet an early demise where
there are no more non-corrupted blocks that aren't in use. It's common to
have files that were written once and left unmodified, wasting the potential
erase cycles of the blocks it sits on.

Wear leveling is a term that describes distributing block writes evenly to
avoid the early termination of a flash part. There are typically two levels
of wear leveling:
1. Dynamic wear leveling - Wear is distributed evenly across all **dynamic**
   blocks. Usually this is accomplished by simply choosing the unused block
   with the lowest amount of wear. Note this does not solve the problem of
   static data.
2. Static wear leveling - Wear is distributed evenly across all **dynamic**
   and **static** blocks. Unmodified blocks may be evicted for new block
   writes. This does handle the problem of static data but may lead to
   wear amplification.

In littlefs's case, it's possible to use the revision count on metadata pairs
to approximate the wear of a metadata block. And combined with the COW nature
of files, littlefs could provide your usual implementation of dynamic wear
leveling.

However, the littlefs does not. This is for a few reasons. Most notably, even
if the littlefs did implement dynamic wear leveling, this would still not
handle the case of write-once files, and near the end of the lifetime of a
flash device, you would likely end up with uneven wear on the blocks anyways.

As a flash device reaches the end of its life, the metadata blocks will
naturally be the first to go since they are updated most often. In this
situation, the littlefs is designed to simply move on to another set of
metadata blocks. This travelling means that at the end of a flash device's
life, the filesystem will have worn the device down nearly as evenly as the
usual dynamic wear leveling could. More aggressive wear leveling would come
with a code-size cost for marginal benefit.


One important takeaway to note, if your storage stack uses highly sensitive
storage such as NAND flash, static wear leveling is the only valid solution.
In most cases you are going to be better off using a full [flash translation layer (FTL)](https://en.wikipedia.org/wiki/Flash_translation_layer).
NAND flash already has many limitations that make it poorly suited for an
embedded system: low erase cycles, very large blocks, errors that can develop
even during reads, errors that can develop during writes of neighboring blocks.
Managing sensitive storage such as NAND flash is out of scope for the littlefs.
The littlefs does have some properties that may be beneficial on top of a FTL,
such as limiting the number of writes where possible, but if you have the
storage requirements that necessitate the need of NAND flash, you should have
the RAM to match and just use an FTL or flash filesystem.

## Summary

So, to summarize:

1. The littlefs is composed of directory blocks
2. Each directory is a linked-list of metadata pairs
3. These metadata pairs can be updated atomically by alternating which
   metadata block is active
4. Directory blocks contain either references to other directories or files
5. Files are represented by copy-on-write CTZ skip-lists which support O(1)
   append and O(n log n) reading
6. Blocks are allocated by scanning the filesystem for used blocks in a
   fixed-size lookahead region that is stored in a bit-vector
7. To facilitate scanning the filesystem, all directories are part of a
   linked-list that is threaded through the entire filesystem
8. If a block develops an error, the littlefs allocates a new block, and
   moves the data and references of the old block to the new.
9. Any case where an atomic operation is not possible, mistakes are resolved
   by a deorphan step that occurs on the first allocation after boot

That's the little filesystem. Thanks for reading!

