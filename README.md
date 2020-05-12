# mirrors-countme

Parse http `access_log` data, find DNF `countme` requests, and output
structured data that lets us estimate the number of people using various
Fedora releases.

See [Changes/DNF Better Counting] for details.

[Changes/DNF Better Counting]: https://fedoraproject.org/wiki/Changes/DNF_Better_Counting

## How it works

The short version:

1. Starting in Fedora 32, DNF adds "countme=N" to one random HTTP request
   per week for each repo that has `countme` enabled.
2. This script parses httpd `access_log` files for mirrors.fedoraproject.org,
   finds those requests, and yields the following information:
   * request timestamp, repo & arch
   * client OS name, version, variant, and arch
   * client "age", from 1-4: 1 week, 1 month, 6 months, or >6 months.
3. We use that data to make cool charts & graphs and estimate how many Fedora
   users there are and what they're using.

## Technical details

### Client behavior & configuration

DNF 4.2.9 added the `countme` option, which [dnf.conf(5)] describes like so:

>    Determines whether a special flag should be added to a single, randomly
>    chosen metalink/mirrorlist query each week.
>    This allows the repository owner to estimate the number of systems
>    consuming it, by counting such queries over a week's time, which is much
>    more accurate than just counting unique IP addresses (which is subject to
>    both overcounting and undercounting due to short DHCP leases and NAT,
>    respectively).
>
>    The flag is a simple "countme=N" parameter appended to the metalink and
>    mirrorlist URL, where N is an integer representing the "longevity" bucket
>    this system belongs to.
>    The following 4 buckets are defined, based on how many full weeks have
>    passed since the beginning of the week when this system was installed: 1 =
>    first week, 2 = first month (2-4 weeks), 3 = six months (5-24 weeks) and 4
>    = more than six months (> 24 weeks).
>    This information is meant to help distinguish short-lived installs from
>    long-term ones, and to gather other statistics about system lifecycle.

[dnf.conf(5)]: https://dnf.readthedocs.io/en/latest/conf_ref.html

Note that the default is False, because we don't want to enable this for every
repo you have configured.

Starting with Fedora 32, we set `countme=1` in Fedora official repo configs:

```
[updates]
name=Fedora $releasever - $basearch - Updates
#baseurl=http://download.example/pub/fedora/linux/updates/$releasever/Everything/$basearch/
metalink=https://mirrors.fedoraproject.org/metalink?repo=updates-released-f$releasever&arch=$basearch
enabled=1
countme=1
repo_gpgcheck=0
type=rpm
gpgcheck=1
metadata_expire=6h
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-$releasever-$basearch
skip_if_unavailable=False
```

This means that the default configuration only adds `countme=N` when using
official Fedora repos, which are all done via HTTPS connections to
mirrors.fedoraproject.org. `countme=N` does _not_ get added in subsequent
requests to the chosen mirror(s).

### Privacy, randomization, and user counting

DNF makes a serious effort to keep the `countme` data anonymous _and_ accurate
by only sending `countme` with one _random_ request to each enabled repo _per
week_. So how does it decide when the week starts, and how does it choose
which request?

First, all clients use the same "week": Week 0 started at timestamp 345600
(Mon 05 Jan 1970 00:00:00 - the first Monday of POSIX time), and weeks are
exactly 604800 (7×24×60×60) seconds long.

Second, all clients have the same random chance - currently 1:4 - to send
`countme` with any request in a given week. Once it's been sent, the client
won't send another `countme` for that repo for the rest of the week.

The default update interval for the `updates` repo is 6 hours, which means
that clients who use `dnf-makecache.service` will probably send `countme`
sometime in the first 24 hours of a given week - and nothing for the rest of
the week.

This means that _daily_ totals of users are unreliably variable, since the
start of the week will have more `countme` requests than the end of the week.
But the weekly totals should be a consistent, representative sample of the
total population.

### Data collected

The only data we look at is in the HTTP request itself. Our log lines are in
the standard Combined Log Format, like so:

```
240.159.140.173 - - [29/Mar/2020:16:04:28 +0000] "GET /metalink?repo=fedora-modular-32&arch=x86_64&countme=1 HTTP/2.0" 200 18336 "-" "libdnf (Fedora 32; workstation; Linux.x86_64)"
```

(NOTE: That's a fake IP address - it's actually the 4-byte UTF-8 encoding for "🌭".)

We only look at log lines where the request is "GET", the query string includes
"countme=N", the result is 200 or 302, and the User-Agent string matches the
libdnf User-Agent header.

The only data we use are the timestamp, the query parameters (repo, arch,
countme), and the libdnf User-Agent values.

#### libdnf User-Agent data

As in the log line above, the User-Agent header that libdnf sends looks like this:

```
User-Agent: libdnf (Fedora 32; workstation; Linux.x86_64)
```

This string is assembled in [`libdnf/utils/os-release.cpp:getUserAgent()`] and
the format is as follows:

```
{product} ({os_name} {os_version}; {os_variant}; {os_canon}.{os_arch})
```

where:

* `product`    = "libdnf"
* `os_name`    = `/etc/os-release` `NAME`
* `os_version` = `/etc/os-release` `VERSION_ID`
* `os_variant` = `/etc/os-release` `VARIANT_ID`
* `os_canon`   = rpm `%_os` (via libdnf `getCanonOS()`)
* `os_arch`    = rpm `%_arch` (via libdnf `getBaseArch()`)

(Older versions of libdnf sent `libdnf/{LIBDNF_VERSION}` for the `product`,
but the version string was dropped in libdnf 0.37.2 due to privacy concerns;
see [commit d8d0984].)

[`libdnf/utils/os-release.cpp:getUserAgent()`]: https://github.com/rpm-software-management/libdnf/blob/0.47.0/libdnf/utils/os-release.cpp#L108
[commit d8d0984]: https://github.com/rpm-software-management/libdnf/commit/d8d0984

#### repo=, arch=, countme=

The `repo=` and `arch=` values are exactly what's set in the URL in the `.repo`
file.

`arch` is usually set as `arch=$basearch`, which means that `os_arch` and
`repo_arch` are _probably_ the same value. But it's totally valid for a client
to set up a repo for a version/arch different from their host OS, so these
are considered separate.

`countme`, as discussed in [dnf.conf(5)], is a value from 1 to 4 indicating
the "age" of the system, counted in _full_ weeks since the system was
first installed. The values are:

1. One week or less (0-1 weeks)
2. Up to one month (2-4 weeks)
3. Up to six months (5-24 weeks)
4. More than six months (25+ weeks)

These are defined in [`libdnf/repo/Repo.cpp:COUNTME_BUCKETS`].

[`libdnf/repo/Repo.cpp:COUNTME_BUCKETS`]: https://github.com/rpm-software-management/libdnf/blob/0.47.0/libdnf/repo/Repo.cpp#L92
