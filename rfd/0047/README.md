---
authors: Trent Mick <trent.mick@joyent.com>
state: draft
---

# RFD 47 Retention policy for Joyent engineering data in Manta

Joyent engineering puts a lot of stuff in Manta through automated processes
(builds, images, logs, crash dumps, etc.). Currently that data is growing
forever. This can prove to be untenable. Let's get a policy for what to
retain and for how long.


## Current Status

Still writing this up and discussing. Nothing is implemented.


## mdu

Having `mdu` per <https://github.com/joyent/node-manta/issues/143> would be
really helpful to understand how much is used where.


## builds

Automatic builds of Triton and Manta bits are done for various reasons
(commits to #master of top-level repos, commits to `release-YYYYMMDD` release
branches, a regular schedule for `headnode*` builds, etc.). In general the
builds all get uploaded to one of:

```
/Joyent_Dev/public/builds/$name/$branch-$buildstamp
/Joyent_Dev/stor/builds/$name/$branch-$buildstamp
```

E.g.:

```
[trent.mick@us-east /Joyent_Dev/public/builds/vmapi]$ cat master-latest
/Joyent_Dev/public/builds/vmapi/master-20160711T195342Z

[trent.mick@us-east /Joyent_Dev/public/builds/vmapi]$ find /Joyent_Dev/public/builds/vmapi/master-20160711T195342Z
/Joyent_Dev/public/builds/vmapi/master-20160711T195342Z
/Joyent_Dev/public/builds/vmapi/master-20160711T195342Z/config.mk
/Joyent_Dev/public/builds/vmapi/master-20160711T195342Z/md5sums.txt
/Joyent_Dev/public/builds/vmapi/master-20160711T195342Z/vmapi
/Joyent_Dev/public/builds/vmapi/master-20160711T195342Z/vmapi/build.log
/Joyent_Dev/public/builds/vmapi/master-20160711T195342Z/vmapi/vmapi-pkg-master-20160711T195342Z-g678e087.tar.bz2
/Joyent_Dev/public/builds/vmapi/master-20160711T195342Z/vmapi/vmapi-zfs-master-20160711T195342Z-g678e087.imgmanifest
/Joyent_Dev/public/builds/vmapi/master-20160711T195342Z/vmapi/vmapi-zfs-master-20160711T195342Z-g678e087.zfs.gz

[trent.mick@us-east /Joyent_Dev/public/builds/vmapi]$ find release-20160707-20160707T035313Z
release-20160707-20160707T035313Z
release-20160707-20160707T035313Z/config.mk
release-20160707-20160707T035313Z/md5sums.txt
release-20160707-20160707T035313Z/vmapi
release-20160707-20160707T035313Z/vmapi/build.log
release-20160707-20160707T035313Z/vmapi/vmapi-pkg-release-20160707-20160707T035313Z-g354e119.tar.bz2
release-20160707-20160707T035313Z/vmapi/vmapi-zfs-release-20160707-20160707T035313Z-g354e119.imgmanifest
release-20160707-20160707T035313Z/vmapi/vmapi-zfs-release-20160707-20160707T035313Z-g354e119.zfs.gz
```



Some considerations for a retention policy:

- Builds for customers should always exist elsewhere in Manta for retention:
  images (including agents, platforms, etc.) in updates.joyent.com (see
  next section), headnode (aka COAL and USB) builds in
  `/Joyent_Dev/public/SmartDataCenter`, onprem headnode releases in
  `/joyentsup/stor/???` (TODO: ask support). Exceptions to this:
  `platform-debug`.
- As an educated guess, my [Trent] bet is that the `headnode*` builds are the
  big culprit for the `Joyent_Dev` account usage.
  TODO: check that
  We currently build these once per day, there are 4 flavours (headnode,
  headnode-debug, headnode-joyent, headnode-joyent-debug), and each is
  ~6G. That's 24G per day. It remains to be seen how significant that is.
- RobertM mentioned "Though I have to admit, having some of the old platform
  builds have helped for bisecting some things."
  In discussion with rm, we felt that having release builds of the platform
  in updates.jo "release" channel going back years should be sufficient.

Proposed retention policy for builds:

- Have a whitelist of build `name`s that will be deleted. This means new ones
  won't get cleaned out by default -- i.e. safe by default.
- Delete anything older than a year.
- Ensure that the `$branch-latest` "link" files stay up to date.


## updates.joyent.com

