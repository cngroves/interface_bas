---
title: "Binding Attribute Scope"
abbrev: Binding Att. Scope
docname: draft-groves-core-bas-01
date: 2017-04-20
category: std

ipr: trust200902
area: art
workgroup: CoRE Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
- ins: C. Groves
  name: Christian Groves
  organization: 
  street: '' 
  city: ''
  code: ''
  country: Australia
  email: cngroves.std@gmail.com
- ins: W. Yang
  name: Weiwei Yang
  organization: Huawei
  street: '' 
  city: ''
  code: ''
  country: P.R.China
  email: tommy@huawei.com

normative:
  RFC2119:
  RFC5988:
  RFC6690:
  I-D.ietf-core-dynlink:
  I-D.ietf-core-etch:
  I-D.ietf-core-interfaces:
  
  
  
informative:



--- abstract
This document specifies an additional CoAP binding attribute that allows binding attributes to be scoped to an item (sub-resource) in a collection resource. This allows synchronisation of multiple resources in a collection based on the value of one resource.

--- middle

Requirements Language     {#reqlang}
=====================
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",   "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in {{RFC2119}}.

Introduction {#introduction}
============
In some cases a CoAP client may require a notification of the value of a resource based on some condition associated with another resource. For example: a client wants to be notified of the machine state once a temperature sensor threshold is crossed. Currently the only way to implement this is for a client to be notified of the temperature resource change and then for the client to do a seperate read of the required resources. It would be more efficient to provide the information in a single response.

This document defines a new "Binding Attribute Scope" binding attribute to allow a client to request collection resource state synchronisation based on a conditional value of an item in the collection resource. 

The "Binding Attribute Scope" (bas) attribute is set on a collection and is used to indicate which item in a collection an Observe and associated binding attributes apply to. A linked batch (or batch) interface (sect.4.3/{{I-D.ietf-core-interfaces}}) is used on the collection. The client constructs a collection resource using the linked batch interface or uses a pre-provisioned collection via the batch interface to set the information that it requires. By placing the required information in a collection it allows a GET on the collection to return all the information based on the conditional value of an item.

Section {{bindingattributes}} below indicates the text that would need to be added to {{I-D.ietf-core-dynlink}} to add the Binding Attribute Scope attribute.

Usage of a collection
---------------------

{{I-D.ietf-core-interfaces}} indicates that a collection may be observed. An observation returns a representation of the entire collection. The "bas" binding attribute is only used on collection (e.g. batch, linked-batch) resources which enables the use of the binding attributes from {{I-D.ietf-core-dynlink}} on a collection. The "bas" attribute points to one item in the collection.

This approach does have the downside that it requires the use of a collection interface even if a single resource is required. The scope of the synchronization is limited to the current collection. However the use of a collection to return the information does seem to fit with the principle that a GET of a resource should return the same representation as a notification.

Multiple Conditions
-------------------

Given that the "bas" binding attribute only applies to one item/sub-resource in a collection it means that multiple resources cannot be used as conditions for a binding synchronization. To allow the use of conditions on multiple resources to determine resource synchronization a different approach would be required. 

One approach would be to use the FETCH method {{I-D.ietf-core-etch}} with a set of request parameters. A content type could be defined that allowed the binding attributes to be specified on multiple resources.

For example:

~~~~
FETCH /s/?pmin=1&pmax=100 content-type=application/conditionals+json

[
  { 
    "n": "/s/light",
    "st": 5
  },
  {
    "n": "/s/temp",
    "st": 1
  },
  {
    "n": "/s/humidity",
    "lt": 40,
    "gt" 70
  }
]
~~~~

This approach would add complexity over the use of the bas binding attribute however such complexity may be warranted in certain use cases. A new content-format would need to be defined. This approach is for further study.

Terminology     {#terminology}
===========
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",   "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this specification are to be interpreted as described in {{RFC2119}}.

This specification requires readers to be familiar with all the terms and concepts that are discussed in {{RFC5988}} and {{RFC6690}}.  

Binding Attributes {#bindingattributes}
===============================

This document introduces one new attributes:

| Attribute Name           | Parameter        | Data Format      |
| Binding Attribute Scope  | bas              | xsd:string       | 
{: #resobsattr title="Binding Attribute Summary"}

The xsd:string contains a Uri-Path.

Binding Attribute Scope
------------------------

The binding attribute scope attribute allows a client to indicate that the binding attributes contained in the query apply to a certain item in a collection resource. 

The collection containing the Observed resource and the additional resources to be returned is first created through the use of the Batch or Linked-Batch {{I-D.ietf-core-interfaces}} interface. The Linked-Batch allows a client to dynamically associate resources.

The client then creates a binding (e.g. an Observe) with the collection indicating the required binding attributes (e.g. "gt", "lt" etc.) and to which sub-resource these apply through the use of the "bas" binding attribute containing a URI-path. 

A client may create multiple bindings for the collection. When using Observe a client may use mutliple GET Observes with different queury parameters. Each observation is identified through the use of different Tokens.

If the server determines that the "bas" attribute contains an unknown URI-path then it responds with response code 4.04 "Not Found".

State sychronisation occurs when the sub-resource (item) value meets the conditions indicated by the binding attributes. When state synchronisation occurs the collection resource (including sub-resources) will be synchronised.

Interactions
------------
The "bas" parameter modifies the scope of other binding attributes in a query to apply to the resource specified in the "bas" URI-path.

**Editor's note: the interaction with sample time window/sample number window needs to be added if accepted.**


Examples
--------
Given the resource links:

~~~~
  Req: GET /.well-known/core
  Res: 2.05 Content (application/link-format)
  </s/>;rt="simple.sen";if="core.b",
  </s/light>;rt="simple.sen.light";if="core.s",
  </s/temp>;rt="simple.sen.tmp";if="core.s";obs,
  </s/humidity>;rt="simple.sen.hum";if="core.s"
~~~~
  
### Example 1 - Item Binding Attribute

~~~~
A Req: GET /s?bas="temp"&gt=37
          Token: 0x4a
          Observe: 0
would produce the following when temp exceeds 37:

  Res: 2.05 Content (application/senml+json)
  Token: 0x4a
  {"e":[
     { "n": "/s/light", "v": 123, "u": "lx" },
     { "n": "/s/temp", "v": 38, "u": "degC" },
     { "n": "/s/humidity", "v": 80, "u": "%RH" }],
  }
~~~~

### Example 2 - Mutliple Observes

In addition to the GET in example 1 the client could also request a notification when the humidity raises above 90%.

~~~~
A Req: GET /s?bas="humidity"&gt=90
          Token: 0x4b
          Observe: 0
would produce the following when temp exceeds 37:

  Res: 2.05 Content (application/senml+json)
  Token: 0x4b
  {"e":[
     { "n": "/s/light", "v": 123, "u": "lx" },
     { "n": "/s/temp", "v": 16, "u": "degC" },
     { "n": "/s/humidity", "v": 92, "u": "%RH" }],
  }
~~~~

*Editor's note: Need to add example for pmin*

Security Considerations
=======================

As per 5/{{I-D.ietf-core-dynlink}}.
  
IANA Considerations
===================

None.

Acknowledgements
================

Michael Koster for his comments and suggestion of the FETCH approach. Friedhelm Rodermund for stimulating discussion of the use case. Mojan Mohajer for raising the multiple observes on a collection use case.

Changelog
=========

draft-groves-core-bas-01

* Section 2: Removed editor's note on alternate solution

* Section 2.1: Changed language around observe on collections.

* Section 4.1: Added text regarding multiple observations.

draft-groves-core-bas-00

* Initial version

