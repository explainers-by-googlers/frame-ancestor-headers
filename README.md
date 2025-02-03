# Frame Ancestor Headers Explainer

This proposal is an early design sketch to describe the problem below and solicit
feedback on the proposed solution. It has not been approved to ship in Chrome.

## Authors

- [Sam LeDoux](https://github.com/sjledoux)

## Participate
- https://github.com/explainers-by-googlers/frame-ancestor-headers/issues


## Table of Contents
<!-- Update this table of contents by running `npx doctoc README.md` -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Motivation](#motivation)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [Use cases](#use-cases)
  - [Partitioned Cookies](#partitioned-cookies)
  - [Storage Access Headers](#storage-access-headers)
  - [Without Third-Party Cookie Blocking](#without-third-party-cookie-blocking)
- [Potential Solution](#potential-solution)
- [Considered alternatives](#considered-alternatives)
  - [Using a Single Header](#using-a-single-header)
  - [Using a Dictionary](#using-a-dictionary)
- [References](#references)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Motivation

With the addition of the [`Sec-Fetch-Site` Fetch Metadata Header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Sec-Fetch-Site), user agents began providing an indication of what relationship a request’s destination has with its initiator. This is vital for servers assessing how to handle a request; however, it provides an incomplete picture. `SameSite` cookie rules, for example, take into account the relationship between the origins of the request's destination, its initiator, and all of the initiator's ancestors.

## Goals

The goals of this proposal are to:
- Introduce a fetch metadata header that conveys the relationship between a request’s destination and its ancestor frames.
- Introduce a fetch metadata header that conveys the relationship between a request’s destination and its top-level frame. 

## Non-goals

This proposal does not aim to:
- Replace any existing fetch metadata headers.

## Use cases

### Partitioned Cookies

Consider [cookie partitioning](https://github.com/privacycg/CHIPS): cookies set with the `Partitioned` attribute are partitioned based on both their top-level context and [their relation to their ancestor frames](https://github.com/privacycg/CHIPS/issues/40#issuecomment-1883726735). Currently, because there is no signal indicating how a request’s destination is related to its ancestors, a site may not be aware of how its cookies will be partitioned when it sets them.

Take the case of a site, `final.example.com`, which sets a cookie in two different contexts using`Set-Cookie: example-cookie=foo; SameSite=None; Partitioned`:

1. A user navigates to `top-level.example.com`, which embeds a frame from `initiator.example.com`, which embeds the frame `final.example.com`, which sets `example-cookie`.
   ![`example-cookie` set for `final.example.com` embed](/assets/images/PartitionedCookiesScen1.jpg)
   
   In this case, request headers for a subsequent navigation to `top-level.example.com` would include `example-cookie`, because the frame’s ancestor chain is same-site with the frame’s source, and thus its partition key does not have an “ancestor chain bit” .

   ![request to `top-level.example.com` with `example-cookie`](/assets/images/PartitionedCookiesHeaders1.jpg)

2. A user navigates to `top-level.example.com`, which embeds a frame from `another-site.com`, which embeds a frame from `initiator.example.com`, which embeds the frame `final.example.com`, which sets `example-cookie`.

   ![`example-cookie` set `final.example.com` embed with cross-site ancestor](/assets/images/PartitionedCookiesScen2.jpg)
   
   In this case, a subsequent navigation to `top-level.example.com` would not include `example-cookie` as the frame had a cross-site ancestor in its ancestor chain, and therefore `example-cookie` had an “ancestor chain bit” in its partition key.

   ![request to `top-level.example.com` without `example-cookie`](/assets/images/PartitionedCookiesHeaders2.jpg)

There is no indication to `final.example.com` of how the partitioning of these cookies will differ. That is, case 1 will have the `cross-site-ancestor` bit as `false`, while case 2 will have the bit as `true`. Other requests that have the origin of  `example.com` and a top-level document of `top-level.example.com` will only include `example-cookie` in their `cookie-list` if their cross-site ancestor bit matches that of the cookie’s partition. An indication of the existence of cross-site ancestors, to a request’s destination, may be helpful to sites that want to understand how cookies set with the `Partitioned` attribute will be partitioned, depending on what contexts they would like to have access to those cookies in the future.


### Storage Access Headers
A similar problem occurs for user agents which implement [the `Sec-Fetch-Storage-Access` fetch metadata header](https://github.com/privacycg/storage-access-headers), wherein a top-level document whose source is origin `example.com` makes a cross-site request to a document at `another-site.com`, which itself makes a cross-site request to `example.com`. In this scenario, when `SameSite=None` cookies are blocked, the final request to A will include the header `Sec-Fetch-Storage-Access: inactive`. Due to the nature of the Storage Access API and the `Sec-Fetch-Storage-Access` fetch metadata header, it is not clear to `example.com` whether or not they are receiving an `inactive` header value because the user has previously allowed storage access for this context in another frame, or if the request is originating from an A>B>A context and thus can be autogranted storage access. Servers may want to treat these scenarios differently to prevent CSRF attacks, in which case a signal of the relationship between a request’s destination and its top-level document would be greatly beneficial to differentiate between the two possibilities.

![A(B(A)) embed with `Sec-Fetch-Storage-Access: inactive` header](/assets/images/SAHScen.jpg)

### Without Third-Party Cookie Blocking
Even when third-party cookie blocking is not enabled, there is ambiguity on the inclusion/absence of a cookie on a request. Consider a scenario where a user visits the site `example.com` in a top level context, and the site sets a cookie, `example-cookie=foo; SameSite=None`. Later on, the user visits a page that embeds `initiator.example.com`, and that embed sends a request to the destination of `final.example.com`. If the top-level document for this request is `top-level.example.com`, then `example-cookie` will be included in the request’s `Cookie` header, as the ["site for cookies"](https://www.ietf.org/archive/id/draft-ietf-httpbis-rfc6265bis-16.html#section-5.2) matches the request’s destination. 

![request to embedded `final.example.com` with `example-cookie`](/assets/images/TPCAllowedScen1.jpg)

However, if the top-level document is instead `another-site.com`, then `example-cookie` will not be included in the `Cookie` header. 

![request to embedded `final.example.com` without `example-cookie`](/assets/images/TPCAllowedScen2.jpg)

In both requests, the value of `Sec-Fetch-Site` will be `same-site`, which does not help the server understand <b>why it does not have access</b> to that particular unpartitioned cookie in the latter example, and <b>why it does</b> in the former. A signal of the relationship between a request’s destination and all of its ancestor documents would therefore be helpful for servers, as this is the criterion used when calculating the [“site for cookies”](https://www.ietf.org/archive/id/draft-ietf-httpbis-rfc6265bis-16.html#section-5.2) of a request.

<!-- In your initial explainer, you shouldn't be attached or appear attached to any of the potential
solutions you describe below this. -->

## Potential Solution

The Frame Ancestors Headers feature proposes the addition of two headers which would help provide additional information about the full context of an embedded request when it is sent to a server. The headers would be as follows:
1. `Sec-Fetch-Ancestors` - This header exposes whether an embedded request is cross-site or cross-origin with any of its ancestor documents.

   The following are the enumerated values that the header may hold:
   - `same-origin`: 
    The request’s destination has the same origin as all of its ancestors.
   - `same-site`:
    The request’s destination is cross-origin to one or more of its ancestors, but same-site with all of them.
   - `cross-site`:
    The request’s destination is cross-site to one or more of its ancestors. 
2. `Sec-Fetch-Top-Frame` - This header exposes the relationship between an embedded request’s destination and its top-frame document.

   The following are the enumerated values that the header may hold:
   - `same-origin`:
    The request has the same origin as the top-level document.
   - `same-site`:
    The request is cross-origin to the top-level document.
   - `cross-site`:
    The request is cross-site to the top-level document.

Requests with these proposed headers would be as follows for a navigation that has a cross-site top-frame ancestor:

![request to `final.example.com` contains `Sec-Fetch-Ancestors: cross-site` and `Sec-Fetch-Top-Frame: cross-site` headers](/assets/images/TPCSolution.jpg)

`final.example.com` is provided with a signal that allows it to understand the absence of `example-cookie` from the `Cookie` header.

Behavior for the A(B(A)) case would be as follows:

![request to `example.com` contains `Sec-Fetch-Top-Frame: same-origin` header](/assets/images/SAHSolution.jpg)

`example.com` is provided with context to determine how it wants to handle storage access for this request. 

The following would be seen when there is a higher order cross-site ancestor for an iframe reading a partitioned cookie:

![request to `final.example.com` contains `Sec-Fetch-Ancestors: cross-site` and `Sec-Fetch-Top-Frame: same-site` headers](/assets/images/CHIPSSolution.jpg)

`final.example.com` is provided with an indication that the `cross-site-ancestor` bit will be `true` for cookies set with the `Partitioned` attribute.

The following table compares the `Sec-Fetch-Site` header with the proposed headers in different scenarios, where: 
- A and B are cross-site.
- A and A* are same-site.
- A and A* are cross-origin.
- -> denotes an embedding relationship (A->B means A embeds B)
- The header values in the table represent what the innermost frame would receive in its document request.

| Scenario | Sec-Fetch-Site | Sec-Fetch-Ancestors | Sec-Fetch-Top-Frame |
| -------- | -------------- | ------------------- | ------------------- |
| A->A->A  | Same-Origin    | Same-Origin         | Same-Origin         |
| A->A->B  | Cross-Site     | Cross-Site          | Cross-Site          |
| A->B->A  | Cross-Site     | Cross-Site          | Same-Origin         |
| A->B->B  | Same-Origin    | Cross-Site          | Cross-site          |
| A->A->A* | Same-Site      | Same-Site           | Same-Site           |
| A->A*->A | Same-Site      | Same-Site           | Same-Origin         |
| A->A*->A* | Same-Origin   | Same-Site           | Same-Site           |


## Considered alternatives

### Using a Single Header

Rather than introducing two separate headers, one could imagine a scenario where a user-agent chooses to emit a single header that conveys information about both a request’s frame ancestors and their top-level relation. This header could have the following values:
- `same-origin`: The request’s destination has the same origin as all of its ancestors (including the top-level).
- `same-site`: The request’s destination is cross-origin to one or more of its ancestors (including the top-level).
- `cross-site:`
The request’s destination is cross-site to one or more of its ancestors (including the top-level). 
- `same-site-with-same-origin-top`:
The request’s destination is cross-origin to one or more of its ancestors, but has the same origin to its top-level document.
- `cross-site-with-same-origin-top`:
The request’s destination is cross-site to one or more of its ancestors, but has the same origin to its top-level document.
- `cross-site-with-same-site-top`:
The request’s destination is cross-site to one or more of its ancestors, but is same-site with its top-level document.

However, this solution would be inconvenient for many sites as their server logic will have to map different header values to singular pieces of behavior depending on the type of relation they care about. Given that, it was decided that two separate headers would be the easiest solution.

Consider a site that cares about whether it is same-origin with the top-level document of any requests it receives. In the scenario where a single header is used, the server’s logic will have to check whether the head matches any of these values: `same-origin`, `same-site-with-same-origin-top`, and `cross-site-with-same-origin-top`. In the case where two headers are used, this site would only have to check for the presence of `Sec-Fetch-Top-Frame: same-origin`. 

Similarly, a site which only cares whether it is cross-site with any of its ancestors would have to parse for the header values `cross-site`, `cross-site-with-same-origin-top` and `cross-site-with-same-site-top` if there were a single header, rather than just checking for the presence of `Sec-Fetch-Ancestors: cross-site`. 


### Using a Dictionary
One could also imagine a similar single header approach, which uses a header whose value is dictionary that maps an `all` key to a value that is equivalent to the proposed `Sec-Fetch-Ancestors` header, and a `top` key to a value that is equivalent to the proposed  `Sec-Fetch-Top-Frame` header. 

While a dictionary approach does simplify the parsing a site would need to do to obtain their header of interest, this solution would suffer from [the same header bloat that was described in the Fetch Metadata Request Headers spec](https://w3c.github.io/webappsec-fetch-metadata/#bloat). Due to the way that HTTP headers are compressed, using multiple headers to represent multiple values is often preferred to using a single dictionary, as [the dictionary approach tends to result in the request headers taking up more bytes of space than if they were to be split into multiple headers](https://www.mnot.net/blog/2018/11/27/header_compression). As such, we concluded that using two different headers to represent the two different relationships would be the best approach.

## References
- [CHIPS Ancestor Frame Bit · Issue #40 · privacycg/CHIPS](https://github.com/privacycg/CHIPS/issues/40#issuecomment-1883726735)
- [Header bloat](https://w3c.github.io/webappsec-fetch-metadata/#bloat)
- [Header compression](https://www.mnot.net/blog/2018/11/27/header_compression)
- [privacycg/CHIPS](https://github.com/privacycg/CHIPS)
- [privacycg/storage-access-headers](https://github.com/privacycg/storage-access-headers)
- [`Sec-Fetch-Site` header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Sec-Fetch-Site)
- [Site for Cookies Spec](https://www.ietf.org/archive/id/draft-ietf-httpbis-rfc6265bis-16.html#section-5.2)

