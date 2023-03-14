# Draft: uri-list scheme

## Abstract

Defines a new URI scheme called ‘uri-list’. Applications can use this to link multiple resources from a single anchor or request.

## Description

It’s common for users of applications to work with multiple concurrent resources. Web browsers have long enabled presenting multiple resources within a single window though “tabs” and more recently they’ve allowed users to organize related tabs into “tab groups”. However, linking on the web is still largely one-to-one. If a user wants to share multiple resources, they must send a list of individual URIs for the recipient to click through each one.

The uri-list scheme enables users and agents to link to multiple resources with this form:
	
	uri-list:uri1;uri2;uri3;...

Agents acting on such a URI may, with a single user action, dereference all the specified resources and group them conveniently.

uri-list has no relative forms and is similar to other inline schemes such as data: and javascript:

### Motivation

As noted in WebArch, introducing a new URI scheme is costly. This section explains the advantages of a new scheme over alternative approaches for the expected use cases.

There are other ways to provide a list of resources for a user agent, some examples:

  * A new HTML element (e.g. <anchor-list>)
  * A new attribute on the existing <a> element.
  * A plain HTTP URI to a server responding with a Content-Type: text/uri-list header and a list of URIs in the response body.

All of these have significant drawbacks:

  * Requires a user to author a document or host a server. This requires time and technical skill, excluding the majority of the web’s users.
  * Uses an extra layer of indirection so its lifetime additionally relies on the intermediate server i.e. if the server is ever decommissioned or moved to a new URI, the link is broken.
  * A user sharing such a link must now trust whoever hosts their document or serves the response, making it more difficult to share a link privately.

There are good reasons why users may still wish to add a layer of indirection, e.g. to shorten a long list into a URI convenient for sharing, or to enable detection of non-supporting agents and provide an alternate representation. However, this choice should be left to users and applications.

Another method that, like the uri-list scheme, avoids indirection could be to use a data URI (defined in RFC2397) with a text/uri-list mediatype:

```
data:text/uri-list,https://example.com/1%0Dhttps://example.com/2
```

However, this introduces some usability issues that are more easily addressed by a new scheme:

* Users of plaintext communication apps (e.g. instant messaging, e-mail, etc.) often rely on the app “linkifying” a plaintext uri so the recipient can open it with a click. This won’t happen for data: URIs. While a new uri-list scheme would also require updates in such apps to support linkification, uri-list: is less flexible than data:, making this more straightforward for linkifier software.
* data: URIs can contain any kind of data, including executable script. Users can (and should) be weary of clicking such links. Promoting such links might unintentionally train users to be more trusting of potentially unsafe data: links.
* Because data: URIs can be dangerous, they’re explicitly blocked in certain contexts. For example, most web browsers block data: URIs in anchor links. Some block HTTP redirects to data URIs.
* The text/uri-list mediatype uses newlines as a delimiter and accepts comments, making for messier links. A uri-list scheme allows a friendlier syntax and mapping.

The uri-list scheme is semantically equivalent to data:text/uri-list, albeit with a dedicated syntax and more user-friendly form. When an agent operates on a `uri-list`, it will interpret the inline data in the URL as a text/uri-list mediatype.


### Use Cases

* User-to-user sharing - a user wishing to share a collection of pages with a friend can use a `uri-list` link. For example, when deciding where to eat with a friend, a user can share a list of restaurant options over a chat app and ask for an opinion.
* Authoring - page can provide a `uri-list` in an anchor link to allow users to open a group of pages. For example, an online retailer can easily provide a comparison option allowing a user to open multiple product pages.
* A standard interchange format - Many applications can produce or ingest multiple resources. For example, consider a photo album app that allows adding images by a user-provided URI. Such apps may already allow some non-standard way to specify multiple images at once, perhaps via a comma-separated list; however, a comma is a valid character in a URI path which could corrupt the list. `uri-list` could be a standard way to do this and would have the added benefit that other software could produce such a list in the same format (e.g. a user’s storage provider service outputting a `uri-list` to the user’s desired photos).

