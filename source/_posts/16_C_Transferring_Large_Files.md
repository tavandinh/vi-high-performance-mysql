
```
APPENDIX C
```
```
Transferring Large Files
```
Copying, compressing, and decompressing huge files (often across a network) are
common tasks when administering MySQL, initializing servers, cloning replicas, and
performing backups and recovery operations. The fastest and best ways to do these
jobs are not always the most obvious, and the difference between good and bad meth-
ods can be significant. This appendix shows some examples of how to copy a large
backup image from one server to another using common Unix utilities.

It’s common to begin with an _uncompressed_ file, such as one server’s InnoDB tablespace
and log files. You also want the file to be decompressed when you finish copying it to
the destination, of course. The other common scenario is to begin with a _compressed_
file, such as a backup image, and finish with a decompressed file.

If you have limited network capacity, it’s usually a good idea to send the files across
the network in compressed form. You might also need to do a secure transfer, so your
data isn’t compromised; this is a common requirement for backup images.

### Copying Files

The task, then, is to do the following efficiently:

1. (Optionally) compress the data.
2. Send it to another machine.
3. Decompress the data into its final destination.
4. Verify the files aren’t corrupted after copying.

We’ve benchmarked various methods of achieving these goals. The rest of this appen-
dix shows you how we did it and what we found to be the fastest way.

For many of the purposes we’ve discussed in this book, such as backups, you might
want to consider which machine to do the compression on. If you have the network
bandwidth, you can copy your backup images uncompressed and save the CPU re-
sources on your MySQL server for queries.

```
715
```

#### A Naïve Example

We begin with a naïve example of how to send an uncompressed file securely from one
machine to another, compress it en route, and then decompress it. On the source server,
which we call server1, we execute the following:

```
server1$ gzip -c /backup/mydb/mytable.MYD > mytable.MYD.gz
server1$ scp mytable.MYD.gz root@server2:/var/lib/myql/mydb/
```
And then, on server2:

```
server2$ gunzip /var/lib/mysql/mydb/mytable.MYD.gz
```
This is probably the simplest approach, but it’s not very efficient because it serializes
the steps involved in compressing, copying, and decompressing the file. Each step also
requires reads from and writes to disk, which is slow. Here’s what really happens during
each of the above commands: the _gzip_ performs both reads and writes on server1, the
_scp_ reads on server1 and writes on server2, and the _gunzip_ reads and writes on server2.

#### A One-Step Method

It’s more efficient to compress and copy the file and then decompress it on the other
end in one step. This time we use SSH, the secure protocol upon which SCP is based.
Here’s the command we execute on server1:

```
server1$ gzip -c /backup/mydb/mytable.MYD | ssh root@server2"gunzip -c - > /var/lib
>/mysql/mydb/mytable.MYD"
```
This usually performs much better than the first method, because it significantly re-
duces disk I/O: the disk activity is reduced to reading on server1 and writing on
server2. This lets the disk operate sequentially.

You can also use SSH’s built-in compression to do this, but we’ve shown you how to
compress and decompress with pipes because they give you more flexibility. For ex-
ample, if you didn’t want to decompress the file on the other end, you wouldn’t want
to use SSH compression.

You can improve on this method by tweaking some options, such as adding- _1_ to make
the _gzip_ compression faster. This usually doesn’t lower the compression ratio much,
but it can make it much faster, which is important. You can also use different com-
pression algorithms. For example, if you want very high compression and don’t care
about how long it takes, you can use _bzip2_ instead of _gzip_. If you want very fast com-
pression, you can instead use an LZO-based archiver. The compressed data might be
about 20% larger, but the compression will be around five times faster.

#### Avoiding Encryption Overhead

SSH isn’t the fastest way to transport data across the network, because it adds the
overhead of encrypting and decrypting. If you don’t need encryption, you can just copy

**716 | Appendix C: Transferring Large Files**


