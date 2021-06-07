---
title: Debug one connection issue of curl with git bisect
date: 2021-03-03 15:36:17
tags:
---

When use below curl command to connect to an IPv6 device, it works on my Debian 9:

```shell
curl -D - -S -T fimage ftp://root:pass@[fe80::baa4:4fff:fe0a:e3cb%eth1]/flash
```

but after upgrade the system to Debian 10, it failed with below error:
**curl: (3) URL using bad/illegal format or missing URL**
it looks the curl broken after the upgrade.

<!-- more -->

Debian 10

```
curl 7.64.0 (x86_64-pc-linux-gnu) libcurl/7.64.0 OpenSSL/1.1.1d zlib/1.2.11 libidn2/2.0.5 libpsl/0.20.2 (+libidn2/2.0.5) libssh2/1.8.0 nghttp2/1.36.0 librtmp/2.3
Release-Date: 2019-02-06
Protocols: dict file ftp ftps gopher http https imap imaps ldap ldaps pop3 pop3s rtmp rtsp scp sftp smb smbs smtp smtps telnet tftp
Features: AsynchDNS IDN IPv6 Largefile GSS-API Kerberos SPNEGO NTLM NTLM_WB SSL libz TLS-SRP HTTP2 UnixSockets HTTPS-proxy PSL
```

Debian 9

```
curl 7.52.1 (x86_64-pc-linux-gnu) libcurl/7.52.1 OpenSSL/1.0.2u zlib/1.2.11 libidn2/2.0.5 libpsl/0.17.0 (+libidn2/0.16) libssh2/1.7.0 nghttp2/1.18.1 librtmp/2.3
Protocols: dict file ftp ftps gopher http https imap imaps ldap ldaps pop3 pop3s rtmp rtsp scp sftp smb smbs smtp smtps telnet tftp
Features: AsynchDNS IDN IPv6 Largefile GSS-API Kerberos SPNEGO NTLM NTLM_WB SSL libz TLS-SRP HTTP2 UnixSockets HTTPS-proxy PSL
```

I would like to know from which version of curl is broke. Then **git bisect** comes to help.

1. Download the source code of Curl

```
git clone https://github.com/curl/curl.git
```

2. Mark the start(bad) and stop(good) rev

```
git co curl-7_64_0 # This is bad rev
git bisect start HEAD curl-7_52_1
```

3. Make a simple scrip to verify if the current version of curl is good or not

```shell
#!/bin/sh

{ make clean && autoreconf -i && ./configure --enable-ipv6 && make; } > /dev/null
./src/curl ftp://root:pass@[fe80::baa4:4fff:fe0a:e3cb%eth1]/etc/ntpd.conf > /dev/null 2>&1
exit $?
```

**NOTE: Be careful with the return value of the script:**

> ... exit with code 0 if the current source code is good/old, and exit with a code between 1 and 127 (inclusive), except 125, if the current source code is bad/new.

4. Run the bisect

