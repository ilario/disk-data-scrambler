
The original post from [Aleksandr Levchuk](https://serverfault.com/users/58336/aleksandr-levchuk) on StackExchange [can be found clicking here](https://serverfault.com/a/1001381).

A Python 3 script, with:

* awareness of block device size
* verification of block device size, by writing bytes at the boundaries
* guarantee of first and last byte destroy
* random data overwrites
* random offset
* random batch sizes

If you need Python 2 then it should be easy to covert by using "".format() instead of f"" strings.

```
import subprocess
import random
import os

REPS = 100
CRASH_ON_FIRST_FAILURE = True  # change to False if the device is not reliable

def run(*cmd, assert_returncode=None, print_stderr=False):
    completed = subprocess.run(cmd, capture_output=True)
    if assert_returncode:
        assert completed.returncode == assert_returncode, f"Unexpected return code (got={assert_returncode}, expected={assert_returncode})"
    if print_stderr:
        print(str(completed.stderr))
    return completed.stdout


def part_size(part):
    """
    Partition size in bytes
    """
    return int(run("blockdev", "--getsize64", part))


def destroy_block(part, bs, seek, assert_returncode=None, print_result=False):
    run(
        "dd", f"bs={bs}", "if=/dev/urandom", f"of={part}", f"seek={seek}", "count=1",
        assert_returncode=assert_returncode
    )
    if print_result:
        print(f"Destroyed bs={bs} of={part} seek={seek}")


def get_random_seek(part_size, bs):
    seek_max = int(part_size / bs)
    seek = random.randint(0, seek_max)
    return seek


def generate_seek_list(part_size, bs)
    seek_list = []
    for _ in range(REPS):
        seek_list.append(get_random_seek(part_size, bs))
    seek_list.sort()
    return seek_list


def destroy_list_blocks(part, bs, seek_list):
    if CRASH_ON_FIRST_FAILURE:
        assert_returncode = 0
    else:
        assert_returncode = None
    for seek in seek_list:
        destroy_block(part, bs=bs, seek=seek, assert_returncode=assert_returncode)


def destory_random_block(part, part_size, bs):
    """
    bs=1 destroys bytes sized block
    bs=1024 destroys KB sized block
    etc.
    """
    seek = get_random_seek(part_size, bs)
    if CRASH_ON_FIRST_FAILURE:
        assert_returncode = 0
    else:
        assert_returncode = None
    destroy_block(part, bs=bs, seek=seek, assert_returncode=assert_returncode)


def destroy(part):
    """
    part - partition to be destroyed
    """
    s = part_size(part)
    bs = os.stat(".").st_blksize # use the block size suggested by the kernel (usually 4096 bytes) for the filesystem currently in use
    destroy_block(part, bs=1, seek=s, assert_returncode=1)  # "test" destroying 1 byte at size boundary, should fail
    destroy_block(part, bs=1, seek=(s - 1), assert_returncode=0, print_result=True)  # "test" destroying 1 bytes before boundary, should pass
    destroy_block(part, bs=1, seek=0, assert_returncode=0, print_result=True)  # "test" destroying first 1 byte
    while True:
        print(f"Destroying {REPS} {bs} bytes sized blocks")
        seek_list = generate_seek_list(part_size, bs=bs)
        destory_list_blocks(part, bs=bs, seek_list)

```


Example output:

```
# "test" destroying 1 byte at size boundary, should fail
# "test" destroying 1 bytes before boundary, should pass
Destroyed bs=1 of=/dev/sdb1 seek=10736369663
# "test" destroying first 1 byte
Destroyed bs=1 of=/dev/sdb1 seek=0

Destroying 100 byte-sized blocks
Destroying 100 KB-sized blocks
Destroying 100 MB-sized blocks
Destroying 100 875091-sized blocks

Rise and repeat


Destroying 100 byte-sized blocks
Destroying 100 KB-sized blocks
Destroying 100 MB-sized blocks
Destroying 100 1028370-sized blocks

Rise and repeat
```

To run:

* Copy the source code all but the last line
* On the victim host log-in as root
* On the victim host run: python3
* Paste the code from step 1
* Type destroy("/dev/XYZ") and return key twice
* After a few hours, hit Ctrl-c

NOTE: "/dev/XYZ" is the partition name that will lose data.

WARNING: Watch out for Fat-finger errors. The data will be gone forever so double-check what partition you are typing in.

WARNING: Running this script for days in cloud services may cost if you are charged for disk write IOPs.

