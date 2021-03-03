What this is
============

This is a fork of [OpenSSL](www.openssl.org). In addition to the website,
the official source distribution is at https://github.com/openssl/openssl.
The OpenSSL `README` can be found at `README-OpenSSL.md`

This fork supports an API that can be used by QUIC implementations. Quoting
the IETF Working group [charter](https://datatracker.ietf.org/wg/quic/about/),
QUIC is a "UDP-based, stream-multiplexing, encrypted transport protocol." If
you don't need QUIC, you should use the official OpenSSL channels.

This API's here are used by Microsoft's
[MSQUIC](https://github.com/microsoft/msquic) and Google's
[Chromium QUIC](https://chromium.googlesource.com/chromium/src/+/master/net/quic/)

We are not in competition with OpenSSL project. In fact, we informed them of
our plans to fork the code before we went public.
We do not speak for the OpenSSL project, and can only point to a
[blog post](https://www.openssl.org/blog/blog/2020/02/17/QUIC-and-OpenSSL/) that
provides their view.

This fork can be considered a supported version of openssl/openssl/#8797
[OpenSSL PR 8797](https://github.com/openssl/openssl/pull/8797).
We will endeavor to track OpenSSL releases within a day or so, and there is an
item below about how we'll follow their tagging. Once OpenSSL provides reasonable
support for the two implementations above, we will gladly shut this project down.

On to the questions and answers.

What about branches?
--------------------
We don't want to conflict with OpenSSL branch names. Our current plan is to append
`+quic`. Release tags are likely to be the QUIC branch with `-releaseX` appended.
For example, the OpenSSL tag `openssl-3.0.0-alpha12` would have a branch named
`openssl-3.0.0-alpha12+quic` and a release tag of `openssl-3.0.0-alpha12+quic-release1`

How are you keeping current with OpenSSL?
-----------------------------------------
(In other words, "what about rebasing?")

Our plan it to continuously rebase on top of an upstream release tag. In particular:
- The changes for QUIC will always be at the tip of the branch - you will know what
is the original OpenSSL for and what is QUIC.
- New versions are quickly created once upstream creates a new tag.
- The use of git commands (such as "cherry") can be used to ensure that all changes
have moved forward with minimal or no changes. You will be able to see "QUIC: Add X"
on all branches and the commit itself will be nearly identical on all branches, and
any changes to that can be easily identified.

What about library names?
-------------------------
This is currently an open issue. We don't want to conflict with the libraries
provided by Linux Distributions, nor with OpenSSL releases. If you have any
suggestions, please open an issue to discuss.

...And the Executable?
----------------------
We currently do not have any plans to change the name, mainly because we
haven't made any changes there. If you see a need, please open an issue.

...and FIPS?
------------
We are not doing anything with FIPS. This is actually good news: you should
be able to load the FIPS module into an application built against this fork
and everything should Just Work&#8482;.

How can I Contribute?
---------------------
We want any code here to be acceptable to OpenSSL. This means that all contributors
must have signed the appropriate
[contributor license agreements](https://www.openssl.org/policies/cla.html). We
will not ask for copies of any paperwork, you just need to tell us that you've
done so (and we might verify with OpenSSL). We are only interested in making it
easier and better for at least the mentioned QUIC implementations to use a variant
of OpenSSL. If you have a pull request that changes the TLS protocol, or adds
assembly support for a new CPU, please contribute that to OpenSSL.
