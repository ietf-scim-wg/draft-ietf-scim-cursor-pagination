---
title: "Cursor-based Pagination of SCIM Resources"
abbrev: "SCIM Cursor Pagination"
docname: draft-ietf-scim-cursor-pagination-03
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
    name: Danny Zollner
    organization: Microsoft
    email: zollnerd@microsoft.com
    role: editor
   -
    name: Anjali Sehgal
    organization: Amazon Web Services
    email: anjalisg@amazon.com

normative: 

informative: 


--- abstract

 This document defines additional SCIM (System for Cross-Domain Identity Management) query parameters and result
   attributes to allow use of cursor-based pagination in SCIM
   implementations that are implemented with existing code bases,
   databases, or APIs where cursor-based pagination is already well-
   established.


--- middle

# Introduction

The two common patterns for result pagination are index-based pagination 
   and cursor-based pagination.  Rather than
   attempt to compare and contrast the advantages and disadvantages of
   competing pagination patterns, this document simply recognizes that
   SCIM service providers are commonly implemented as an
   interoperability layer on top of already existing application
   codebases, databases, and/or APIs that already have a well-
   established pagination pattern.

   Translating from an underlying cursor-based pagination pattern to the
   index-based pagination defined in Section 3.4.2.4 of [RFC7644]
   ultimately requires the SCIM service provider to fully iterate the
   underlying cursor, store the results, and then serve indexed pages
   from the stored results.  This task of "pagination translation"
   dramatically increases complexity and memory requirements for
   implementing a SCIM Service Provider, and may be an impediment to
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

The following table describes the URL pagination parameters requests for using cursor-based pagination:

| Parameter | Description |
cursor | The string value of the nextCursor attribute from a previous result page. The cursor value MUST be empty or omitted for the first request of a cursor-paginated query. This value may only contained characters from the unreserved characters set defined in section 2.2 of [RFC3986] |
count | A positive integer. Specifies the desired maximum number of query results per page, e.g., count=10. When specified, the service provider MUST NOT return more results than specified, although it MAY return fewer results. If count is not specified in the query, the maximum number of results is set by the service provider.
{: title="Query Parameters"}


The following table describes cursor-based pagination attributes
returned in a paged query response:

| Element | Description |
nextCursor | A cursor value string that MAY be used in a subsequent request to obtain the next page of results. Service providers supporting cursor-based pagination MUST include nextCursor in all paged query responses except when returning the last page. nextCursor is omitted from a response only to indicate that there are no more result pages. |
previousCursor | A cursor value string that MAY be used in a subsequent request to obtain the previous page of results. Use of previousCursor is OPTIONAL.
{: title="Response Attributes"}

   Cursor values are opaque; clients MUST not make assumptions about their structure. When the client wants to retrieve
   another result page for a query, it should query the same Service
   Provider endpoint with all query parameters and values being
   identical to the initial query with the exception of the cursor value
   which should be set to a nextCursor (or previousCursor) value that
   was returned by Service Provider in a previous response.

   For example, to retrieve the first 10 Users with userName starting
   with "J", use an empty cursor and set the count to 10:

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
   of nextCursor ("VZUTiyhEQJ94IR"):

