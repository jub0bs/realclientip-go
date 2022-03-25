[![GoDoc](https://godoc.org/github.com/realclientip/realclientip-go?status.svg)](http://godoc.org/github.com/realclientip/realclientip-go)
[![Test](https://github.com/realclientip/realclientip-go/actions/workflows/test.yml/badge.svg)](https://github.com/realclientip/realclientip-go/actions/workflows/test.yml)
![coverage](https://img.shields.io/badge/coverage-100%25-success?style=flat)
![license](https://img.shields.io/badge/license-Unlicense-important.svg?style=flat)

# realclientip-go

`X-Forwarded-For` and other "real" client IP headers are [often used incorrectly](https://adam-p.ca/blog/2022/03/x-forwarded-for/), resulting in bugs and security vulnerabilities. This library is an attempt to create a reference implementation of the correct ways to use such headers.

This library is written in Go, but the hope is that it will be reimplemented in other languages. Please open an issue if you would like to create such an implementation.


This library is freely licensed. You may use it as a dependency or copy it or modify it or anything else you want. It has no dependencies, is written in pure Go, and supports Go versions as far back as 1.13.

## Usage

This library provides strategies for extracting the desired "real" client IP from various headers or from `http.Request.RemoteAddr` (the client socket IP). 

```golang
clientIPStrategy, err := realclientip.RightmostTrustedCountStrategy("X-Forwarded-For", 1)
...
clientIP := clientIPStrategy(req.Header, req.RemoteAddr)
```

Try it out [in the playground](https://go.dev/play/p/_NiOh3WF0-3).

There are a number of different strategies available -- the right one will depend on your network configuration. See the [documentation](https://pkg.go.dev/github.com/realclientip/realclientip-go) to find out what's available and which you should use.

There are some more extensive examples of use in the [`_examples` directory](/_examples/).

### Strategy failures

The Strategy used must be chosen and tuned for your network configuration. This _should_ result in the Strategy _never_ returning an empty string -- i.e., never failing to find a candidate for the "real" IP. Consequently, getting an empty-string result should be treated as an application error, perhaps even worthy of panicking.

For example, if you have 2 levels of trusted reverse proxies, you would probably use `RightmostTrustedCountStrategy` and it should work every time. If you're directly connected to the internet, you would probably use `RemoteAddrStrategy` or something like `ChainStrategies(LeftmostNonPrivateStrategy(...), RemoteAddrStrategy)` and you will be sure to get a value every time. If you're behind Cloudflare, you would probably use `SingleIPHeaderStrategy("Cf-Connecting-IP")` and it should work every time.

So if an empty string is returned, it is either because the Strategy choice or configuration is incorrect, or your network configuration has changed. In either case, immediate remediation is required.

### Headers

Leftmost-ish and rightmost-ish strategies support the `X-Forwarded-For` and `Forwarded` headers.

`SingleIPHeaderStrategy` supports any header containing a single IP address or IP:port. For a list of some common headers, see the [Single-IP Headers wiki page](https://github.com/realclientip/realclientip-go/wiki/Single-IP-Headers).

### IPv6 zones

IPv6 zone identifiers are retained in the IP address returned by the strategies. [Whether you should keep the zone](https://adam-p.ca/blog/2022/03/strip-ipv6-zone/) depends on your specific use case. As a general rule, if you are not immediately using the IP address (for example, if you are appending it to the `X-Forwarded-For` header and passing it on), then you _should_ include the zone. This allows downstream consumers the option to use it. If your code is the final consumer of the IP address, then keeping the zone will depend on your specific case (for example: if you're logging the IP, then you probably want the zone; if you are rate limiting by IP, then you probably want to discard it).

To split the zone off and discard it, you may use `realclientip.SplitHostZone`.

### Known IP ranges

There is a copy of [Cloudflare's IP ranges](https://www.cloudflare.com/ips/) under `realclientip.CloudflareIPRanges`. This can be used with `realclientip.RightmostTrustedRangeStrategy`. We may add more known cloud provider ranges in the future. Contributions are welcome to add new providers or update existing ones.

(It might be preferable to use [provider APIs](https://api.cloudflare.com/#cloudflare-ips-properties) to retrieve the ranges, as they are guaranteed to be up to date.)

## Implementation decisions and notes

### `net` vs `netip`

At the time of writing this library, Go 1.18 was only just released. It made sense to use the older `net` package rather than the newer `netip`, so that the required Go version wouldn't be so high as to exclude some users of the library.

In the future we may wish to switch to using `netip`, but it will require API changes to `AddressesAndRangesToIPNets`, `RightmostTrustedRangeStrategy`, and `ParseIPAddr`.

### Disallowed valid IPs

The values `0.0.0.0` (zero) and `::` (unspecified) are valid IPs, strictly speaking. However, this library treats them as invalid as they don't make sense to its intended uses. If you have a valid use case for them, please open an issue.

### Normalizing IPs

All IPs output by the library are first convert to a structure (like `net.IP`) and then stringified. This helps normalize the cases where there are multiple ways of encoding the same IP -- like `192.0.2.1` and `::ffff:192.0.2.1`, and the various zero-collapsed states of IPv6 (`fe80::1` vs `fe80::0:0:0:1`, etc.).

### Input format strictness

Some input is allowed that isn't strictly correct. Some examples:

* IPv4 with brackets: `[2.2.2.2]:1234`
* IPv4 with zone: `2.2.2.2%eth0`
* Non-numeric port values: `2.2.2.2:nope`
* `Forwarded` header with too much space: `For= 2.2.2.2`
* `Forwarded` header no quotes around IPv6 values: `For=[2001:db8:cafe::18]:1234`
* `Forwarded` header components that are invalid (as long as `For=` is also present): `For="2001:db8:cafe::18";Nope=what`

It could be argued that it would be better to be absolutely strict in what is accepted.

### Code comments

As this library aspires to be a "reference implementation", the code is heavily commented. Perhaps more than is strictly necessary.

### Pre-creating Strategies

Most of the strategies are created by calling a function, like `RightmostTrustedCountStrategy("Forwarded", 2)`. That can make it awkward to create-and-call at the same time, like `RightmostTrustedCountStrategy("Forwarded", 2)(r.Header, r.RemoteAddr)`. We could have instead implemented non-pre-created functions, like `RightmostTrustedCountStrategy("Forwarded", 2, r.Header, r.RemoteAddr)`. The reasons for the way we did it include:
1. A consistent call signature -- i.e., the `Strategy` type. This enables `ChainStrategies`.
2. Pre-creation allows us to put as much of the invariant processing as possible into the creation step. (Although, in practice, so far, this is only the header name canonicalization.)
3. No error return is required from the strategies. (Although they can -- but should not -- return empty string.) All error-prone processing is done in the pre-creation.

An alternative approach could be using functions like:

```
func RightmostTrustedCountStrategy(headerName string, trustedRanges []*net.IPNet, headers http.Header, remoteAddr string) (Strategy, ip, error) {
...
strat, _, err := RightmostTrustedRangeStrategy("Forward", 2, "", "")              // pre-create
_, ip, err := RightmostTrustedRangeStrategy("Forward", 2, r.Header, r.RemoteAddr) // use direct
```

But perhaps that's no less awkward.

## Other language implementations

If you want to reproduce this implementation in another language, please create an issue and we'll make a repo under this organization for you to use.
