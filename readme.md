# TTM2 Demo of Sleuthkit using disk.dd

## What are we working with?

### List the partitions on the disk

    $ mmls disk.dd

### Look in the unallocated partitions

    $ mmcat disk.dd 01 | strings
    $ mmcat disk.dd 03 | strings

Partition `02` starts at sector `0000002048` and is described as `Linux (0x83)`

### Extract the main partition to a body file
Lets extract the allocated partition so that we can work on it without having to type the offset all the time. 
**WARNING** This takes a long time

    $ dd if=disk.dd of=body.dd bs=512 skip=2048 count=6834176

- `if` is the input file
- `of` is the output file
- `bs` is the input and output block size 
- `skip` denotes number of sector skipped
- `count` denotes number of sector to copy

### Get the NSRL known software hashes

**WARNING** I can't get this to work
First download the NSRL database

    $ wget http://www.nsrl.nist.gov/voting/20140608/NSRLFile.txt

If you are on OS X mavericks+, and don't have wget, you can download the file using

    $ curl "http://www.nsrl.nist.gov/voting/20140608/NSRLFile.txt" -o "NSRLFile.txt"

Then index the database for MD5 (see `-n nsrl_db` in `man sorter` and `man hfind`)

    $ hfind -i nsrl-md5 NSRLFile.txt 

### Find files with on the drive
First lets start the search from the `home` directory, so we need to find the `meta addr` of this directory:

    $ fls disk.dd -o 2048

We get that `585:   home` meaning that the meta addr we want is 585. Lets look at the different users home directories:

    $ fls disk.dd -o 2048 585
    d/d 4610:   void

We see that the only user is `void`. Lets see what he has in his home directory `/home/void/`

    $ fls -o 2048 disk.dd 4610
    ...
    d/d 38225:  .hacking
    ...

Whoa! Lets search for files in this directory with an extension missmatch.

    $ mkdir sorter_output
    $ sorter -e -n NSRLFile.txt -o 2048 -d sorter_output disk.dd 38225 2> /dev/null

- `-e` - look for files with an extention missmatch
- `-n NSRLFILE.txt` - ignore known software
- `-o 2048` - the disk partition offset we learned form mmls
- `-d sorter_output` - the directory to store the generated report
- `disk.dd 38225` - search in this disk image at the meta directory representing .hacking

Ok, so now we have the inode for the file, if we look in `sorter_output/mismatch.txt`

    D4_06_this_is_weird.pdf
      empty (Zip archive data, at least v2.0 to extract)  (Ext: pdf)
      Image: disk.dd  Inode: 38611
      MD5: a495b7b6b126c09db019bd2c84908f97

We can use the inode to print the contents of the file with `icat`

    $ icat -o 2048 disk.dd 38611 > file.zip

And to extract the data, we can simply do the following

    $ unzip file.zip -d file

**Oups!***, we require a password. Luckily we found one with `cat mmcat disk.dd 02`

# Demo of Autopsy and deleted files

