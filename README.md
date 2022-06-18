#  disk-data-scrambler - Insecurely but quickly destroy some of the data on a hard drive

Use at your own risk! The code here presented is still **untested**.

The initial code has been posted from [Aleksandr Levchuk](https://serverfault.com/users/58336/aleksandr-levchuk) on StackExchange [can be found clicking here](https://serverfault.com/a/1001381).

A Python 3 script for coarsely (not securely! Most of the data will remain untouched!) destroy the data on a partition or a device, with:

* awareness of block device size
* verification of block device size, by writing bytes at the boundaries
* guarantee of first and last byte destroy
* random data overwrites
* random offset

It has been designed for old mechanical hard drives in the case that a proper wipe would take too long.

If you need Python 2 then it should be easy to covert by using "".format() instead of f"" strings.

## Is this tool for you?

**Most likely not. There are many other much better tools which _actually_ destroy all the data!**

If you want to destroy the data on a hard drive, think about the following:

* if you don't need the drive anymore, consider a mechanical destruction (e.g. open the case and hammer the disks)
* use a secure secure disk wiping utility (e.g. dd writing the whole disk with random data), that will make the data recovery quite impossible, check [this page for suggestions](https://wiki.archlinux.org/title/Securely_wipe_disk)
* if you can't wait for a proper wipe to be completed (e.g. the drive is slow or damaged or you're in a hurry) AND if you're fine with data being possible to recover (it is very likely that **most of your files will not be even touched by this quick tool**) in an easy way (it's very easy, for example, using [PhotoRec](https://www.cgsecurity.org/wiki/PhotoRec) which is open source and can easily recover also deleted files, just try it), but you just want to make the data recovery a bit harder, you can use the code reported here.

## Preparation

* Make sure to **properly shred at least the important files**, for example the Firefox passwords storage and the shadow file (in the example, the partition to destroy has been mounted on `/mntYourDiskToDESTROY`):

```
shred /mntYourDiskToDESTROY/home/*/.mozilla/firefox/*/key3.db
shred /mntYourDiskToDESTROY/home/*/.mozilla/firefox/*/logins.json
shred /mntYourDiskToDESTROY/etc/shadow
```

* Damage all the non-deleted files. The code reported here goes around the disk writing rubbish with the only purpose to destroy also the traces of the deleted files, but it is very likely that most of your disk is empty and that most of this shredding is targetting useless areas. A command like this should write one kB of rubbish at the beginning of each file in the partition you mounted on `/mntYourDiskToDESTROY`:

```
find /mntYourDiskToDESTROY -type f -exec dd if=/dev/zero bs=1024 count=1 status=noxfer of="{}" \;
```

* Unmount the partition /mntYourDiskToDESTROY.
* Format the partitions you want to destroy. It is not going to destroy the data but at least you make sure that the easily accessible index to the files names is not as easily accessible anymore:

```
mkfs.ext2 /dev/sdYourDiskToDESTROY1
mkfs.ext2 /dev/sdYourDiskToDESTROY2
...
```

## Code to copy and paste

Please note that this is designed for working on a drive or partition not currently in use, so if you have opened it, unmount it before running the following code.

Make sure you understand the code for **avoiding deleting the wrong thing!**

***Use at your own risk! This will randomly damage some of your data but does not destroy all of it!***

```
import subprocess
import random
import os
from hurry.filesize import size

REPS = 100
CRASH_ON_FIRST_FAILURE = True  # change to False if the device is not reliable

def run(*cmd, assert_returncode=None, print_stderr=False):
    completed = subprocess.run(cmd, capture_output=True)
    if assert_returncode:
        assert completed.returncode == assert_returncode, f"Unexpected return code (got={assert_returncode}, expected={assert_returncode})"
    if print_stderr:
        print(str(completed.stderr))
    return completed.stdout


def target_identify(target):
    """
    Partition or large file?
    """
    target_is_partition = stat.S_ISBLK(os.stat(target).st_mode): # detect if target is a block device, taken from https://www.systutorials.com/how-to-check-whether-a-file-of-a-given-path-is-a-block-device-in-python/
    target_is_file = os.stat.isfile(target):
    # ^ is xor
    assert target_is_partition ^ target_is_file,
        "The indicated target is neither a block device (partition or whole device) nor a file. Make sure you know what you're doing!"
    if target_is_partition:
        target_size = part_size(target)
    else:
        target_size = file_size(target)

    return target_is_partition, target_size


def part_size(target):
    """
    Partition or whole device size in bytes
    """
    size = int(run("blockdev", "--getsize64", target))
    return size


def file_size(target)
    """
    Large file size in bytes
    """
    size = os.path.getsize(target)
    return size


def destroy_block(target, bs, seek, assert_returncode=None, print_result=False):
    """
    Executes dd taking /dev/urandom in input as source of pseudo-random data
    """
    run(
        # conv=notrunc is used when writing on large files, in order to edit the file in the indicated position without setting the end of the file just after that point
        # if you think that urandom random data generator is the bottleneck, you can try with OpenSSL, check out here: https://unix.stackexchange.com/questions/72216/fast-way-to-randomize-hd but really, if the disk write is not the limiting factor, why are you using this tool? Just wipe the whole disk!
        "dd", f"bs={bs}", "if=/dev/urandom", f"of={target}", f"seek={seek}", "count=1", "conv=notrunc",
        assert_returncode=assert_returncode
    )
    if print_result:
        print(f"Destroyed bs={bs} of={target} seek={seek}")


def get_random_seek(target_size, bs):
    seek_max = int(target_size / bs)
    seek = random.randint(0, seek_max)
    return seek


def generate_seek_set(target_size, bs):
    seek_set = {} # using a set instead of a list for avoiding duplicates and having it already sorted
    for _ in range(REPS):
        seek_set.add(get_random_seek(target_size=target_size, bs=bs))
    return seek_set


def destroy_set_blocks(target, bs, seek_set):
    """
    seek_set - set of elements indicating position of blocks to be destroyed
    """
    if CRASH_ON_FIRST_FAILURE:
        assert_returncode = 0
    else:
        assert_returncode = None
    for seek in seek_set:
        destroy_block(target=target, bs=bs, seek=seek, assert_returncode=assert_returncode)


def destroy(target):
    """
    target - partition or large file to be destroyed
    """
    bs = os.stat(".").st_blksize # use the block size suggested by the kernel (usually 4096 bytes) for the filesystem currently in use, very likely this value requires optimization. Could be obtained via "blockdev --getbsz"
    target_is_partition, target_size = target_identify(target)
    target_size_human = size(target_size)
    confirmation = input(f"The data in {target} with size {target_size_human} will be damaged, including the partitions table if present. Type YES if you are willing to proceed: ")
    assert confirmation == "YES", "Exiting"
    target_size_in_blocks = int(target_size / bs)
    if target_is_partition:
        destroy_block(target=target, bs=bs, seek=target_size_in_blocks, assert_returncode=1)  # "test" destroying 1 block at size boundary, should fail. When writing on a file it would not fail.
    destroy_block(target=target, bs=bs, seek=(target_size_in_blocks - 1), assert_returncode=0, print_result=True)  # "test" destroying 1 block before boundary, should pass
    os.sync()
    destroy_block(target=target, bs=bs, seek=0, assert_returncode=0, print_result=True)  # "test" destroying first 1 block, deleting the first copy of the partitions table if present
    os.sync()
    blocks_done = 0
    while True:
        seek_set = generate_seek_set(target_size=target_size, bs=bs)
        seek_set_len = len(seek_set)
        blocks_done = blocks_done + seek_set_len*(1 - blocks_done/target_size_in_blocks) # the factor in parenthesis accounts for the probability of destroying blocks that have been already destroyed in previous rounds
        percentage_done = 100 * blocks_done / target_size_in_blocks
        print(f"Destroying {seek_set_len} x {bs} bytes sized blocks, {percentage_done:.3} %")
        destroy_set_blocks(target=target, bs=bs, seek_set=seek_set)
        os.sync()

```


## Example output

```
# "test" destroying 1 byte at size boundary, should fail
# "test" destroying 1 bytes before boundary, should pass
Destroyed bs=1 of=/dev/sdb1 seek=10736369663
# "test" destroying first 1 byte
Destroyed bs=1 of=/dev/sdb1 seek=0

Destroying 100 4096 bytes sized blocks
Destroying 100 4096 bytes sized blocks
Destroying 100 4096 bytes sized blocks

```


## To run

* Read again the "Is this tool for you?" section in this README
* Copy the source code
* On the victim host log-in as root
* On the victim host run: python3
* Paste the code
* Type `destroy("/dev/XYZ")` making extremely sure to not indicate the wrong partition/device
* Hit Enter
* After a few hours, hit Ctrl-c

NOTE: "/dev/XYZ" is the partition or device name that will LOSE ALL THE DATA.

WARNING: Watch out for Fat-finger errors. The data will be gone forever so **DOUBLE-CHECK what PARTITION or DEVICE you are typing in**.

WARNING: Running this script for days in cloud services may cost if you are charged for disk write IOPs.

***Use at your own risk! This will destroy your data but does not guarantee that it's 100% destroyed!***

