# Cookies-over-HTTP Bad

## A Problem

Cookies sent over plaintext HTTP are visible to anyone on the network. This visibility exposes substantial amounts of
data to network attackers (passive or active). We know, for example, that long-lived and stable cookies have enabled
[pervasive monitoring][1] in the past (see [Google's PREF cookie][2]), and we know that HTTPS provides significant
confidentiality protections against this kind of attack.

Ideally, browsers would mitigate these monitoring opportunities by making it more difficult to persistently track users
via cookies sent over non-secure connections.

## A Proposal

TL;DR: Expire cookies early rather than sending them over non-secure connections.

When building the `Cookie` header for an outgoing request to a non-secure URL, let's first check each cookies'
creation date. If that date is older than some arbitrary cutoff (let's start with twelve months), let's not add the
cookie to the outgoing `Cookie` header. Instead, let's delete the cookie. Over time, that cutoff could be reduced to
something suitably small (say, a few days).

Let's also slightly modify the creation-time algorithm in [step 17.3 of section 5.4 of RFC6265bis][3] by persisting
the creation time only for cookies whose value doesn't change:

```
3.  If the old-cookie's value is the same as the newly-created cookie's value, then update the
    creation-time of the newly-created cookie to match the creation-time of the old-cookie.
```

That's it.

_Note that this proposal does not aim to prevent third-party tracking. It aims to set HTTPS as the minimum bar for
state on the web (which, hopefully, we'd all agree is "powerful")._

### Hrm. Spell this out a bit?

Sure thing. Let's say that `http://A.com` embeds `http://B.com`, and their respective cookie jars look something like
the following:

| A.com ||
|-------|-|
| Name  | Created |
| cookie1 | 2017-04-01 |
| cookie2 | 2018-01-01 |

| B.com ||
|-------|-|
| Name  | Created |
| cookie3 | 2017-04-01 |
| cookie4 | 2018-01-01 |


On 2018-03-01, the user loads `http://A.com` as a top-level document. All of its cookies are less than a year old,
so they're all sent in the `Cookie` header. It embeds `http://B.com`. All of its cookies are likewise less than a
year old, so they're included in the `Cookie` header as usual.

On 2018-04-02, the user loads `http://A.com` again. `cookie1` is now older than a year, while `cookie2` is younger.
The request's `Cookie` header contains `cookie2=value`, and the browser deletes `cookie1`. It embeds `http://B.com`.
Since `cookie3` is now more than a year old, the browser deletes that cookies, and sends `cookie4=value` in the `Cookie`
header.

## FAQ

### Won't this break things?

Cookies are somewhat fragile, and can be [evicted][4] at any time for reasons outside developers' control, so there
is unlikely to be a high compatibility cost: users are not likely to see breakage. On the other hand, services that
use long-lived non-secure cookies are likely to be unhappy, which is good. There are distinct risks to sending
cookies over non-secure channels, especially when done at scale as part of an advertising network.

Developers responsible for affected services have a few options:

1.  They can migrate to HTTPS, which is good for everyone.

2.  They can adopt a system similar to DoubleClick's rotation of its `ID`, whose value is re-encrypted ~daily. Folks
    who need to maintain state in areas of the world where HTTPS is difficult to implement may be forced to stick
    with this option longer than they'd like.

3.  They could migrate away from cookies as identifiers, towards something like `localStorage`, or, much worse,
    fingerprinting. Our goal should be to ensure that the friction involved with rebuilding their entire
    infrastructure to use a different identification mechanism will be higher than the pain introduced as we roll
    out this change.

4.  They could trivially modify the cookie value as a perversion of the approach in #2 above. That is,
    `Set-Cookie: name=[value]-1`, followed by `Set-Cookie: name=[value]-2`, followed by
    `Set-Cookie: name=[value]-3`, and so on. If this is the route developers choose, browsers would be well-served
    to consider countermeasures. Given past deprecation experience, this doesn't seem likely: for example, only a
    very small number of sites switched away from `<input type="password">` to something more esoteric when
    browsers started showing the "Not Secure" chip for password fields on non-secure pages.

The most worrisome group for unintentional breakage are Enterprisey intranet single-sign on servers. This doesn't
seem terribly concerning, because it seems useful to encourage even enterprises to migrate to HTTPS for critical
services like SSO. If it turns out that we ought to be more concerned about this, browsers could consider adding
an enterprise policy to unblock old cookies for a given set of sites (though it seems valuable to avoid doing so
if possible).

### What's a reasonable cutoff point to start with?

An excellent question, which I think we'll need to answer with data. Chrome has collected metrics to measure the
age of the oldest cookie in each same-site/cross-site request sent to a non-secure endpoint. As of December 2019,
the percentile buckets break down as follows (average ages in ~days):

|       | Same-Site | Cross-Site |
|-------|-----------|------------|
| 25%   | 0.7   | 5.2 |
| 50%   | 10.4  | 58  |
| 75%   | 93.9  | 207.4 |
| 95%   | 464.9 | 609.1 |
| 96%   | 522.1 | 661.9 |
| 97%   | 588.6 | 714.5 |
| 98%   | 677.1 | 754.5 |
| 99%   | 761.8 | 823.2 |
| 99.5% | 848.9 | 956.2 |

Squinting a bit, it seems reasonable to start at somewhere around two years, which falls into a bucket that would have
a one-time effect on ~2% of same-site requests, and ~3% of cross-site requests. It's a compromise between a
short-enough lifetime to have a real impact on pervasive monitoring and non-secure tracking in general, while at the
same time not breaking things like SSO on an ongoing basis (being forced to reauthenticate once a year doesn't seem
like a massive burden).

### Why base this on creation date rather than limiting expiration?

We could limit the expiration time for cookies set over HTTP (that's what Martin Thompson's ahead-of-its-time
[omnomnom][5] proposal suggests). We'd have a hard time doing the same for cookies set over HTTPS, however, and
those cookies can also be sent over HTTP if they lack the `Secure` attribute. [Mozilla's research on the topic][6] as
well as Chrome's data on cookie types suggests that that happens far too often to be easily adjustable.

The approach proposed in this doc allows a request-time decision which targets only those cookies which would
actually be sent over HTTP, which seems like a net that's just wide enough to catch the cookies we care about,
while not attacking those which _could_ be a problem, but _aren't_ in practice.

### Why base this on creation date rather than the Secure attribute?

Developers who are already serving their sites over HTTPS can avoid any impact of this proposal by annotating
their cookies with the `Secure` attribute, which is a robust protection against being delivered or modified
over HTTP. It would be lovely if more folks used that attribute: perhaps we should encourage them to do so by
modifying this proposal to affect all cookies that lack that attribute?

That approach might be simpler to communicate to developers, and it would go beyond the current protections
against pervasive monitoring to include a potential defense against DNS poisoning.

These are reasonable arguments. On the other hand, only something like 7.5% of cookies use the `Secure`
attribute according to Chrome's data. Since we'd be expiring cookies regardless of the _actual_ risk, and
since [something like 70% of users' navigations are to secure pages][7] that the current proposal excludes,
we'd end up affecting significantly more requests.

Still, this would be a more robust defense for users and developers. Browsers would be well-served to add more
metrics to see what the impact of this kind of approach might look like if we take more things into account (HSTS
with `includeSubdomains` and a sufficiently long lifetime obviates the needs for the `Secure` attribute, for example).

### Doesn't this make users type passwords more often? Isn't that bad?

That would be bad. We should actively discourage folks from typing passwords into non-secure pages. Browsers are
moving on this already by labeling sites as "Not Secure" in various ways when they contain password forms. I
expect that trend to continue.

## Open Questions

1.  **All or nothing**? Is it better to treat a request's cookies as a monolithic set by deleting _all_ of
    them if _any_ cookie is expired due to age? That is, in the 2018-04-02 example above, should
    `http://A.com` receive no cookies at all?
     
    This is fairly draconian, but could be justified by noting that browsers don't understand cookies'
    intent, so ensuring a consistent state is important. Sending one cookie but not another might confuse
    a server in a dangerous way; as an extreme example, `doSecurityChecks=1` might be set once a year,
    while `sessionID` might be set daily.

    Counterpoints in favor of the current proposal include:
    
    1.  A server shouldn't be punished for old cookies it's forgotten about and isn't using.
    2.  Cookies are fragile today, and older cookies will be expired first regardless.

2.  **Should we delete cookies? Or simply hide them?** A variant of this proposal would not remove the cookies
    when they'd be sent over non-secure transport, but instead treat them as though they'd been marked as
    `Secure`, at least for the purposes of building the `Cookie` header. That approach raises a few questions
    which we discussed in an earlier version of this proposal. For example:

    1.  Should we allow the server to overwrite the cookie we didn't send? (e.g. if `name=value` is older than
        the cutoff, would we accept `Set-Cookie: name=other-value` in the response?)

    2.  Should we inform the server that there's a hidden cookie they're not getting (perhaps by sending the
        cookie name and a blanked-out value, like `name=[value-omitted]`).

3.  **Do we need to care deeply about bypasses?** We discuss trivially bypassing this mechanism in #4 above.
    It seems that we could come up with heuristics to poke at that if it becomes common. Is that important?
    Should we care more than this document suggests that we should?

4.  **Should we special-case the cookie value "OPT_OUT"?** It would be unfortunate indeed if removing old cookies
    meant that users who had opted out of interest-based advertising started being targeted again. Perhaps
    excluding the special value `OPT_OUT` (and asking advertisers to standardize on it?) is justifiable.

## Feedback?

[Blink's "Intent to Deprecate: Nonsecurely delivered cookies." thread][8] is a good place to start.

[1]: https://tools.ietf.org/html/rfc7258
[2]: https://www.eff.org/deeplinks/2013/12/nsa-turns-cookies-and-more-surveillance-beacons
[3]: https://tools.ietf.org/html/draft-ietf-httpbis-rfc6265bis#section-5.3
[4]: https://tools.ietf.org/html/draft-ietf-httpbis-rfc6265bis#section-5.4
[5]: https://tools.ietf.org/html/draft-thomson-http-omnomnom-00
[6]: https://bugzilla.mozilla.org/show_bug.cgi?id=1160368
[7]: https://transparencyreport.google.com/https/overview
[8]: https://groups.google.com/a/chromium.org/forum/#!topic/blink-dev/r0UBdUAyrLk
