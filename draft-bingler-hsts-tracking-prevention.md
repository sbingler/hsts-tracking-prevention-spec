---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "HSTS Tracking Prevention"
category: info

docname: draft-hsts-tracking-prevention-latest
submissiontype: independent  # also: "IETF", "editorial", "IAB", or "IRTF"
date: {DATE}
consensus: false
v: 3
area: sec
keyword:
 - HSTS
 - tracking prevention

author:
 -
    fullname: Steven Bingler
    organization: Google
    email: bingler@google.com

normative:
  RFC6797:

--- abstract

As mentioned in {Section 14.9 of RFC6797}, it's possible for entities in
control of multiple hosts to encode arbitrary data within a user agent's HSTS
policy store. This data can later be retrieved and could be used to identify
users.

This document proposes a change to the user agent processing model of {RFC6797}
to enable user agents to employ tracking mitigation methods.


--- middle

# Introduction

HTTP Strict Transport Security (HSTS), {RFC6797}, improves security for users
by allowing a host to tell the user agent that it should only ever be accessed
over a secure connection.

However, the design of HSTS allows for servers to abuse its state caching in
order to set and retrieve arbitrary data, potentially for the purposes of
tracking a user around the web. {Section 14.9 of RFC6797} warns of this
possibility but does not discuss prevention.

This document offers some minor modifications to {Section 8 of RFC6797}
that allows for user agents to implement a way to prevent tracking of the user.

It also touches on potential implementations.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# HSTS Tracking Prevention

Tracking prevention could take one of a few forms:

* Prevent servers from setting tracking data.

* Prevent servers from retrieving tracking data.

* Make correlating data between sites infeasible.

The first could be implemented by ignoring Strict-Transport-Security response
headers from tracking hosts. This requires user agents to ignore otherwise
valid Strict-Transport-Security response headers if the user agent has reason
to believe that header would be used for tracking purposes.

The second could be implemented by not upgrading URI for tracking hosts.
This requires user agents to not upgrade otherwise eligible URIs to
"https" if the user agent has reason to believe the upgraded URI would be used
for tracking purposes.

The third could be implemented by partitioning the HSTS policy store by the
current site the user is on. This would force any server interested in tracking
a user to set new data on every site, making it infeasible to correlate the
user’s visits between sites via HSTS upgrades.

## Monkey-Patches against RFC6797

### Prevent Servers From Saving Tracking Data

To allow user agents the option of preventing hosts they believe are tracking
the user from setting HSTS data {Section 8.1.1 of RFC6797} should be changed
as follows:

~~~
   If the substring matching the host production from the Request-URI
   (of the message to which the host responded) syntactically matches
   the IP-literal or IPv4address productions from Section 3.2.2 of
   [RFC3986], then the UA MUST NOT note this host as a Known HSTS Host.

   Otherwise, if the substring...
~~~

should be modified to include

~~~
   If the substring matching the host production from the Request-URI
   (of the message to which the host responded) syntactically matches
   the IP-literal or IPv4address productions from Section 3.2.2 of
   [RFC3986], then the UA MUST NOT note this host as a Known HSTS Host.

   If the UA has reason to believe this host intends to set tracking
   data then the UA MAY not note this host as a Known HSTS Host.

   Otherwise, if the substring...
~~~

### Prevent Servers From Retrieving Tracking Data.

To allow user agents the option of preventing hosts they believe are tracking
the user from retrieving HSTS data {Section 8.3 of RFC6797} should be changed
as follows:

~~~
   1.  Extract from the URI any substring described by the host
       component of the authority component of the URI.

   2.  If the substring is null, then there is no match with any Known
       HSTS Host.
~~~

should be modified to include

~~~
   1.  Extract from the URI any substring described by the host
       component of the authority component of the URI.

   2.  If the UA has reason to believe this host intends to retrieve tracking
       data then the UA MAY abort this algorithm.

   2.  If the substring is null, then there is no match with any Known
       HSTS Host.
~~~

### Make Correlating Data Between Sites Infeasible

Making data correlation infeasible for tracking servers could be implemented a
number of ways, but a straightforward design would be to keep the HSTS policies
of different sites separated, so that even if tracker.example is embedded into
both site.example and othersite.example it cannot easily track users between
those sites. Doing this is frequently referred to as partitioning.

This can be allowed by modifying the storage and indexing requirement of
{Section 5.3 of RFC6797} to allow for partitioning of the HSTS policy Store.

~~~
   UAs store and index HSTS Policies based strictly upon the domain
   names of the issuing HSTS Hosts.

   This means that UAs will maintain the HSTS Policy of any given HSTS
   Host separately from any HSTS Policies issued by any other HSTS Hosts
   whose domain names are...
~~~

Should be modified to include

~~~
   UAs store and index HSTS Policies based strictly upon the domain
   names of the issuing HSTS Hosts.

   Note: UAs MAY decide to partition their HSTS policies by other inputs such
   as the top-level site the agent has been navigated to. In this case the UA
   should store and index HSTS Policies within that partition based strictly
   upon the domain names of the issuing HSTS Hosts.

   This means that UAs will maintain the HSTS Policy of any given HSTS
   Host separately from any HSTS Policies issued by any other HSTS Hosts
   whose domain names are...
~~~

{Section 8.2 of RFC6797} should receive a similar change

~~~
   A given domain name may match a Known HSTS Host's domain name in one
   or both of two fashions: a congruent match, or a superdomain match.
   Alternatively, there may be no match.

   The steps below determine whether there are any matches, and if so,
   of which fashion:
~~~

Should be modified to include

~~~
   A given domain name may match a Known HSTS Host's domain name in one
   or both of two fashions: a congruent match, or a superdomain match.
   Alternatively, there may be no match.

   Note: If the UA has decided to partition its HSTS Policies then the
   following algorithm MUST be performed only on HSTS Policies within the
   applicable partition.

   The steps below determine whether there are any matches, and if so,
   of which fashion:
~~~

# Security Considerations

Depending on the tracking prevention technique employed the UA may risk
reducing the security of HTTP requests. Many UAs however implement some form
of mixed content blocking and upgrading along with automatic upgrades of
navigations to https top-level sites. These behaviors help limit any potential
security decreases.

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