```
$ git bisect run ./check_curl.sh
running ./check_curl.sh
configure: WARNING: OpenSSL headers and library versions do not match.
/usr/bin/ar: `u' modifier ignored since `D' is the default (see `U')
Bisecting: 642 revisions left to test after this (roughly 9 steps)
[685dc3c082504898e2925d635ab7241c9b8a3b20] lib/curl_setup.h: remove unicode character
running ./check_curl.sh
In function ‘ossl_keylog_callback.part.2’,
    inlined from ‘ossl_keylog_callback’ at vtls/openssl.c:238:13:
vtls/openssl.c:256:5: warning: ‘strncpy’ specified bound depends on the length of the source argument [-Wstringop-overflow=]
     strncpy(buf, line, linelen);
     ^~~~~~~~~~~~~~~~~~~~~~~~~~~
vtls/openssl.c: In function ‘ossl_keylog_callback’:
vtls/openssl.c:247:22: note: length computed here
     size_t linelen = strlen(line);
                      ^~~~~~~~~~~~
Bisecting: 321 revisions left to test after this (roughly 8 steps)
[238494fc819ad4b63d3e56ca657b7af75efcfead] KNOWN_BUGS: Fix various typos
running ./check_curl.sh
Bisecting: 160 revisions left to test after this (roughly 7 steps)
[d7c4213bd0cfd16054fa75a887d3e1a9a796cb53] multiplex: enable by default
running ./check_curl.sh
In function ‘ossl_keylog_callback.part.2’,
    inlined from ‘ossl_keylog_callback’ at vtls/openssl.c:238:13:
vtls/openssl.c:256:5: warning: ‘strncpy’ specified bound depends on the length of the source argument [-Wstringop-overflow=]
     strncpy(buf, line, linelen);
     ^~~~~~~~~~~~~~~~~~~~~~~~~~~
vtls/openssl.c: In function ‘ossl_keylog_callback’:
vtls/openssl.c:247:22: note: length computed here
     size_t linelen = strlen(line);
                      ^~~~~~~~~~~~
Bisecting: 80 revisions left to test after this (roughly 6 steps)
[46e164069d1a5230e4e64cbd2ff46c46cce056bb] url: use the URL API internally as well
running ./check_curl.sh
Bisecting: 39 revisions left to test after this (roughly 5 steps)
[ddb06ffc0f680322ce746c6e8d524dee4de84955] urlglob: improve error message
running ./check_curl.sh
Bisecting: 19 revisions left to test after this (roughly 4 steps)
[23524bf85b887adbc513bc015c9530355967bc04] examples: Fix memory leaks from realloc errors
running ./check_curl.sh
Bisecting: 9 revisions left to test after this (roughly 3 steps)
[5c73093edb3bd527db9c8abdee53d0f18e6a4cc1] urlapi: document the error codes, remove two unused ones
running ./check_curl.sh
Bisecting: 4 revisions left to test after this (roughly 2 steps)
[2097cd515289581df5dfb6eeb5942d083a871fa4] urlapi: fix support for address scope in IPv6 numerical addresses
running ./check_curl.sh
Bisecting: 2 revisions left to test after this (roughly 1 step)
[6f0afb842c733034bf36b553059270321b38c856] configure: force-use -lpthreads on HPUX
running ./check_curl.sh
Bisecting: 0 revisions left to test after this (roughly 1 step)
[f078361c0e2539689df9962f35ab22f8ea25afe9] URL and mailmap updates, remove an obsolete directory [ci skip]
running ./check_curl.sh
46e164069d1a5230e4e64cbd2ff46c46cce056bb is the first bad commit
commit 46e164069d1a5230e4e64cbd2ff46c46cce056bb
Author: Daniel Stenberg <daniel@haxx.se>
Date:   Fri Sep 14 23:33:28 2018 +0200

    url: use the URL API internally as well

    ... to make it a truly unified URL parser.

    Closes #3017

:040000 040000 fada85f23dcfaea3100d7fcce418f65d1ae983f3 ed327b156ba705d3ed0f506ed11b625d6445f49d M	lib
:040000 040000 810ff4be3b0caad4c6bc3a1aa53a067c68310874 2961ccc75c6a17f647f5b36b63f8a83bc6e60653 M	tests
bisect run success
```

Just in few minutes, I could find the wrong commit which broke the function. :)

5. Replay what did 'git bisect' do with **git bisect log**

```
$ git bisect log
# bad: [f3294d9d86e6a7915a967efff2842089b8b0d071] RELEASE-NOTES: 7.64.0
# good: [44b9b4d4f56d6f6de92c89636994c03984e9cd01] RELEASE-NOTES: curl 7.52.1
git bisect start 'HEAD' 'curl-7_52_1'
# good: [d3ab7c5a21ebfa0e3ceb3a395f23aceb5ddc58b6] openssl: fix too broad use of HAVE_OPAQUE_EVP_PKEY
git bisect good d3ab7c5a21ebfa0e3ceb3a395f23aceb5ddc58b6
# good: [685dc3c082504898e2925d635ab7241c9b8a3b20] lib/curl_setup.h: remove unicode character
git bisect good 685dc3c082504898e2925d635ab7241c9b8a3b20
# bad: [238494fc819ad4b63d3e56ca657b7af75efcfead] KNOWN_BUGS: Fix various typos
git bisect bad 238494fc819ad4b63d3e56ca657b7af75efcfead
# good: [d7c4213bd0cfd16054fa75a887d3e1a9a796cb53] multiplex: enable by default
git bisect good d7c4213bd0cfd16054fa75a887d3e1a9a796cb53
# bad: [46e164069d1a5230e4e64cbd2ff46c46cce056bb] url: use the URL API internally as well
git bisect bad 46e164069d1a5230e4e64cbd2ff46c46cce056bb
# good: [ddb06ffc0f680322ce746c6e8d524dee4de84955] urlglob: improve error message
git bisect good ddb06ffc0f680322ce746c6e8d524dee4de84955
# good: [23524bf85b887adbc513bc015c9530355967bc04] examples: Fix memory leaks from realloc errors
git bisect good 23524bf85b887adbc513bc015c9530355967bc04
# good: [5c73093edb3bd527db9c8abdee53d0f18e6a4cc1] urlapi: document the error codes, remove two unused ones
git bisect good 5c73093edb3bd527db9c8abdee53d0f18e6a4cc1
# good: [2097cd515289581df5dfb6eeb5942d083a871fa4] urlapi: fix support for address scope in IPv6 numerical addresses
git bisect good 2097cd515289581df5dfb6eeb5942d083a871fa4
# good: [6f0afb842c733034bf36b553059270321b38c856] configure: force-use -lpthreads on HPUX
git bisect good 6f0afb842c733034bf36b553059270321b38c856
# good: [f078361c0e2539689df9962f35ab22f8ea25afe9] URL and mailmap updates, remove an obsolete directory [ci skip]
git bisect good f078361c0e2539689df9962f35ab22f8ea25afe9
# first bad commit: [46e164069d1a5230e4e64cbd2ff46c46cce056bb] url: use the URL API internally as well
```

