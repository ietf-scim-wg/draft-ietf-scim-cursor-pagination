---
title: "Cursor-based Pagination of SCIM Resources"
abbrev: "SCIM Cursor Pagination"
docname: draft-ietf-scim-cursor-pagination-08




category: std

ipr: trust200902
area: IETF
workgroup: SCIM
keyword: [Internet-Draft, SCIM]

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]
submissionType: IETF

author:
   -
    name: Matt Peterson
    organization: Entrust
    email: matt.peterson@entrust.com
   -
    name: Danny Zollner
    organization: Microsoft
    email: danny.zollner@microsoft.com
    role: editor
   -
    name: Anjali Sehgal
    organization: Amazon Web Services
    email: anjalisg@amazon.com

normative:
  RFC3986:
  RFC6585:
  RFC7643:
  RFC7644:

informative:
  BCP195:
  RFC9110:

--- abstract

This document defines additional SCIM (System for Cross-Domain Identity Management) query parameters and result
attributes to allow use of cursor-based pagination in SCIM
implementations that are implemented with existing code bases,
databases, or APIs where cursor-based pagination is already well established.


--- middle

# Introduction

The two common patterns for result pagination are index-based pagination
and cursor-based pagination.  Rather than
attempt to compare and contrast the advantages and disadvantages of
competing pagination patterns, this document simply recognizes that
SCIM (System for Cross-Domain Identity Management) service providers are commonly implemented as an
interoperability layer on top of already existing application
codebases, databases, and/or APIs that already have a well established pagination pattern.

Translating from an underlying cursor-based pagination pattern to the
index-based pagination defined in Section 3.4.2.4 of [RFC7644]
ultimately requires the SCIM service provider to fully iterate the
underlying cursor, store the results, and then serve indexed pages
from the stored results.  This task of "pagination translation"
dramatically increases complexity and memory requirements for
implementing a SCIM service provider, and may be an impediment to
SCIM adoption for some applications and identity systems.

This document defines a simple addition to the SCIM protocol that
allows SCIM service providers to reuse underlying cursors without
expensive translation.  Support for cursor-based pagination in SCIM
encourages broader cross-application identity management
interoperability by encouraging SCIM service provider implementations
for applications and identity systems where cursor-based pagination
is already well-established.

## Notational Conventions

{::boilerplate bcp14-tagged}

# Query Parameters and Response Attributes

The following table describes the URL pagination query parameters for requesting cursor-based pagination:

| Parameter | Description |
| `cursor` | The string value of the nextCursor attribute from a previous result page. The cursor value MUST be empty or omitted for the first request of a cursor-paginated query. This value may only contain characters from the unreserved characters set defined in section 2.3 of [RFC3986]. |
| `count` | A positive integer. Specifies the desired maximum number of query results per page, e.g., `count=10`. When specified, the service provider MUST NOT return more results than specified, although it MAY return fewer results. If count is not specified in the query, the maximum number of results is set by the service provider.
{: title="Query Parameters"}

The following table describes cursor-based pagination attributes returned in a paged query response:

| Element | Description |
| `nextCursor` | A cursor value string that MAY be used in a subsequent request to obtain the next page of results. Service providers supporting cursor-based pagination MUST include `nextCursor` in all paged query responses except when returning the last page. `nextCursor` is omitted from a response only to indicate that there are no more result pages. |
| `previousCursor` | A cursor value string that MAY be used in a subsequent request to obtain the previous page of results. Returning `previousCursor` is OPTIONAL.
{: title="Response Attributes"}

Cursor values are opaque; clients MUST not make assumptions about their structure. When the client wants to retrieve
another result page for a query, it MUST query the same service
provider endpoint with all query parameters and values being
identical to the initial query with the exception of the cursor value
which SHOULD be set to a `nextCursor` (or `previousCursor`) value that
was returned by service provider in a previous response.

For example, to retrieve the first 10 Users with `userName` starting
with `J`, use an empty cursor and set the count to 10:

~~~
GET /Users?filter=userName%20sw%20J&cursor&count=10
Host: example.com
Accept: application/scim+json
Authorization: Bearer U8YJcYYRMjbGeepD
~~~

