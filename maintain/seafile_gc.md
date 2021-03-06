# Seafile GC

Seafile uses storage de-duplication technology to reduce storage usage. The underlying data blocks will not be removed immediately after you delete a file or a library. As a result, the number of unused data blocks will increase on Seafile server.

To release the storage space occupied by unused blocks, you have to run a
"garbage collection" program to clean up unused blocks on your server.

The GC program cleans up two types of unused blocks:

1. Blocks that no library references to, that is, the blocks belong to deleted libraries;
2. If you set history length limit on some libraries, the out-dated blocks in those libraries will also be removed.

**Before running GC, you must shutdown the Seafile program on your server if you use the community edition. For professional edition, from version 3.1.11, online GC operation is supported. If you use Professional edition, you don't need to shutdown the Seafile program if you are using MySQL or PostgreSQL as database.**  This is because new blocks written into Seafile while GC is running may be mistakenly deleted by the GC program. 

## Run GC in version 4.1.1 and later

In community edition 4.1.1 and Pro edition 4.1.0, GC program's command line and output are improved.

### Dry-run Mode

To see how much garbage can be collected without actually removing any garbage, use the dry-run option:

```
seaf-gc.sh --dry-run [repo-id1] [repo-id2] ...
```

The output should look like:

```
[03/19/15 19:41:49] seafserv-gc.c(115): GC version 1 repo My Library(ffa57d93)
[03/19/15 19:41:49] gc-core.c(394): GC started. Total block number is 265.
[03/19/15 19:41:49] gc-core.c(75): GC index size is 1024 Byte.
[03/19/15 19:41:49] gc-core.c(408): Populating index.
[03/19/15 19:41:49] gc-core.c(262): Populating index for repo ffa57d93.
[03/19/15 19:41:49] gc-core.c(308): Traversed 5 commits, 265 blocks.
[03/19/15 19:41:49] gc-core.c(440): Scanning unused blocks.
[03/19/15 19:41:49] gc-core.c(472): GC finished. 265 blocks total, about 265 reachable blocks, 0 blocks can be removed.

[03/19/15 19:41:49] seafserv-gc.c(115): GC version 1 repo aa(f3d0a8d0)
[03/19/15 19:41:49] gc-core.c(394): GC started. Total block number is 5.
[03/19/15 19:41:49] gc-core.c(75): GC index size is 1024 Byte.
[03/19/15 19:41:49] gc-core.c(408): Populating index.
[03/19/15 19:41:49] gc-core.c(262): Populating index for repo f3d0a8d0.
[03/19/15 19:41:49] gc-core.c(308): Traversed 8 commits, 5 blocks.
[03/19/15 19:41:49] gc-core.c(264): Populating index for sub-repo 9217622a.
[03/19/15 19:41:49] gc-core.c(308): Traversed 4 commits, 4 blocks.
[03/19/15 19:41:49] gc-core.c(440): Scanning unused blocks.
[03/19/15 19:41:49] gc-core.c(472): GC finished. 5 blocks total, about 9 reachable blocks, 0 blocks can be removed.

[03/19/15 19:41:49] seafserv-gc.c(115): GC version 1 repo test2(e7d26d93)
[03/19/15 19:41:49] gc-core.c(394): GC started. Total block number is 507.
[03/19/15 19:41:49] gc-core.c(75): GC index size is 1024 Byte.
[03/19/15 19:41:49] gc-core.c(408): Populating index.
[03/19/15 19:41:49] gc-core.c(262): Populating index for repo e7d26d93.
[03/19/15 19:41:49] gc-core.c(308): Traversed 577 commits, 507 blocks.
[03/19/15 19:41:49] gc-core.c(440): Scanning unused blocks.
[03/19/15 19:41:49] gc-core.c(472): GC finished. 507 blocks total, about 507 reachable blocks, 0 blocks can be removed.

[03/19/15 19:41:50] seafserv-gc.c(124): === Repos deleted by users ===
[03/19/15 19:41:50] seafserv-gc.c(145): === GC is finished ===

[03/19/15 19:41:50] Following repos have blocks to be removed:
repo-id1
repo-id2
repo-id3
```

If you give specific library ids, only those libraries will be checked; otherwise all libraries will be checked.

Notice that at the end of the output there is a "repos have blocks to be removed" section. It contains the list of librareis that have garbage blocks. Later when you run GC without --dry-run option, you can use these libraris ids as input arguments to GC program.

### Removing Garbage

To actually remove garbage blocks, run without the --dry-run option:

```
seaf-gc.sh [repo-id1] [repo-id2] ...
```

If libraries ids are specified, only those libraries will be checked for garbage.

As described before, there are two types of garbage blocks to be removed. Sometimes just removing the first type (those belong to deleted libraries) of unused blocks is good enough. In this case, the GC program won't bother to check the libraries for outdated historic blocks. The "-r" option implements this feature:

```
seaf-gc.sh -r
```

**In Seafile version 4.1.1 and later, libraries deleted by the users are not immediately removed from the system. Instead, they're moved into a "trash" in the system admin page. Before they're cleared from the trash, their blocks won't be garbage collected.**

## Run GC in version 3.1.2 and later

To run GC program

    ./seaf-gc.sh run

If you want to do sanity check before actually removing any data, you can use the --dry-run option

    ./seaf-gc.sh dry-run

It will show you the total block number vs. the number of blocks to be removed.

To check data integrity after running GC, you can use [seaf-fsck](seafile_fsck.md)


## Run GC before version 3.1.2

To run GC program

    cd seafile-server-{version}/seafile
    export LD_LIBRARY_PATH=./lib:${LD_LIBRARY_PATH}
    ./bin/seafserv-gc -c ../../ccnet -d ../../seafile-data

If you built seafile server from source, just run

    seafserv-gc -c ../../ccnet -d ../../seafile-data

After the GC program terminates, you may also check whether it mistakenly removed any
useful data blocks:

    seafserv-gc -c ../../ccnet -d ../../seafile-data --verify

It will print a warning if any useful blocks are missing.

If you want to do sanity check before actually removing any data, you can use the --dry-run option

    seafserv-gc -c ../../ccnet -d ../../seafile-data --dry-run

It will show you the total block number vs. the number of blocks to be removed.

If the metadata of some libraries are damaged on the server, the GC program would stop proceeding because it can't tell whether a block is used by some damaged libraries or not. If you don't want to keep the data of the damaged library, you can run GC with --ignore-errors or -i option

    seafserv-gc -c ../../ccnet -d ../../seafile-data --ignore-errors

This will mask the blocks of the damaged libraries as unused and delete them.