~~~
     GET /Users?filter=username%20sw%20J&cursor=VZUTiyhEQJ94IR&count=10
     Host: example.com
     Accept: application/scim+json
     Authorization: Bearer U8YJcYYRMjbGeepD

	 HTTP/1.1 200 OK
 	 Content-Type: application/scim+json
 	 
     {
       "totalResults":100,
       "itemsPerPage":10,
       "previousCursor: "ze7L30kMiiLX6x"
       "nextCursor":"YkU3OF86Pz0rGv",
       "schemas":["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
       "Resources":[{
          ...
        }]
     }
~~~

   In the example above, the response includes the OPTIONAL
   previousCursor indicating that the Service Provider supports forward
   and reverse traversal of result pages.

   As described in Section 3.4.1 of [RFC7644] Service Providers SHOULD
   return an accurate value for totalResults which is the total number
   of resources for all pages.  Service Providers implementing cursor
   pagination that are unable to estimate totalResults MAY choose to omit the totalResults attribute.

## Pagination errors

   If a Service Provider encounters invalid pagination query
   parameters (invalid cursor value, count value, etc), or other error
   condition, the Service Provider SHOULD return the appropriate HTTP
   response status code and detailed JSON error response as defined in
   Section 3.12 of [RFC7644].  Most pagination error conditions would
   generate an HTTP response with status code 400.  Since many pagination
   error conditions are not user recoverable, error messages SHOULD
   focus on communicating error details to the SCIM client developer.

   For HTTP status code 400 (Bad Request) responses, the following detail error types are defined. These error types extend the list of error types defined in RFC 7644 Section 3.12, Table 9: SCIM Detail Error Keyword Values.

| scimType | Description | Applicability |
invalidCursor | Cursor value is invalid. Cursor value should be empty to request the first page and set to the nextCursor or previousCursor value for subsequent queries.| GET (Section 3.4.2 of [RFC7644])|
expiredCursor | Cursor has expired. Do not wait longer than cursorTimeout (600 sec) to request additional pages.| GET (Section 3.4.2 of [RFC7644])|
invalidCount | Count value is invalid. Count value must be between 1 - and maxPageSize (500) | GET (Section 3.4.2 of [RFC7644])|
{: title="Pagination Errors"}

## Sorting

   If sorting is implemented as described Section 3.4.2.3 of [RFC7644] ,
   then cursor-paged results SHOULD be sorted.

## Cursors as the Only Pagination Method

  A SCIM Service Provider MAY require cursor-based pagination to
   retrieve all results for a query by including a "nextCursor" value in
   the response even when the query does not include the "cursor"
   parameter.

   For example:

~~~

      GET /Users
      Host: example.com
      Accept: application/scim+json

~~~

   The SCIM Service Provider may respond to the above query with a page
   containing defaultPageSize results and a "nextCursor" value as shown
   in the below example (Resources omitted for brevity):

~~~
     HTTP/1.1 200 OK
     Content-Type: application/scim+json
     
     {
       "totalResults":5000,
       "itemsPerPage":100,
       "nextCursor":"HPq72Pax3JUaNa",
       "schemas":["urn:ietf:params:scim:api:messages:2.0:ListResponse"],
       "Resources":[{
          ...
        }]
     }
~~~

# Querying Resources using HTTP POST

  Section 3.4.2.4 of [RFC7644] defines how clients MAY execute the HTTP
   POST method combined with the "/.search" path extension to issue
   execute queries without passing parameters on the URL.  When using
   "/.search", the client would pass the parameters defined in Section 2

~~~
     POST /User/.search
     Host: example.com
     Accept: application/scim+json
     Authorization: Bearer U8YJcYYRMjbGeepD
     
     {
       "schemas": [
         "urn:ietf:params:scim:api:messages:2.0:SearchRequest"],
       "attributes": ["displayName", "userName"],
       "filter":
          "displayName sw \"smith\"",
       "cursor": "",
       "count": 10
     }
~~~

   Which would return a result containing a "nextCursor" value which may
   be used by the client in a subsequent call to return the next page of
   resources

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

# Service Provider Configuration

 The /ServiceProviderConfig resource defined in Section 4 of [RFC7644]
   facilitates discovery of SCIM service provider features.  A SCIM
   Service provider implementing cursor-based pagination SHOULD include
   the following additional attribute in JSON document returned by the
   /ServiceProviderConfig endpoint:

pagination
: A complex type that indicates pagination configuration options.
OPTIONAL.

   cursor  
   : A Boolean value specifying support of cursor-based pagination.
   REQUIRED.

   index  
   : A Boolean value specifying support of index-based pagination.
   REQUIRED.

   defaultPageSize 
   : Non-negative integer value specifying the default number of results
   returned in a page when a count is not specified in the query.  
   OPTIONAL.

   maxPageSize
   : Non-negative integer specifying the maximum number of results 
   returned in a page regardless of what is specified for the count 
   in a query. The maximum number of results returned may be further 
   restricted by other criteria. OPTIONAL.

   cursorTimeout 
   : Non-negative integer specifying the maximum number seconds that a 
   cursor is valid between page requests.  Clients waiting too long 
   between cursor pagination requests may receive an invalid cursor 
   error response. No value being specified may mean that there is no 
   cursor timeout or that the cursor timeout is not a static 
   duration.  OPTIONAL.

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
   (full ServiceProviderConfig schema defined in Section 5 of [RFC7643]
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
          "index": true
       },

       ...

      }