The SCIM provider in response to the query above returns metadata regarding pagination similar
to the following example (actual resources removed for brevity):

~~~
HTTP/1.1 200 OK
Content-Type: application/scim+json

{
   "totalResults":100,
   "itemsPerPage":10,
   "nextCursor":"VZUTiyhEQJ94IR",
   "schemas":["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
   "Resources":[{
      ...
   }]
}
~~~

Given the example above, to request the next page or results, use the
same query parameters and values except set the cursor to the value
of `nextCursor` (`VZUTiyhEQJ94IR`):

~~~
GET /Users?filter=username%20sw%20J&cursor=VZUTiyhEQJ94IR&count=10
Host: example.com
Accept: application/scim+json
Authorization: Bearer U8YJcYYRMjbGeepD

HTTP/1.1 200 OK
Content-Type: application/scim+json

{
   "totalResults": 100,
   "itemsPerPage": 10,
   "previousCursor: "ze7L30kMiiLX6x"
   "nextCursor": "YkU3OF86Pz0rGv",
   "schemas": ["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
   "Resources":[{
      ...
   }]
}
~~~

In the example above, the response includes the OPTIONAL
previousCursor indicating that the service provider supports forward
and reverse traversal of result pages.

As described in Section 3.4.1 of [RFC7644] service providers SHOULD
return an accurate value for totalResults which is the total number
of resources for all pages.  Service providers implementing cursor
pagination that are unable to estimate totalResults MAY choose to omit the totalResults attribute.

## Pagination errors

If a service provider encounters invalid pagination query
parameters (invalid cursor value, count value, etc), or other error
conditions, the service provider SHOULD return the appropriate HTTP
response status code and detailed JSON error response as defined in
Section 3.12 of [RFC7644].  Most pagination error conditions would
generate an HTTP response with status code 400.  Since many pagination
error conditions are not user recoverable, error messages SHOULD
focus on communicating error details to the SCIM client developer.

For HTTP status code 400 (Bad Request) responses, the following detail error types are defined. These error types extend the list of error types defined in [RFC7644] Section 3.12, Table 9: SCIM Detail Error Keyword Values.

| `scimType` | Description | Applicability |
| `invalidCursor` | Cursor value is invalid. Cursor value SHOULD be empty to request the first page and set to the `nextCursor` or `previousCursor` value for subsequent queries.| `GET` (Section 3.4.2 of [RFC7644])|
| `expiredCursor` | Cursor has expired. Do not wait longer than service provider's `cursorTimeout` to request additional pages.| `GET` (Section 3.4.2 of [RFC7644])|
| `invalidCount` | Count value is invalid. Count value must be between 1 - and service provider's maxPageSize | `GET` (Section 3.4.2 of [RFC7644])|
{: title="Pagination Errors"}

## Sorting

If sorting is implemented as described Section 3.4.2.3 of [RFC7644],
then cursor-paged results SHOULD be sorted.

When a service provider supports both index- and cursor-based pagination, clients can use the 'startIndex' and 'cursor' query parameters to request a specific method.

Service providers supporting both pagination methods MUST choose a pagination method to use when responding to requests that have not specified a pagination query parameter. Service providers MUST NOT return an error due to the pagination method being unspecified when pagination is required to complete the response.

If the default pagination method is not advertised in the Service Provider Configuration data, service provider implementers MAY dynamically determine which pagination method is used for each response based on criteria of their choosing.

## Cursors as the Only Pagination Method

A service provider MAY require cursor-based pagination to
retrieve all results for a query by including a `nextCursor` value in
the response even when the query does not include the `cursor`
parameter.

For example:

~~~
GET /Users
Host: example.com
Accept: application/scim+json
~~~

The service provider may respond to the above query with a page
containing defaultPageSize results and a `nextCursor` value as shown
in the below example (Resources omitted for brevity):

~~~
HTTP/1.1 200 OK
Content-Type: application/scim+json

{
   "totalResults": 5000,
   "itemsPerPage": 100,
   "nextCursor": "HPq72Pax3JUaNa",
   "schemas": ["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
   "Resources": [{
      ...
   }]
}
~~~

## Backwards Compatibility Considerations

Implementers of SCIM service providers that previously supported
index-based pagination and are adding support for cursor-based pagination
SHOULD carefully consider the impact to existing SCIM clients before
changing the default pagination method in a return set. SCIM clients
that previously expected index-based pagination may not be compatible
with cursor-based pagination without making changes to the SCIM client.
Adding cursor-based pagination support but leaving the default return
set pagination method as-is SHOULD not impact existing SCIM clients.

SCIM clients can query the provider configuration endpoint to determine
if index-based, cursor-based or both types of pagination are supported.

# Querying Resources using HTTP POST

Section 3.4.3 of [RFC7644] defines how clients MAY execute queries without passing parameters on the URL by using the `POST` verb combined with the `/.search` path extension execute. When posting to `/.search`, the client would pass the parameters defined in Section 2 in the body of the POST request.

~~~
POST /User/.search
Host: example.com
Accept: application/scim+json
Authorization: Bearer U8YJcYYRMjbGeepD

{
   "schemas": ["urn:ietf:params:scim:api:messages:2.0:SearchRequest"],
   "attributes": ["displayName", "userName"],
   "filter": "displayName sw \"smith\"",
   "cursor": "",
   "count": 10
}
~~~

Which would return a result containing a `nextCursor` value which may
be used by the client in a subsequent call to return the next page of
resources

~~~
HTTP/1.1 200 OK
Content-Type: application/scim+json

{
   "totalResults": 100,
   "itemsPerPage": 10,
   "nextCursor": "VZUTiyhEQJ94IR",
   "schemas": ["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
   "Resources": [{
      ...
   }]
}
~~~

# Service Provider Configuration

The `/ServiceProviderConfig` resource defined in Section 4 of [RFC7644]
facilitates discovery of SCIM service provider features.  A SCIM
service provider implementing cursor-based pagination SHOULD include
the following additional attribute in JSON document returned by the
`/ServiceProviderConfig` endpoint:

pagination
: A complex type that indicates pagination configuration options. OPTIONAL.  The following sub-attributes are defined:

   cursor
   : A Boolean value specifying support of cursor-based pagination. REQUIRED.

   index
   : A Boolean value specifying support of index-based pagination. REQUIRED.

   defaultPaginationMethod
   : A string value specifying the type of pagination that the service provider defaults to when the client has not specified which method it wishes to use. Possible values are "cursor" and "index".  OPTIONAL.

   defaultPageSize
   : Positive integer value specifying the default number of results returned in a page when a count is not specified in the query. OPTIONAL.

   maxPageSize
   : Positive integer specifying the maximum number of results returned in a page regardless of what is specified for the count in a query. The maximum number of results returned may be further restricted by other criteria. OPTIONAL.

   cursorTimeout
   : Positive integer specifying the minimum number of seconds that a cursor is valid between page requests. Clients waiting too long between cursor pagination requests may receive an invalid cursor error response. No value being specified may mean that there is no cursor timeout or that the cursor timeout is not a static duration.  OPTIONAL.

Service providers may choose not to advertise Service Provider Configuration information regarding default pagination method, page size or cursor validity. Clients MUST NOT interpret the lack of published Service Provider Configuration values to mean that no defaults or limits on page sizes or cursor lifetimes exist, or that there is no default pagination method. Service providers may choose not to publish values for the pagination sub-attributes for many reasons. Examples include:

* Service providers containing multiple resource types may have different values set for each resource type.
* Default and maximum page size may be determined by factors besides or in addition to the number of resources returned, such as the size of each resource on the page.

Before using cursor-based pagination, a SCIM client MAY fetch the
Service Provider Configuration document from the SCIM service
provider and verify that cursor-based pagination is supported.

For example:

~~~
GET /ServiceProviderConfig
Host: example.com
Accept: application/scim+json
~~~

A service provider supporting both cursor-based pagination and index-
based pagination would return a document similar to the following
(full `ServiceProviderConfig` schema defined in Section 5 of [RFC7643]
has been omitted for brevity):

~~~
HTTP/1.1 200 OK
Content-Type: application/scim+json

{
   "schemas": [
      "urn:ietf:params:scim:schemas:core:2.0:ServiceProviderConfig"],
      ...

   "pagination": {
      "cursor": true,
      "index": true,
      "defaultPaginationMethod": "cursor",
      "defaultPageSize": 100,
      "maxPageSize": 250,
      "cursorTimeout": 3600
   },

   ...
}
~~~

Service provider implementors SHOULD ensure that misuse of pagination
by a SCIM client does not deplete service provider resources or
prevent valid requests from other clients being handled.  Defenses
for a SCIM service provider are similar those used to protect other
Web API services -- including the use of a "Web API gateway" layer,
to provide authentication, rate limiting, IP allow/block lists,
logging and monitoring, response caching, etc.

For example, an obvious protection against abuse is for the service
provider to require client authentication in order to retrieve large
result sets and enforce an overriding `totalResults` limit for non-
authenticated clients.  Another example would be for a service
provider that implements cursor pagination to restrict the number of
cursors that can be allocated by a client or enforce cursor lifetimes.

# Security Considerations

This section elaborates on the security considerations associated with the implementation of cursor pagination in SCIM. This document is under the same security and privacy considerations of those described in [RFC7644]. It is imperative that implementers additionally consider the following security aspects to safeguard against both deliberate attacks and inadvertent misuse that may compromise the system's security posture.

## Threat Model and Security Environment

The threat landscape is characterized by two primary types of actors:

1. Unauthenticated and Authenticated Malicious Actors: These individuals or entities represent a malevolent threat. Their objectives include unauthorized access to data, alteration, or deletion through cursor-enabled queries. They may also seek to deplete service provider resources deliberately, aiming to cause a denial-of-service state, thereby reducing service availability.
2. Authenticated Benign Users: This category includes legitimate users who, due to confusion or a lack of understanding, inadvertently engage in actions that consume service provider resources excessively. Such actions, while not ill-intended, can lead to unintended denial of service by overwhelming the system's capacity.

## Confidentiality

To ensure that confidential data remains appropriately secured:

* Implementors MUST ensure that pagination through results sets is strictly confined to the data that the actor's current identity has been authorized to access. This holds true even in cases where the actor has obtained a cursor pertaining to a result set that was generated by a different actor.
* Authorization checks MUST BE continuously applied as an actor navigates through the result set associated with a cursor. Under no circumstances should possession of a cursor be interpreted as granting any supplementary access privileges to the actor.
* In alignment with Section 2, cursor values SHOULD be treated as opaque entities. Clients should avoid making any inferences or assumptions about their internal structure.
* The system SHOULD handle error scenarios gracefully, while not exposing sensitive data. For instance, if an actor attempts to access a page of results outside their authorized scope, or if a request is made for a non-existent page, the system should respond with identical error messages, so as not to disclose any details of the underlying data or the nature of the authorization failure. It is acceptable, however, for the system to log different messages to a log accessible by administrators or other authorized personnel.

## Integrity

The extension discussed herein is query-only and does not inherently pose a substantial risk to data integrity. However, the focus is placed on safeguarding the integrity of the applications and clients that depend on this extension, rather than the integrity of the service provider. Specific considerations include:
It is not required to tie a cursor to specific actor. However, if a cursor is tied to an actor and if the actor's permissions change, and the actor is still using the cursor, the actor may miss records OR there may be unauthorized access to data.

* When possible, service providers SHOULD invalidate all tokens/watermarks corresponding to an actor immediately following a change in permissions. This ensures that any queries executed post-permission change, utilizing old tokens/watermarks, will be denied.
* As an alternative approach, service provider may opt to retain the existing tokens/watermarks but must ensure that any metadata tied to the result set, such as record counts, is updated to reflect the new permissions accurately.

## Availability

The concern for availability primarily stems from the potential for Denial of Service (DoS) attacks. If the service provider elects to retain substantial data or metadata for each cursor, numerous concurrent queries with &cursor could strain and eventually exhaust service provider resources. This could be orchestrated by an attacker with malicious intent or could occur innocuously as a result of actions taken by a benign but confused actor.

To mitigate such risks, the following strategies are recommended:

* Implementation of rate limiting to control the volume and cadence of cursor requests. This approach should adhere to established standards for rate limiting, details of which can be found in [RFC6585].
* Cursor mechanisms must be designed in a manner that avoids any additional consumption of service provider resources with the initiation of new &cursor requests.
* It is advisable to establish a ceiling on the number of cursors permissible at any given time. Alternatively, the adoption of an opaque identifier system that conservatively utilizes resources may be used.
* Token invalidation mechanisms (including mechanisms triggered by permissions changes) must be designed to be resource-efficient to prevent them from being exploited for DoS attacks.
* Actors may face challenges in maintaining a seamless pagination experience if their permissions are in a state of flux. Proactive measures should be taken to ensure that permission changes do not disrupt the user experience.

## Other Security References

Using URIs to describe and locate resources has its own set of security considerations discussed in Section 7 of [RFC3986].  Implementations SHOULD also refer to [BCP195] and [RFC9110] for additional security considerations that are relvant for underlying TLS and HTTP protocols.


# IANA Considerations

This specification requests IANA to amends the registry "SCIM Schema URIs for Data Resources" established by [RFC7643], for the `urn:ietf:params:scim:api:messages:2.0:SearchRequest` message URI and adds the following new fields:

SCIM `cursor` attribute

 - Field Name: `cursor`.
 - Status: permanent.
 - Specification Document: this specification, Section 2
 - Comments: see section 3.4.3 of [RFC7644] System for Cross-domain Identity Management: Protocol

SCIM `count` attribute

  - Field Name: `count`
  - Status: permanent
  - Specification Document: this specification, Section 2
  - Comments: see section 3.4.3 of [RFC7644] System for Cross-domain Identity Management: Protocol

This specification amends the entry  for urn:ietf:params:scim:api:messages:2.0:ListResponse message URI, and adds the following fields:

SCIM `nextCursor` attribute

  - Field Name: `nextCursor`
  - Status: permanent
  - Specification Document: this specification, Section 2
  - Comments: see section 3.4.2 of [RFC7644] System for Cross-domain Identity Management: Protocol

SCIM `previousCursor` attribute

  - Field Name: `previousCursor`
  - Status: permanent
  - Specification Document: this specification, Section 2
  - Comments: see section 3.4.2 of [RFC7644] System for Cross-domain Identity Management: Protocol

This specification amends the entry  for urn:ietf:params:scim:schemas:core:2.0:ServiceProviderConfig schema URI, and adds the following field:

SCIM `pagination` attribute

  - Field Name: `pagination`
  - Status: permanent
  - Specification Document: this specification, Section 4
  - Comments: see section 5 of [RFC7643] System for Cross-domain Identity Management: Protocol

# Change Log

RFC Editor: Please remove this section in the release version of the document.

-8

* Fix several typos and wording consistencies
* Add reference to RFC7644 in Security Considerations
* Adjust indenting and wording to clarify the definition of the pagination attribute in serviceProviderConfig.
* Reference RFC section 2.3 (not section 2.2) for unreserved characters 
* Reference section RFC 7644 3.4.3 (not section 3.4.2.4 ) for POST query 

-07

* Minor grammar change
* Add informative reference to BCP195 and RFC9110

-05

* Various updates in response to WG/IETF Last Call feedback

-04

* Added IANA Considerations section
* Added Security Considerations section
* Added Backwards Compatibility Considerations section

-03

* Minor grammatical/typo fixes, rename + changes to maxPageSize SCP definition

-02

* Typos/semantics, acknowledgements, expansion of cursorTimeout SCP definition

-01

* Updated after Httpdir review.

-00

* Adopted by SCIM WG.


# Acknowledgments and Contributions

The authors would like to acknowledge the contribution of Paul Lanzi (IDenovate) in leading the writing of security considerations section.

The authors would also like to acknowledge the following individuals who provided valuable feedback while reviewing the document:

* Aaron Parecki - Okta
* David Brossard - Axiomatics
* Dean H. Saxe - Beyond Identity
* Pamela Dingle - Microsoft