### Example

```
uri-list:https://en.wikipedia.org/wiki/Cat;https://en.wikipedia.org/wiki/Dog;https://en.wikipedia.org/wiki/Parrot
```


## Syntax

A valid uri-list scheme URI consists of the scheme name “uri-list” followed by a colon “:” followed by the URI components “hier-part”, “query”, and “fragment” as defined by RFC3986.

The “hier-part” component of a uri-list scheme URI consists of a non-empty list of individual URIs, separated by semi-colons “;”. Individual URIs in the list containing a semicolon “;”, question mark “?”, or number sign “#” must have these characters percent-encoded to prevent conflicting with their use as delimiters in the containing uri-list URI.

The uri-list syntax does not place any requirements on the individual URIs syntax beyond restricting them to the valid set of characters for a URI. That is, a valid uri-list may contain invalid URIs as elements.

```
uri-list  = “uri-list:” uri-data *( “;” uri-data  ) [ “?” query ] [ “#” fragment ]
uri-data  = 1*urichar
uri-char  = unreserved / pct-encoded / gen-delims-limited / sub-delims-limited

; TODO: Maybe we should support (at least as a potential extension) URIs such as:
; uri-list://example.com/uri-list which is an instruction to do a GET from
; example.com/uri-list and expect the result to be content-type: text/uri-list.
; We’d need some restrictions on the uri-data syntax to differentiate this from a 1 element 
; list containing “//example.com/uri-list”, perhaps requiring each uri-data to begin with a 
; scheme.

gen-delims-limited  = ":" / "/" / "[" / "]" / "@"       ; gen-delims with ‘?’ and ‘#’ removed
sub-delims-limited  = "!" / "$" / "&" / "'" / "("       ; sub-delims with ‘;’ removed
                    / ")" / "*" / "+" / "," / "="

; imported from RFC3986
pchar         = unreserved / pct-encoded / sub-delims / ":" / "@"
query         = *( pchar / "/" / "?" )
fragment      = *( pchar / "/" / "?" )
pct-encoded   = "%" HEXDIG HEXDIG
unreserved    = ALPHA / DIGIT / "-" / "." / "_" / "~"
```

## Operations
text/uri-list content retrieval

The only operation defined on a uri-list URI is to retrieve the text/uri-list representation of the list.

As the data is provided inline in the URI there is no need to “resolve”, in the sense of RFC2986-1.2.2, the URI. An agent directly dereferences the URI by executing the algorithm of section 5, which maps the uri-list syntax to a `text/uri-list` resource.


## Mapping from `uri-list` URIs to `text/uri-list`

This section defines an algorithm that maps a uri-list URI into a text/uri-list resource.

1. Let uriList be a URI being parsed
    ASSERT(uriList is a valid URI per the URI General Syntax)
2. Let listData be the hier-part component of uriList as parsed per the URI General Syntax.
3. Let items be the list that results from performing a string split on listData using the semicolon “;” as a delimiter.
4. Let output be an empty string
5. For each element item of items perform the following steps:
    1. If output is non-empty, append a new line by appending a CRLF character pair (U+000D, U+000A ) to output.
    2. Percent-decode any instances of semicolon “;”, question mark “?”, and number sign “#”. That is, in item, replace each 
    case-insensitive instance of:
        * “%3B” with “;”
        * “%3F” with “?”
        * “%23” with “#”
    3. Append item to output.


## Security Considerations

### Abuse

In some contexts, such as web browsing, a link's ability to open multiple pages can be an abuse vector. For example, making it difficult for a user to close a given page, showing them lots of unwanted ads, or sharing a user identifier across third-party sites. The most common web browsers now include “popup blockers” to limit how many new windows a page can open and when. User agents should make it clear to users when a link will open multiple resources and provide users with sufficient controls to prevent unwanted behavior.

### Scaling Leaks

In limited circumstances, opening a resource may allow a “one-bit leak”, where an attacker can glean some information about the user-specific data within the resource. For example, fragment navigation may allow an attacker to tell whether a given id/name appears on a destination page. These are typically of limited utility for attackers as they’re limited to a single bit of information but the ability to open many pages from a single click could be used to scale these attacks.