At last, remember to finish the bisect by **git bisect reset**.

6. More

Actually when I debug this issue, I also build the curl with the latest version on my Debian 10, it also work. So I'm interestig for which commit fix this issue since curl-7_64_0. The **git bisect** also can help on it.
The latest version is "curl-7_75_0", I will mark this as "fixed", and "unfixed" on "curl-7_64_0" as below

```
$ git co curl-7_75_0
Previous HEAD position was a0c461434 gtls: survive not being able to get name/issuer
HEAD is now at 2f33be817 RELEASE-NOTES: synced
chenhuiz@lnxchenhuiz2:~/work/curl
$ git bisect start --term-new=fixed --term-old=unfixed
chenhuiz@lnxchenhuiz2:~/work/curl
$ git bisect fixed HEAD
chenhuiz@lnxchenhuiz2:~/work/curl
$ git bisect unfixed curl-7_64_0
Bisecting: 1412 revisions left to test after this (roughly 11 steps)
[1722eb83b4adf879426908bc6006f869ca224564] curl_getenv.3: Fix the memory handling description
```

Then for the script, I need to modify the return value to match the rule for **git bisect**

```
#!/bin/sh

{ make clean && autoreconf -i && ./configure --enable-ipv6 && make; } > /dev/null
if ./src/curl ftp://root:pass@[fe80::baa4:4fff:fe0a:e3cb%eth1]/etc/ntpd.conf > /dev/null 2>&1; then
	exit 1;
else
	exit 0
fi
```

The progress is also smooth:

```
$ git bisect run ./check_curl.sh
running ./check_curl.sh
Bisecting: 705 revisions left to test after this (roughly 10 steps)
[887ebc384c5687a8e154b3b6feb74cdaa8df4adc] ngtcp2: don't reinitialize SSL on Retry
running ./check_curl.sh
Bisecting: 352 revisions left to test after this (roughly 9 steps)
[ae3f838b9a8b185d98b2a5442a3d220ac9a3a11d] RELEASE-NOTES: synced
running ./check_curl.sh
Bisecting: 176 revisions left to test after this (roughly 8 steps)
[35b8bea20f90873e321e1e77d8a9936245c11ac9] tests: update fixed IP for hostip/clientip split
running ./check_curl.sh
Bisecting: 87 revisions left to test after this (roughly 7 steps)
[44ea2bef38f3f66f6c4f2ef5f965c7008e628c26] appveyor: add support for other build systems
running ./check_curl.sh
Bisecting: 43 revisions left to test after this (roughly 6 steps)
[8fba2d6a6b15ca287740a36715d679825c38df83] url: convert the zone id from a IPv6 URL to correct scope id
running ./check_curl.sh
Bisecting: 21 revisions left to test after this (roughly 5 steps)
[3cfcdf08d85488d162ce10a6ce5433dbe510264d] hostip: CURL_DISABLE_SHUFFLE_DNS
running ./check_curl.sh
Bisecting: 10 revisions left to test after this (roughly 4 steps)
[320cec284d142e67cdf3f6c76f3851449991d87f] ssh: move variable declaration to where it's used
running ./check_curl.sh
Bisecting: 5 revisions left to test after this (roughly 3 steps)
[0da8441298569dfd714e7b21f74aab373b95d2f7] mbedtls: enable use of EC keys
running ./check_curl.sh
Bisecting: 2 revisions left to test after this (roughly 2 steps)
[528b284e4b145ff38c5ab87b4d6daa5d796fd938] udpateconninfo: mark variable unused
running ./check_curl.sh
Bisecting: 0 revisions left to test after this (roughly 1 step)
[9406d93e77e7923e2788e43df50cf3a577f1c42b] configure: detect getsockname and getpeername on windows too
running ./check_curl.sh
8fba2d6a6b15ca287740a36715d679825c38df83 is the first fixed commit
commit 8fba2d6a6b15ca287740a36715d679825c38df83
Author: Daniel Stenberg <daniel@haxx.se>
Date:   Tue May 21 09:43:10 2019 +0200

    url: convert the zone id from a IPv6 URL to correct scope id

    Reported-by: GitYuanQu on github
    Fixes #3902
    Closes #3914

:040000 040000 2013af3dc3dce59d10c55213151f0b63c3bad53f cca388c0c0e568c6f9f0d05c0c8c2f4197ad4f1b M	lib
bisect run success
```

That's it!

Reference: [git bisect](https://git-scm.com/docs/git-bisect)