The Joyent updates server holds all the images/platforms/agentsshars et al
used for updating Triton builds after initial install. All commits to #master
of top-level repositories lead to new builds being added. There are currently
around 25k images, which naively (this doesn't count copies, or actual
blocks used in Manta) takes 1.7 TiB of space.

```
$ updates-imgadm -C '*' list -H | wc -l
   25335
$ updates-imgadm -C '*' list -H -o size | awk '{s+=$1} END {print s}'
1896458696205
```

Updates.jo has channels, which are relevant for retention policy:

```
$ updates-imgadm channels
NAME          DEFAULT  DESCRIPTION
experimental  -        feature-branch builds (warning: 'latest' isn't meaningful)
dev           true     main development branch builds
staging       -        staging for release branch builds before a full release
release       -        release bits
support       -        Joyent-supported release bits
```

One wrinkle is for Manta: Currently Manta's deployment procedure pulls images
from the *default* "dev" channel, and not from release. This isn't currently
configurable. That means there is a need for greater rentention of *Manta*
images on the "dev" channel.

Suggested retention policy for each channel:

1. `support`: No automated deletion. We can delete from here manually after
   the Joyent support org confirms all onprem customers have moved past.
2. `release`: No automated deletion. At a later date we might consider removal
   of ancient builds after a period of time after which long term support for
   that version has expired.
3. `staging`: Delete after six months. This channel is a staging area to group
   images for a release. The release process adds all latest images to the
   "release" channel as part of release. Given bi-weekly releases, 6 months
   should be ample retention.
4. `dev`: Keep if less than a year old. If this is a Manta image, keep if
   less than 3 years old (Q: is this long enough for Manta images?). Have a
   specific list of image `name` values to consider for automated deletion --
   this is intended to help guard against deleting some images that shouldn't
   be: the large and laborious manta-marlin images used for Manta jobs, the
   origin images (see RFD 46), etc.
5. `experimental`: Keep if less than 3 months old. Keep the latest by `(name,
   branch)` for at least one year. Here "branch" is inferred from `version` if
   it matches the pattern `$branch-$buildstamp-g$gitsha` or the whole version if
   it doesn't match that pattern.

Deletion requirements:

- The script should log all deletions to a log file that gets uploaded to
  Manta for verification.
- If reasonable, it would be nice to have a "purgatory" for deleted images
  so that an accidentally delete image is recoverable up to some period after
  it is removed from access via the updates API.

Ticket: <https://devhub.joyent.com/jira/browse/IMGAPI-572>



## Cruft dirs in `/Joyent_Dev/stor/builds`

```
$ mls /Joyent_Dev/stor/builds | while read d; do echo "# $d"; mget /Joyent_Dev/stor/builds/${d}release-20160707-latest; done
# firmware-tools/
/Joyent_Dev/stor/builds/firmware-tools/release-20160707-20160707T045517Z
# fwapi/
mget: ResourceNotFoundError: /Joyent_Dev/stor/builds/fwapi/release-20160707-latest was not found
# headnode-joyent/
/Joyent_Dev/stor/builds/headnode-joyent/release-20160707-20160707T062959Z
# headnode-joyent-debug/
/Joyent_Dev/stor/builds/headnode-joyent-debug/release-20160707-20160707T061159Z
# mockcloud/
/Joyent_Dev/stor/builds/mockcloud/release-20160707-20160707T044945Z
# mockcn/
mget: ResourceNotFoundError: /Joyent_Dev/stor/builds/mockcn/release-20160707-latest was not found
# old/
mget: ResourceNotFoundError: /Joyent_Dev/stor/builds/old/release-20160707-latest was not found
# propeller/
/Joyent_Dev/stor/builds/propeller/release-20160707-20160707T035011Z
# sdc-system-tests/
/Joyent_Dev/stor/builds/sdc-system-tests/release-20160707-20160707T045322Z
# sdcsso/
mget: ResourceNotFoundError: /Joyent_Dev/stor/builds/sdcsso/release-20160707-latest was not found
# vapi/
mget: ResourceNotFoundError: /Joyent_Dev/stor/builds/vapi/release-20160707-latest was not found
# vmapi/
mget: ResourceNotFoundError: /Joyent_Dev/stor/builds/vmapi/release-20160707-latest was not found
# volapi/
mget: ResourceNotFoundError: /Joyent_Dev/stor/builds/volapi/release-20160707-latest was not found
```

Those "was not found" cases are, I think, cruft.


## Other stuff

After an `mdu` implementation it should be easier to poke around looking for
particularly heavy usage areas where we need spend effort. E.g. log updates,
crash dumps.