~~~

   Service Provider implementors SHOULD ensure that misuse of pagination
   by a SCIM client does not deplete Service Provider resources or
   prevent valid requests from other clients being handled.  Defenses
   for a SCIM Service Provider are similar those used to protect other
   Web API services -- including the use of a "Web API gateway" layer,
   to provide authentication, rate limiting, IP allow/block lists,
   logging and monitoring, response caching, etc.

   For example, an obvious protection against abuse is for the Service
   Provider to require client authentication in order to retrieve large
   result sets and enforce an overriding totalResults limit for non-
   authenticated clients.  Another example would be for a Service
   Provider that implements cursor pagination to restrict the number of
   cursors that can be allocated by a client or enforce cursor lifetimes.
   
# IANA Considerations
This specification amends the registry "SCIM Schema URIs for Data Resources" established by [RFC7643], for the urn:ietf:params:scim:api:messages:2.0:SearchRequest message URI and adds the following new fields:

SCIM “cursor” attribute
  
  - Field Name:  cursor. 
  - Status: permanent. 
  - Specification Document: this specification, Section 2
 - Comments: see section 3.4.3 of [RFC7644] System for Cross-domain Identity Management: Protocol

SCIM “count” attribute

  - Field Name: count
  - Status: permanent
  - Specification Document: this specification, Section 2
  - Comments: see section 3.4.3 of [RFC7644] System for Cross-domain Identity Management: Protocol

This specification amends the entry  for urn:ietf:params:scim:api:messages:2.0:ListResponse message URI, and adds the following fields:

SCIM “nextCursor” attribute

  - Field Name: nextCursor
  - Status: permanent
  - Specification Document: this specification, Section 2
  - Comments: see section 3.4.2 of [RFC7644] System for Cross-domain Identity Management: Protocol

SCIM “previousCursor” attribute  

  - Field Name: previousCursor
  - Status: permanent
  - Specification Document: this specification, Section 2
  - Comments: see section 3.4.2 of [RFC7644] System for Cross-domain Identity Management: Protocol


This specification amends the entry  for urn:ietf:params:scim:schemas:core:2.0:ServiceProviderConfig schema URI, and adds the following field:

  SCIM “pagination” attribute
  
  - Field Name: pagination
  - Status: permanent
  - Specification Document: this specification, Section 4
  - Comments: see section 5 of [RFC7643] System for Cross-domain Identity Management: Protocol

# Change Log
v04 - January 2024 - Added IANA Considerations
v03 - January 2024 - Minor grammatical/typo fixes, rename + changes to maxPageSize SCP definition
v02 - July 2023 - Typos/semantics, acknowledgements, expansion of cursorTimeout SCP definition
v01 - May 2023 - Updated after Httpdir review.
v00 - December 2022 - Adopted by SCIM WG. 


# Acknowledgments
{:numbered="false"}

The editor would like to acknowledge the tremendous contribution of Matt Peterson for his work in authoring the original versions of this draft and in providing continuing feedback after stepping back.

Matt Peterson
One Identity

The editor would also like to acknowledge the contributions of the following individuals who provided valuable feedback while reviewing the draft:

Aaron Parecki
Okta

David Brossard
Axiomatics

Dean H. Saxe
Amazon Web Services

Pamela Dingle
Microsoft