the “raw” bits over the network with _netcat_. You invoke this tool as _nc_ for noninteractive
operations, which is what we want.

Here’s an example. First, let’s start listening for the file on port 12345 (any unused port
will do) on server2, and uncompress anything sent to that port to the desired data file:

```
server2$ nc -l -p 12345 | gunzip -c - > /var/lib/mysql/mydb/mytable.MYD
```
On server1, we then start another instance of _netcat_ , sending to the port on which the
destination is listening. The _-q_ option tells _netcat_ to close the connection after it sees
the end of the incoming file. This will cause the listening instance to close the destina-
tion file and quit:

```
server1$ gzip -c - /var/lib/mysql/mydb/mytable.MYD | nc -q 1 server2 12345
```
An even easier technique is to use _tar_ so filenames are sent across the wire, eliminating
another source of errors and automatically writing the files to their correct locations.
The _z_ option tells _tar_ to use _gzip_ compression and decompression. Here’s the command
to execute on server2:

```
server2$ nc -l -p 12345 | tar xvzf -
```
And here’s the command for server1:

```
server1$ tar cvzf - /var/lib/mysql/mydb/mytable.MYD | nc -q 1 server2 12345
```
You can assemble these commands into a single script that will compress and copy lots
of files into the network connection efficiently, then decompress them on the other side.

#### Other Options

Another option is _rsync. rsync_ is convenient because it makes it easy to mirror the source
and destination and because it can restart interrupted file transfers, but it doesn’t tend
to work as well when its binary difference algorithm can’t be put to good use. You
might consider using it for cases where you know most of the file doesn’t need to be
sent—for example, for finishing an aborted _nc_ copy operation.

You should experiment with file copying when you’re not in a crisis situation, because
it will take a little trial and error to discover the fastest method. Which method performs
best will depend on your system. The biggest factors are how many disk drives, network
cards, and CPUs you have, and how fast they are relative to each other. It’s a good idea
to monitor _vmstat -n 5_ to see whether the disk or the CPU is the speed bottleneck.

If you have idle CPUs, you can probably speed up the process by running several copy
operations in parallel. Conversely, if the CPU is the bottleneck and you have lots of
disk and network capacity, omit the compression. As with dumping and restoring, it’s
often a good idea to do these operations in parallel for speed. Again, monitor your
servers’ performance to see if you have unused capacity. Trying to overparallelize might
just slow things down.

```
Copying Files | 717
```

### File Copy Benchmarks

For the sake of comparison, Table C-1 shows how quickly we were able to copy a
sample file over a standard 100 Mbps Ethernet link on a LAN. The file was 738 MB
uncompressed and compressed to 100 MB with _gzip_ ’s default options. The source and
destination machines had plenty of available memory, CPU resources, and disk ca-
pacity; the network was the bottleneck.

_Table C-1. Benchmarks for copying files across a network_

```
Method Time (seconds)
rsync without compression 71
scp without compression 68
nc without compression 67
rsync with compression (-z) 63
gzip, scp, and gunzip 60 (44 + 10 + 6)
ssh with compression 44
nc with compression 42
```
Notice how much it helped to compress the file when sending it across the network—
the three slowest methods didn’t compress the file. Your mileage will vary, however.
If you have slow CPUs and disks and a gigabit Ethernet connection, reading and
compressing the data might be the bottleneck, and it might be faster to skip the
compression.

By the way, it’s often much faster to use fast compression, such as _gzip --fast_ , than to
use the default compression levels, which use a lot of CPU time to compress the file
only slightly more. Our test used the default compression level.

The last step in transferring data is to verify that the copy didn’t corrupt the files. You
can use a variety of methods for this, such as _md5sum_ , but it’s rather expensive to do
a full scan of the file again. This is another reason why compression is helpful: the
compression itself typically includes at least a cyclic redundancy check (CRC), which
should catch any errors, so you get error checking for free.

**718 | Appendix C: Transferring Large Files**
