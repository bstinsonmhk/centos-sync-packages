sync2git
========

This will take a bunch of modules and/or packages from a koji tag, or a
koji compose, and compare against the sources in a CentOS style git
repository ... for newer versions of the modules/packages it will call
[alt-src](https://github.com/release-engineering/alt-src) to sync them.

It can filter modules/packages manually and also optionally call a
CVE checker service to filter embargoed data.

Also has optional, time limited, caching.

Default setup is to sync from internal Red Hat EL8 nightly composes to
[CentOS Stream 8](https://git.centos.org/rpms/centos-release-stream/).

Usage:
    `cron-sync2git.sh`
    `cron-sync2build.sh`
    `cron-dashboard.sh`

Other commands:
 * ./sync2git.py - Take packages from koji tag/compose and sync them to git.
 * * ./sync2git.py force-push-module N:S:V:C - Push without checking CVE.
 * * ./sync2git.py force-push-package N-V-R - Push without checking CVE.
 * * ./sync2git.py push - Push packages/modules to git (depending on options).
 * * fgrep 'Filtered Mod' logs/sync2git-$(date --iso=date)
 * * fgrep 'Filtered Pkg' logs/sync2git-$(date --iso=date)
 * ./sync2build.py - Take packages from git and sync them to koji builds.
 * * ./sync2build.py packages
 * * ./sync2build.py modules
 * * ./sync2build.py check-nvr - Check a given NVR against git.
 * * ./sync2build.py check-nvra - Check a given NVRA against git.
 * * ./sync2build.py build-nvr - Build a given NVR from git.
 * * ./sync2build.py build-nvra - Build a given NVRA from git.
 * * ./sync2build.py bpids-list - list koji build tasks
 * * ./sync2build.py bpids-wait - wait for current koji build tasks
 * * ./sync2build.py tag-rpms-hash - Give a hash for the tag, based on all rpms.
 * * ./sync2build.py tag-srpms-hash - Give a hash for the tag, based on srpms.
 * * ./sync2build.py summary-packages - ?
 * * ./sync2build.py list-packages - ?
 * * ./sync2build.py nvra-unsigned-packages - List packages which aren't signed.
 * * ./sync2build.py list-unsigned-packages - List packages which aren't signed.
 * ./compose.py - Get data from a compose.
 * * ./compose.py &lt;compose-base-url>
 * ./access.py - Query CVE checker data
 * * ./access.py -h = Do history lookups, for speed
 * * ./access.py -t &lt;duration> = set the query timeout
 * * ./access.py logs &lt;query-id> = Show the log data for the query id
 * * ./access.py history name[-version[-release]] = Show the history of the n/nvr
 * * ./access.py history-n name = Show the history of the name
 * * ./access.py nvrs nvr... = Query the given NVRs
 * * ./access.py names name... = Query the given NVRs for local pkgs. named
 * * ./access.py names name... = Query the given NVRs for local pkgs. named
 * * ./access.py file-nvrs nvr... = Query all the nvrs in the files
 * ./rpmvercmp.py = rpm version comparison in python
 * * ./rpmvercmp.py &lt;s1> &lt;s2> = compare s1 vs. s2 using rpmvercmp logic
 * ./mtimecache.py = caching of data based on mtime
 * * ./mtimecache.py dur  &lt;secs> = convert seconds into human time
 * * ./mtimecache.py durs &lt;secs> = convert seconds into human time
 * * ./mtimecache.py secs &lt;time> = convert human time into seconds
 * * ./mtimecache.py time &lt;secs> = convert seconds into digitial clock
.py = rpm version comparison in python

Finding modules to force push:

 * fgrep 'Filtered Mod' logs/sync2git-$(date --iso=date)*

WTF is it doing?
================

This is roughly how packages make it to el8 stream, with all the
arguments/configuration/options we use:

 * Developer/packager creates new code for el8.
 * That code gets push to an internal compose.
 * **sync2git**
 * * Looks at the compose, for packages and modules.
 * * Packages:
 * * * Filters out any package names in `conf/sync2git-packages-denylist.txt`
 * * * Compares the packages versions vs. what is in git, removing packages already there (kind of assumes we don't go backwards).
 * * * Filters out any package NVRs that are denied by the CVE checker.
 * * * **alt-src** the packages
 * * Modules:
 * * * Filters out any modules which contain a package NVR that is denied by the CVE checker.
 * * * Compares the module versions vs. what is in git, removing modules already there (kind of assumes we don't go backwards).
 * * * **alt-src** the module
 * * * **alt-src** the packages within the module.
 * **sync2build**
 * * Looks at the koji tag `dist-c8-strea`, for packages.
 * * * Filters out any names/nvrs in `conf/sync2build-packages-denylist.txt`. Note that only a single package is in the koji tag, so removing that NVR will remove that package.
 * * For each of the remaining packages we then:
 * * * Check if we've built that package recently, and if so skip it.
 * * * On a timed basis we then look at the git repo. for that package:
 * * * * If it has a README.debrand file, we skip it.
 * * * We then check the tags on that git repo. and parse NVRs from them and for each tagged NVR we:
 * * * * If it is a non-stream package we skip it.
 * * * * If it is a el8 branch release we skip it.
 * * * * If it is a module release we skip it.
 * * * * If it is a rebuild release we skip it.
 * * * * If it matches `conf/sync2build-gittags-denylist.txt` we skip it.
 * * * * If it is older than the latest build, we skip it.
 * * * * For any NVRs that are left, we ask **koji to build it**.
