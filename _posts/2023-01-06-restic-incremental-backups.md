---
layout: post
title:  Restic
categories: [Backup,Reliability]
excerpt: Backing up your workstation is important, but even more important, is for it to be quick and easy. In this post we'll see how Restic does just that!
---

# Incremental backup with restic

## Summary

[Restic: Backups done right!](https://restic.net/) is a modern, cross-platform tool to allow for quick incremental backups. It is also free and easily configurable, as we'll see.


## Installation

They have a really good documentation for installation instruction cross-platform available [here](https://restic.readthedocs.io/en/latest/020_installation.html)


## Getting started

### Initial setup

Restic needs us to create a "repository" which mostly is a folder where restic will store all data related to this repository:
  - config
  - locks to prevent concurrent writing
  - snapshots

Thus, we will create a folder to initiate our repository

```bash
$ mkdir /srv/restic-repo
$ restic init --repo /srv/restic-repo

enter password for new repository:
enter password again:
created restic repository 085b3c76b9 at /srv/restic-repo
Please note that knowledge of your password is required to access the repository.
Losing your password means that your data is irrecoverably lost.
```


### Backing up

Here we are going to cover a simple use case, using a local backup repository in you local filesystem. Even though this is just for demonstration purposes you could already use this simple setup with virtually any type of storage that can be mounted on your filesystem (sometimes with some considerations for example for SMB / CIFS mounts: [#2659](https://github.com/restic/restic/issues/2659))


But once you got the gist of it, know that restic is compatible with a lot of of different backend (storage type / protocols) out of the box, such as but not limited to:
  - Amazon S3
  - Azure Blob Storage
  - Google Cloud storage
  - sFTP

Note that restic's backend catalogue can be extended even further when combined with [Rclone](https://rclone.org/), alongside some other really interesting features that will be covered in a post dedicated to rclone


We will then be able to create our very first "snapshot" like so:

```bash
$ restic -r /srv/restic-repo --verbose backup ~/work
open repository
enter password for repository:
password is correct
lock repository
load index files
start scan
start backup
scan finished in 1.837s
processed 1.720 GiB in 0:12
Files:        5307 new,     0 changed,     0 unmodified
Dirs:         1867 new,     0 changed,     0 unmodified
Added:      1.200 GiB
snapshot 40dc1520 saved
```

Now that you snapshot is generated, all data in work has been backed-up successfully. But what would happen if we decide to add a new file in our `~/work` folder ?

```bash
$ restic -r /srv/restic-repo --verbose backup ~/work
open repository
enter password for repository:
password is correct
lock repository
load index files
using parent snapshot d875ae93
start scan
start backup
scan finished in 1.951s
processed 1.720 GiB in 0:05
Files:           1 new,     0 changed,  5307 unmodified
Dirs:            0 new,     0 changed,  1867 unmodified
Added to the repository: 2KiB (1.720 GiB stored)
snapshot 79766175 saved
```

As you can see restic will check for changes in the backup folder (herer `~/work`) and only apply changes since last snapshot. This allow to reduce storage space needed for backups, as well as significantly speed up the process if you plan to backup regularly

More info on restic documentation [Backing up](https://restic.readthedocs.io/en/latest/040_backup.html)


### Restoring backup

We all learnt the saying "Untested backup data is just data" so we'll quickly cover how to make sure everything is safely stored and our latest backup is healthy

More info on restic documentation [Restoring from a backup](https://restic.readthedocs.io/en/latest/050_restore.html)


## Going deeper

### Exclude unwanted files

If you run restic in verbose mode, you'll probably see a lot of files that you wouldn't ideally want in your backup. I recommend taking a close look a what exactly is included in your backup path, as you will most likely identify a lot of unwanted files such as:
- Binaries
- Cache folder
- Plethora of small, unused files 

#### Cache files

Excluding these files from your backup will not only reduce the storage space required on your backend (ex: binaries) but can also dramatically speed up your backup process. These folders are bound to see their content change often, so you will most likely slow down the whole process for nothing

This can simply be done by adding `--exclude-caches` flag to your backup command

```bash
$ restic -r /srv/restic-repo --verbose backup ~/work
```

#### Manually filtering content

You might want to filter a handfull of files / folders from your backup, you could then manually instruct restic to exclude some files / path:

- Exclude patterns by adding flag `--exclude=<glob>`
- Exclude patterns regardless of path's case `--iexclude=<glob>`
- Exclude files that are too large `--exclude-larger-than <size>`


If you prefer, exclusion rules can even be stored in a dedicated configuration file, simply add one pattern per line, and pass configuration's file path as parameter upon  

```bash
$ cat .my_restic_exclude_file
.trash
*.terraform
```

The best with using an exclude file, is that it can easily be shared with your team, versioned in a git repo or even backed-up as part of your snapshot ! :D

```bash
restic -r /srv/restic-repo --verbose --exclude-file .my_restic_exclude_file backup ~/work
```

_Note: Be careful when excluding files to make sure you know what you're doing, as mistakenly excluding a required configuration file might lead to unusable backup data_


There can be pretty detailed configuration as there a variety of options, here are some link to dive deeper:
- More exclude options at [restic documentation: Excluding files](https://restic.readthedocs.io/en/latest/040_backup.html#excluding-files))

- File including options at [restic documentation: Including files](https://restic.readthedocs.io/en/latest/040_backup.html#including-files)

- Now there is even a way to chain exclusions / inclusions in a single path (more informations on [ restic's github related pull request discussion](https://github.com/restic/restic/pull/2311))


### Tagging

Each snapshot has an id, here `40dc1520` and can be tagged for better readability for when you will need them

```bash
restic tag --add 'Before update of XX' [snapshot-ID]
```

Tag can also be associated to a snapshot upon creation


## List existing snapshots

```bash
restic snapshots -v -r /srv/restic-repo
enter password for repository:
repository xxxxxxx opened (repository version 1) successfully, password is correct
found 2 old cache directories in [...], run `restic cache --cleanup` to remove them
ID        Time                 Host               Tags                          Paths
----------------------------------------------------------------------------------------------
472d0e60  2022-05-19 15:36:16  local-machine                                ~/work
317e2319  2022-05-19 18:44:14  local-machine  Before update of XX           ~/work
----------------------------------------------------------------------------------------------
2 snapshots
```


## Sum up: Real life exemple

Here is now a real life exemple leveraging all topics mentioned in this post:

```bash
restic backup -r /srv/restic-repo backup ~/work \ 
--exclude-file ./.utilities/.restic_exclude_file \
--exclude-caches \
--tag 'Before OS update XX' \
--verbose 

open repository
enter password for repository:
repository xxxxxxx opened (repository version 1) successfully, password is correct
lock repository
using parent snapshot xxxxxx
load index files
start scan on [/srv/restic-repo]
start backup on [/srv/restic-repo]
scan finished in 66.214s: 908063 files, 53.312 GiB
[...]
[30:51] 85.55%  823077 files 45.608 GiB, total 908063 files 53.312 GiB, 3 errors ETA 5:12
[...]

Files:       398996 new,  9208 changed, 499881 unmodified
Dirs:        138391 new, 94843 changed,     0 unmodified
Data Blobs:  257009 new
Tree Blobs:  227854 new
Added to the repository: 19.331 GiB (19.362 GiB stored)

processed 908085 files, 53.323 GiB in 42:37
snapshot 317e2319 saved
```

Backups using restic are indeed fast and easy to setup, so give it a try here at [https://restic.net/](https://restic.net/)

In a future post we'll cover more advanced use cases using rclone to integrate even more backends and even allow for highly available backups.