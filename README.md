<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Introduction</a></li>
<li><a href="#sec-2">2. Installation and Configuration</a></li>
<li><a href="#sec-3">3. Examples</a>
<ul>
<li><a href="#sec-3-1">3.1. Mapping a number of pnfs filenames to dCache IDs and then to cache locations on fileservers</a></li>
<li><a href="#sec-3-2">3.2. Erasing cached-only files from a pool</a></li>
<li><a href="#sec-3-3">3.3. Finding and releasing hanging transfers</a></li>
<li><a href="#sec-3-4">3.4. Finding all active movers for WAN or local dcap accesses for a VO</a></li>
<li><a href="#sec-3-5">3.5. Finding a number of movers and selectively kill them based on which files they are accessing</a></li>
<li><a href="#sec-3-6">3.6. Consistency checks</a>
<ul>
<li><a href="#sec-3-6-1">3.6.1. Correct all the files in a pool with known error state</a></li>
<li><a href="#sec-3-6-2">3.6.2. Locate a pool's files with no pnfs entries</a></li>
<li><a href="#sec-3-6-3">3.6.3. Find files with no replicates</a></li>
</ul>
</li>
</ul>
</li>
</ul>
</div>
</div>


# Introduction

The Shellutils are a collection of shell scripts which help to
ineract with the dcache (<http://www.dcache.org/>) storage
manager. They work by just piping commands via ssh to the dcache
admin shell and then parse the output. They all use the same basic
bash library `dc_utils_lib.sh` to execute commands on the admin
shell and retrieve output, so it is very easy to add new commands.

LICENSE: GPL 

The shellutils make it easy to **execute commands on large lists** of
filenames or IDs, and to chain commands with pipes. Working with
them offers the same kind of possibilities as with typical unix
shell commands. Most tools take some list from a file or stdin as
input, and they will return a list on stdout, enabling them to be
chained in typical Unix style.

The tools also help to get acquainted with the internals of dcache
and they are nicer to handle than the cumbersome admin
shell. Looking at the scripts, one can see what admin shell commands
are used.

Note: All tools will display documentation when invoked with an `-h`
flag:

    dc_get_pool_movers.sh -h
    
    Synopsis:
          dc_get_pool_movers.sh [pool-listfile]
    Options:
          -b       :   beautify. Print only pnfsID and poolname. This can be directly
                       piped into commands working on the pnfs IDs
          -k           generate a list that can be filtered and piped to the dc_kill_pool_movers
                       command (includes pnfs filenames. may take slightly longer)
          -q queue :   list only movers for the named mover queue
          -d       :   debug. Show what commands are executed. The output will
                       be sent to stderr to not contaminate stdout.
    
    Description:
          Shows all movers of the respective pools. The listing matches exactly
          the output of 'movers ls' given in a pool cell, but with the name of
          the pool inserted at the beginning of the line.
          When no pool-listfile is given, the pool list is expected on stdin
    
          Note: querying a large number of pools can take some time
    
    Examples:
          dc_get_pool_movers.sh cmspools.lst
          cat cmspools.lst | dc_get_pool_movers.sh
          dc_get_pool_list.sh | dc_get_pool_movers.sh

# Installation and Configuration

First of all, you should deposit your public SSH key in the
appropriate `/opt/d-cache/config/authorized_keys` (dcache versions
< 2) or `/etc/dcache/admin/authorized_keys2` (dcache versions > 2)
file on the node running the admin shell service. **IMPORTANT**: The
`name tag` (last field of the key line) must be changed to `admin`
(seems no longer required in the latest versions)!  This will allow
you to connect without having to type a password every time.

Define the following three environment variables

-   `DCACHEADMINHOST`: hostname of admin interface
-   `DCACHEADMINPORT`: port number of admin interface
-   `DCACHE_SHELLUTILS`: directory to which you checked out the shellutils (needed, so the scripts can find the `dc_utils_lib.sh` library)

In addition you may also want to define

-   `DCACHEADMIN_KEY`: location of an ssh keyfile for accessing the dcache admin shell (if not found in the default location)
-   `DCACHE_VERSION`: dcache version (e.g. `2.2`). Newer versions of dcache allow SSH v2 access.

# Examples

## Mapping a number of pnfs filenames to dCache IDs and then to cache locations on fileservers

Put the filenames into a file `files-pnfs.lst` , one per line (you could also pipe the list directly into the dc<sub>\*</sub> commands):

    /pnfs/lcg.cscs.ch/cms/trivcat/store/phedex_monarctest/monarctest_CSCS-DISK1/LoadTest07_CSCS_FA
    /pnfs/lcg.cscs.ch/cms/trivcat/store/phedex_monarctest/monarctest_CSCS-DISK1/LoadTest07_CSCS_FB
    /pnfs/lcg.cscs.ch/cms/trivcat/store/phedex_monarctest/monarctest_CSCS-DISK1/LoadTest07_CSCS_FC
    /pnfs/lcg.cscs.ch/cms/trivcat/store/phedex_monarctest/monarctest_CSCS-DISK1/LoadTest07_CSCS_FE
    /pnfs/lcg.cscs.ch/cms/trivcat/store/phedex_monarctest/monarctest_CSCS-DISK1/LoadTest07_CSCS_FF

Then use the following command:

    $> dc_get_ID_from_pnfsnamelist.sh files-pnfs.lst
    000200000000000000048010 /pnfs/lcg.cscs.ch/cms/trivcat/store/phedex_monarctest/monarctest_CSCS-DISK1/LoadTest07_CSCS_FA
    00020000000000000004BE80 /pnfs/lcg.cscs.ch/cms/trivcat/store/phedex_monarctest/monarctest_CSCS-DISK1/LoadTest07_CSCS_FB
    00020000000000000004DE88 /pnfs/lcg.cscs.ch/cms/trivcat/store/phedex_monarctest/monarctest_CSCS-DISK1/LoadTest07_CSCS_FC
    000200000000000000053B38 /pnfs/lcg.cscs.ch/cms/trivcat/store/phedex_monarctest/monarctest_CSCS-DISK1/LoadTest07_CSCS_FE
    000200000000000000056238 /pnfs/lcg.cscs.ch/cms/trivcat/store/phedex_monarctest/monarctest_CSCS-DISK1/LoadTest07_CSCS_FF

We can use a pipe to get the cache locations from the previous command's output (the commands will ignore the second column of the input, so no need to cut the filename strings away)

    dc_get_ID_from_pnfsnamelist.sh files-pnfs.lst |dc_get_cacheinfo_from_IDlist.sh
    000200000000000000048010 se05_cms
    00020000000000000004BE80 se07_cms,se02_cms
    00020000000000000004DE88 se05_cms
    000200000000000000053B38 se05_cms,se06_cms
    000200000000000000056238 se06_cms

## Erasing cached-only files from a pool

The `dc_get_rep_ls.sh` command prints out the pnfs IDs of the files in a given pool. By adding the `-r` flag (raw) one can obtain the detailed property flags for each file. Other flags allow for the filtering of the entries, e.g. `-c` for cached-only files.

    dc_get_rep_ls.sh -r -c se05_cms > cachedfiles.lst
    
    # the output in the file contains lines like this
    00006D0A348BF472498D98DC2320368F1ABB <C----------L(0)[0]> 1929001991 si={cms:cms}                                      
    0000A1DF66B86A5D45D49183FF44414B5188 <C----------L(0)[0]> 1977244606 si={cms:cms}                                      
    0000FC22E489F79740A6ABCC3D4309AA7B67 <C----------L(0)[0]> 1930490535 si={cms:cms}                                      
    00002AB438A089B744A1B3EC981E5B379F7C <C----------L(0)[0]> 1989635935 si={cms:cms}                                      
    000021DC9CAA93AE41019B0C365B6FD93AD1 <C----------L(0)[0]> 1970407278 si={cms:cms}                                      
    00005DFEE404A0A24E0A99720123A52607CA <C----------L(0)[0]> 1970806686 si={cms:cms}
    ...
    The Shellutils can be checked out from the SVN at %SVNBASE%/d-cache/dcache-utilities/shellutils.

You may wish to double check whether indeed all these files have multiple copies on the cluster, by finding the cache locations of all files

    dc_get_cacheinfo_from_IDlist.sh cachedfiles.lst
    
    00006D0A348BF472498D98DC2320368F1ABB se05_cms,se30_cms                                                                 
    0000A1DF66B86A5D45D49183FF44414B5188 se05_cms,se21_cms,se33_cms                                                        
    0000FC22E489F79740A6ABCC3D4309AA7B67 se05_cms,se37_cms                                                                 
    00002AB438A089B744A1B3EC981E5B379F7C se05_cms,se36_cms
    ...

Now we remove the files from the pool by invoking the `dc_rep_rm_list.sh` command. The command will not remove pinned files, unless a `-f` force flag is given, so the operation is relatively safe.

    dc_rep_rm_list.sh se05_cms  cachedfiles.lst

## Finding and releasing hanging transfers

When a pool goes down or is overloaded, it may happen that transfers get into a hanging state. This can be seen on the [Tape Transfer Queue](http://storage01.lcg.cscs.ch:2288/poolInfo/restoreHandler/*) or [Detailed Tape Transfer Queue](http://storage01.lcg.cscs.ch:2288/poolInfo/restoreHandler/lazy) web pages. In the admin interface the information can be listed by typing `rc ls` in the `PoolManager` cell. A transfer can be retried by giving the ID obtained from this listing
to the `rc retry` command.

The shellutils provide `dc_get_pending_requests.sh` for listing the hanging requests.

    $> dc_get_pending_requests.sh
    000200000000000000D86628@0.0.0.0/0.0.0.0-*/* m=2 r=0 [<unknown>] [Suspended (pool unavailable) 05.16 09:37:36] {0,}
    000200000000000000D77E38@0.0.0.0/0.0.0.0-*/* m=1 r=0 [<unknown>] [Suspended (pool unavailable) 05.16 09:37:34] {0,}
    000200000000000000CE6530@0.0.0.0/0.0.0.0-*/* m=1 r=1 [<unknown>] [Suspended (pool unavailable) 05.16 09:34:07] {0,}
    ...

To retry all of these transfer, we can construct a chain with `dc_generic_cellcommand.sh`:

    $> dc_get_pending_requests.sh |cut -f1 -d' '|dc_generic_cellcommand.sh -f -c 'rc retry $n' PoolManager
    
    
    [storage01.lcg.cscs.ch] (local) admin > cd PoolManager
    [storage01.lcg.cscs.ch] (PoolManager) admin > rc retry 000200000000000000D86628@0.0.0.0/0.0.0.0-*/*
    [storage01.lcg.cscs.ch] (PoolManager) admin > rc retry 000200000000000000D77E38@0.0.0.0/0.0.0.0-*/*
    ...
    [storage01.lcg.cscs.ch] (PoolManager) admin > rc retry 000200000000000000D79E28@0.0.0.0/0.0.0.0-*/*
    [storage01.lcg.cscs.ch] (PoolManager) admin > ..
    [storage01.lcg.cscs.ch] (local) admin > logoff

Some of the transfers may remain hanging. These you can kill by `rc fail` using the same kind of construct

    $> dc_get_pending_requests.sh |cut -f1 -d' '|dc_generic_cellcommand.sh -d -f -c 'rc failed $n' PoolManager

You can list the cache locations and the names of the files using these commands

    $> dc_get_pending_requests.sh |cut -f1 -d'@'|dc_get_cacheinfo_from_IDlist.sh
    $> dc_get_pending_requests.sh |cut -f1 -d'@'|dc_get_pnfsname_from_IDlist.sh

## Finding all active movers for WAN or local dcap accesses for a VO

Look at the dc<sub>get</sub><sub>pool</sub><sub>movers</sub>.sh command. If we want to see all active dcap movers (default) queue:

    dc_get_pool_list.sh | grep cms | dc_get_pool_movers.sh -q default

## Finding a number of movers and selectively kill them based on which files they are accessing

`dc_get_pool_movers.sh` offers a `-k` flag which will produce output containing one more column with the pnfs mapped filename 

    ...
    t3fs04_cms 162961 W H {GFTP-t3fs07-Unknown-20760@gridftp-t3fs07Domain:22094}    000200000000000002D3A518 /pnfs/psi.ch/cms/trivcat/store/user/...
    ...

This file format can be directly used as an argument for the `dc_kill_pool_movers.sh` command. Since the file contains the full filenames, one can filter by grep or similar on the filenames. E.g. this chain can be used:

    dc_get_pool_list.sh | grep cms | dc_get_pool_movers.sh -k | grep "/store/user/somename"| dc_kill_pool_movers.sh

## Consistency checks

There are now two tools which do all necessary steps automatically:
-   `dc_poolconsistency_checker.sh`: finds files with no corresponding pnfs entry and with error states.
-   `dc_pnfs_replica_checker.sh`: checks part of the pnfs namespace or a list of pnfs names for files with no replicas.

However, to illustrate how to use all the basic dcache shellutil tools, all the interactive steps are demonstrated below.
For the pool based checks we'll use the `se03-lcg_cms` pool.

### Correct all the files in a pool with known error state

We need to get the list of pool files with recognized error states (i.e. the pool can detect the problem by itself).
Specifying the `-r` flag to the script would print the raw details

    dc_get_rep_ls-errors.sh se03-lcg_cms
    
    00040000000000000056B1E8
    00040000000000000056B6B0
    000400000000000000569790
    00040000000000000056B608
    000400000000000000569668
    000400000000000000569348
    000400000000000000569638
    00040000000000000056B1F8
    00040000000000000056B600
    00040000000000000056B6A0

We can try to get a mapping of these IDs to pnfs filenames:

    dc_get_rep_ls-errors.sh se03-lcg_cms |dc_get_pnfsname_from_IDlist.sh
    
    00040000000000000056B1E8 Error:Missing
    00040000000000000056B6B0 Error:Missing
    000400000000000000569790 Error:Missing
    00040000000000000056B608 Error:Missing
    000400000000000000569668 Error:Missing
    000400000000000000569348 Error:Missing
    000400000000000000569638 Error:Missing
    00040000000000000056B1F8 Error:Missing
    00040000000000000056B600 Error:Missing
    00040000000000000056B6A0 Error:Missing
    The Shellutils can be checked out from the SVN at %SVNBASE%/d-cache/dcache-utilities/shellutils.

So, all of these files are not connected to any pnfs entry. This is an explanation for the error state (but there are cases of files without  pnfs entries that are not recognized as errors, so this list is not necessarily complete)

Files with missing pnfs entries are worthless, so we remove all of these files. I usually filter the output of the above commands by grepping for the `Error` strings and then cutting off everything except the IDs:

    dc_get_rep_ls-errors.sh se03-lcg_cms |dc_get_pnfsname_from_IDlist.sh|grep "Error:Missing"|cut -f1 -d" " > toremove.lst
    The Shellutils can be checked out from the SVN at %SVNBASE%/d-cache/dcache-utilities/shellutils.

For safety reasons I redirected the list to the `toremove.lst` file. Now we can safely remove all of these entries from the pool

    dc_rep_rm_list.sh -f se03-lcg_cms toremove.lst
    The Shellutils can be checked out from the SVN at %SVNBASE%/d-cache/dcache-utilities/shellutils.

### Locate a pool's files with no pnfs entries

This is similar to the previous procedure. We first get a list of all IDs in that pool:

    dc_get_rep_ls.sh se03-lcg_cms > se03-lcg_cms-ID.lst
    wc -l se03-lcg_cms-ID.lst
       9166 se03-lcg_cms-ID.lst
    The Shellutils can be checked out from the SVN at %SVNBASE%/d-cache/dcache-utilities/shellutils.

Then we use the same sequence of commands as above to map the files to pnfs filenames and grep for the errors (I prefer to always save the intermediate lists to files for these bigger lists. Also, the commands may take quite some time to run):

    dc_get_pnfsname_from_IDlist.sh se03-lcg_cms-ID.lst > se03-lcg_cms-IDpnfs.lst
    The Shellutils can be checked out from the SVN at %SVNBASE%/d-cache/dcache-utilities/shellutils.

The command took more than 9 minutes to resolve the 9166 file entries. As before, the resulting list contains ID to pnfs filename mappings:

    head se03-lcg_cms-IDpnfs.lst
    
    000400000000000000499F98 /pnfs/projects.cscs.ch/cms/local/eggel/skimmed/Bs2MuMuPi0/output_3.root
    0004000000000000003C7F18 /pnfs/projects.cscs.ch/cms/trivcat/store/mc/2007/7/25/Spring07-b0sjpsiphi-2079/0023/487D6805-1A48-DC11-AF87-00E08140679B.root
    ...
    The Shellutils can be checked out from the SVN at %SVNBASE%/d-cache/dcache-utilities/shellutils.

Let's look how many pool files lack a pnfs enrtry:

    grep "Error:Missing" se03-lcg_cms-IDpnfs.lst |wc -l
        799
    The Shellutils can be checked out from the SVN at %SVNBASE%/d-cache/dcache-utilities/shellutils.

So, 799 files are not connected to any logical filenames, and therefore can be erased. I think that most of them are leftovers from failed deletions (I usually erase files on the CMS pools by doing `rm -rf` directly in the pnfs space. Although I do it in small batches, still some physical files seem to fail to be deleted).

**The files must be erased by using the pool's `rm` command .** Otherwise, the pool would still have them registered, and the pool space counting would be wrong. You can use the shelltools dc<sub>rep</sub><sub>rm</sub><sub>list</sub>.sh command.

    grep "Error:Missing" se03-lcg_cms-IDpnfs.lst |cut -f1 -d" " > toremove.lst
    
    dc_rep_rm_list.sh -f se03-lcg_cms toremove.lst
    The Shellutils can be checked out from the SVN at %SVNBASE%/d-cache/dcache-utilities/shellutils.

### Find files with no replicates

Usually, we will do this for a certain data set, so we need a list of the pnfs filenames belonging to that set. In CMS a data sets always can be found under a specific directory tree, so it is fairly easy to get a list using a `find` command with the respective path.
For this example I use our local test area instead of the set, because I know that there are a few problematic files.
This is one of the worst possible inconsistencies, because it deals with real file loss which goes unnoticed until the files are tested somehow. Transfer commands will often block and timeout, depending on the configuration of the system. 

    find /pnfs/projects.cscs.ch/cms/local_tests/ -type f > sampleset_pnfs.lst
    The Shellutils can be checked out from the SVN at %SVNBASE%/d-cache/dcache-utilities/shellutils.

First we need to get the pnfsIDs of these files. The result we then feed into `dc_get_cacheinfo_from_IDlist.sh` to obtain the replica locations for each ID in the list:

    dc_get_ID_from_pnfsnamelist.sh sampleset_pnfs.lst |dc_get_cacheinfo_from_IDlist.sh > cacheinfo.lst
    cat cacheinfo.lst
    ...
    0004000000000000001D2A88 se04-lcg_cms
    000400000000000000515480 se02-lcg_cms,se03-lcg_cms
    000400000000000000082AB8
    ...
    The Shellutils can be checked out from the SVN at %SVNBASE%/d-cache/dcache-utilities/shellutils.

So, the first ID has one replicate on se04, the second has two replicas, and the third file lacks any replicate and therefore is a stale entry in pnfs. If this was a real data set, the damage needs to be repaired by VO people. The admin should send a list of missing filenames to the VO site contact, so that he can invalidate the files and probably fetch them again.

The list of file without a replicate can now easily be generated using, e.g.

    while read id cache;do if test x"$cache" = x;then echo $id;fi;done < cacheinfo.lst
    The Shellutils can be checked out from the SVN at %SVNBASE%/d-cache/dcache-utilities/shellutils.

And we naturally can directly get the pnfs name mappings again using a pipe on that command

    while read id cache;do if test x"$cache" = x;then echo $id;fi;done < cacheinfo.lst |dc_get_pnfsname_from_IDlist.sh
    ...
    00040000000000000008C788 /pnfs/projects.cscs.ch/cms/local_tests/automatic_test-29508
    0004000000000000000B1E20 /pnfs/projects.cscs.ch/cms/local_tests/ccctesttree.dat
    0004000000000000000B1C58 /pnfs/projects.cscs.ch/cms/local_tests/derek/LoadTest07_FZK_5E
    0004000000000000000B1C38 /pnfs/projects.cscs.ch/cms/local_tests/derek/LoadTest07_FZK_24
    ...
    The Shellutils can be checked out from the SVN at %SVNBASE%/d-cache/dcache-utilities/shellutils.