### Global Resource Exhaustion

Opening many resources could allow an attacker to exhaust a user’s system resources for malicious purposes. For example, some user agents place resources from different sources into separate operating system processes to leverage the operating system’s isolation benefits. By exhausting the process limit an attacker could force a victim resource into sharing a process with their own malicious resource. Another example is the connection pool limit. The ability to open multiple resources could render the user agent’s defenses against these kinds of attack ineffective.

### Social Engineering

“Phishing” is a common technique used by attackers to impersonate a trusted site to deceive a user into performing an unsafe action. For example, to provide their secret credentials or payment information. An attacker could use a uri-list to conceal a deceptive page by hiding it in a group of legitimate pages, hoping the user will not scrutinize every page.


## IANA Considerations

This document registers the ‘uri-list' scheme as a permanent scheme in the Uniform Resource Identifier scheme registry as per [BCP0035].

## Length Considerations

Real world URIs are already often uncomfortably long; concatenating them into a single URI may result in an unwieldy identifier. Creators of such links should consider the usability of such URIs and whether they’d be better served via a response body.

Besides usability, many applications limit the length of a URI. RFC9110 recommends implementers to support at least 8000 characters, however, legacy software can have limits much lower than this [link]. Although legacy user agents aren’t a concern, since they won’t understand the scheme anyway, and the URI is meant to be dereferenced locally, there are situations where a uri-list URI might be passed through intermediate entities. For example, as a location header field in a 3xx Redirection response, or as an attribute in an HTML document. 


## Future extension

Should multi-resource links become common, applications may wish to enable enhancements to make them more useful. This section explores some potential ideas and how uri-list could be adapted in a backwards-compatible way. It is meant for illustrative purposes only and must not be interpreted as conformance requirements of this specification.

### Configuration options

As user agents evolve to support this use case their UI could allow for richer forms of display. For example, grouping common pages together in different ways with different decorations, titles, etc. Users may wish to provide a title to a uri-list to distinguish them in a busy UI (e.g. “RFCs to read”, “Restaurant Recommendations”, etc.). Authors may wish to provide a hint to the agent as to how best to display the resources (e.g. “Grid view”, “Carousel”, etc.).

Uri-list can provide options using the query string, resulting in a URI of this form:

```
uri-list:https://a.com;https://b.com;https://c.com?type=grid-view&title=My%20example
```

Note: the query string is unambiguously part of the uri-list rather than c.com since the question-mark ‘?’ character in component URIs must be percent encoded.

The content retrieval operation on the uri-list can map the query parameters into content in the text/uri-list representation. text/uri-list supports comments; one option is to append query parameters to the data in special comments interpreted by implementing user agents and ignored in non-implementing agents. The text/uri-list data in the above example would be:

```
https://a.com
https://b.com
https://c.com
#?type=grid-view
#?title=My example
```

### Hierarchical lists

Resources are often grouped in a nesting hierarchy. For example, most browsers allow storing “bookmarks” in nested folders for organization. Such an application wishing to exchange bookmarks may wish to use a uri-list for this.

This could be achieved by nesting uri-lists as a component URI in the list, taking care to percent-encode the necessary delimiters, and providing the text/uri-list representation with some marker denoting hierarchy. For example: to nest the example URi from 8.1:

```
uri-list:https://foo.com;uri-list:https://a.com%3Bhttps://b.com%3Bhttps://c.com%3Ftype=grid-view&title=My%20example;https://bar.com?title=Bookmarks
```


Could result in text/uri-list data:

```
foo.com
#
https://a.com
https://b.com
https://c.com
#?type=grid-view
#?title=My example
#
bar.com
#?title=Bookmarks
```

Nesting depth could be represented by the number of number-sign ‘#’ characters:

```
#

##
https://…
https://…
#?title=Toronto
##

##
https://…
https://…
#?title=Montreal
##

#?title=Restaurants
```
