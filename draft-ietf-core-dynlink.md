---
title: "Dynamic Resource Linking for Constrained RESTful Environments"
abbrev: Dynamic Resource Linking for CoRE
docname: draft-ietf-core-dynlink-latest
date: 2021-7-27
category: info

ipr: trust200902
area: art
workgroup: CoRE Working Group
keyword: [Internet-Draft, CoRE, CoAP, Link Binding, Observe]

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
- ins: M. Koster
  name: Michael Koster
  organization: SmartThings
  street: 665 Clyde Avenue
  city: Mountain View
  code: 94043
  country: USA
  email: michael.koster@smartthings.com
- role: editor
  ins: B. Silverajan
  name: Bilhanan Silverajan
  org: Tampere University
  street: Kalevantie 4
  city: Tampere
  code: 'FI-33100'
  country: Finland
  email: bilhanan.silverajan@tuni.fi

normative:
  RFC8288: link
  RFC6690: link-format


informative:
  RFC7252: coap
  RFC7641: observe
  RFC8132: patch
  I-D.irtf-t2trg-rest-iot: server-push
  I-D.ietf-core-conditional-attributes: conditional-attributes


--- abstract

This specification defines Link Bindings, which provide dynamic linking of state updates between resources, either on an endpoint or between endpoints, for systems using CoAP (RFC7252). 
--- note_Editor_note
 
The git repository for the draft is found at https://github.com/core-wg/dynlink

--- middle

Introduction        {#introduction}
============

IETF Standards for machine to machine communication in constrained environments describe a REST protocol {{-coap}} and a set of related information standards that may be used to represent machine data and machine metadata in REST interfaces. CoRE Link-format {{-link-format}} is a standard for doing Web Linking {{-link}} in constrained environments. 

This specification introduces the concept of a Link Binding, which defines a new link relation type to create a dynamic link between resources over which state updates are conveyed. Specifically, a Link Binding is a unidirectional link for binding the states of source and destination resources together such that updates to one are sent over the link to the other. CoRE Link Format representations are used to configure, inspect, and maintain Link Bindings.

Terminology     {#terminology}
===========

{::boilerplate bcp14}

This specification requires readers to be familiar with all the terms and concepts that are discussed in {{-link}}, {{RFC6690}} and {{RFC7641}}.  This specification makes use of the following additional terminology:

Link Binding:
: A unidirectional logical link between a source resource and a destination resource, over which state information is synchronized.

State Synchronization:
: Depending on the binding method (Polling, Observe, Push) different REST methods may be used to synchronize the resource values between a source and a destination. The process of using a REST method to achieve this is defined as "State Synchronization". The endpoint triggering the state synchronization is the synchronization initiator.


Link Bindings        {#bindings}
=============
In a M2M RESTful environment, endpoints may directly exchange the content of their resources to operate the distributed system. For example, a light switch may supply on-off control information that may be sent directly to a light resource for on-off control. Beforehand, a configuration phase is necessary to determine how the resources of the different endpoints are related to each other. This can be done either automatically using discovery mechanisms or by means of human intervention and a so-called commissioning tool. 

In this specification such an abstract relationship between two resources is defined, called a Link Binding. The configuration phase necessitates the exchange of binding information, so a format recognized by all CoRE endpoints is essential. This specification defines a format based on the CoRE Link-Format to represent binding information along with the rules to define a binding method which is a specialized relationship between two resources. 

The purpose of such a binding is to synchronize content updates between a source resource and a destination resource. The destination resource MAY be a group resource if the authority component of the destination URI contains a group address (either a multicast address or a name that resolves to a multicast address). Since a binding is unidirectional, the binding entry defining a relationship is present only on one endpoint. The binding entry may be located either on the source or the destination endpoint depending on the binding method. 

Conditional Notification Attributes defined in {{I-D.ietf-core-conditional-attributes}} can be used with Link Bindings in order to customize the notification behavior and timing.

The &quot;bind&quot; attribute and Binding Methods    {#binding_methods}
---------------

A binding method defines the rules to generate the network-transfer exchanges that synchronize state between source and destination resources. By using REST methods content is sent from the source resource to the destination resource. 

This specification defines a new CoRE link attribute &quot;bind&quot;. This is the identifier for a binding method which defines the rules to synchronize the destination resource. This attribute is mandatory.

| Attribute         | Parameter | Value            |
| --- | --- | --- |
| Binding method    | bind      | xs:string        |
{: #bindattribute title="The bind attribute"}

The following table gives a summary of the binding methods defined in this specification.

 | Name    | Identifier  | Location    | Method        |
 | --- | --- | --- | --- |
| Polling | poll        | Destination | GET           |
 | Observe | obs         | Destination | GET + Observe |
 | Push    | push        | Source      | PUT           |
 | Execute | exec        | Source      | POST          |
{: #bindsummary title="Binding Method Summary"}

The description of a binding method defines the following aspects:

Identifier: 
: This is the value of the &quot;bind&quot; attribute used to identify the method.

Location: 
: This information indicates whether the binding entry is stored on the source or on the destination endpoint.

REST Method: 
: This is the REST method used in the Request/Response exchanges.

Conditional Notification: 
: How Conditional Notification Attributes defined in {{I-D.ietf-core-conditional-attributes}} are used in the binding.

The binding methods are described in more detail below.

###Polling

The Polling method consists of sending periodic GET requests from the destination endpoint to the source resource and copying the content to the destination resource. The binding entry for this method MUST be stored on the destination endpoint. The destination endpoint MUST ensure that the polling frequency does not exceed the limits defined by the pmin and pmax attributes of the binding entry. The copying process MAY filter out content from the GET requests using value-based conditions (e.g based on the Change Step, Less Than, Greater Than attributes defined in {{I-D.ietf-core-conditional-attributes}}).

###Observe
 
The Observe method creates an observation relationship between the destination endpoint and the source resource. On each notification the content from the source resource is copied to the destination resource. The creation of the observation relationship requires the CoAP Observation mechanism {{RFC7641}} hence this method is only permitted when the resources are made available over CoAP. The binding entry for this method MUST be stored on the destination endpoint. The binding conditions are mapped as query parameters in the Observe request (see {{I-D.ietf-core-conditional-attributes}}).

###Push 

The Push method can be used to allow a source endpoint to replace an outdated resource state at the destination with a newer representation. When the Push method is assigned to a binding, the source endpoint sends PUT requests to the destination resource when the Conditional Notification Attributes are satisfied for the source resource. The source endpoint SHOULD only send a notification request if any included Conditional Notification Attributes are met. The binding entry for this method MUST be stored on the source endpoint.

###Execute 

An alternative means for a source endpoint to deliver change-of-state notifications to a destination resource is to use the Execute Method. While the Push method simply updates the state of the destination resource with the representation of the source resource, Execute can be used when the destination endpoint wishes to receive all state changes from a source. This allows, for example, the existence of a resource collection consisting of all the state changes at the destination endpoint. When the Execute method is assigned to a binding, the source endpoint sends POST requests to the destination resource when the Conditional Notification Attributes are satisfied for the source resource. The source endpoint SHOULD only send a notification request if any included Conditional Notification Attributes are met. The binding entry for this method MUST be stored on the source endpoint.

Note: Both the Push and the Execute methods are examples of Server Push mechanisms that are being researched in the Thing-to-Thing Research Group (T2TRG) {{I-D.irtf-t2trg-rest-iot}}.

Link Relation    {#relation_type}
------
Since Binding involves the creation of a link between two resources, Web Linking and the CoRE Link-Format used to represent binding information. This involves the creation of a new relation type, "boundto". In a Web link with this relation type, the target URI contains the location of the source resource and the context URI points to the destination resource. 


Binding Table     {#binding_table}
=============
The Binding Table is a special resource that describes the bindings on an endpoint. An endpoint offering a representation of the Binding Table resource SHOULD indicate its presence and enable its discovery by advertising a link at "/.well-known/core" {{RFC6690}}. If so, the Binding Table resource MUST be discoverable by using the Resource Type (rt) 'core.bnd'.

The Methods column defines the REST methods supported by the Binding Table, which are described in more detail below. 

| Resource      | rt=      | Methods  | Content-Format |
| --- | --- | --- | --- |
| Binding Table | core.bnd | GET, PUT | link-format    |
{: #intdesc title="Binding Table Description"}

The REST methods GET and PUT are used to manipulate a Binding Table. A GET request simply returns the current state of a Binding Table. A request with a PUT method and a content format of application/link-format is used to clear the bindings to the table or replaces its entire contents. All links in the payload of a PUT rquest MUST have a relation type &quot;boundto&quot;. 

The following example shows requests for discovering, retrieving and replacing bindings in a binding table.

~~~~
Req: GET /.well-known/core?rt=core.bnd (application/link-format)
Res: 2.05 Content (application/link-format)
</bnd/>;rt=core.bnd;ct=40

Req: GET /bnd/
Res: 2.05 Content (application/link-format)
<coap://sensor.example.com/a/switch1/>;
	rel=boundto;anchor=/a/fan,;bind="obs",
<coap://sensor.example.com/a/switch2/>;
	rel=boundto;anchor=/a/light;bind="obs"

Req: PUT /bnd/ (Content-Format: application/link-format)
<coap://sensor.example.com/s/light>;
  rel="boundto";anchor="/a/light";bind="obs";pmin=10;pmax=60
Res: 2.04 Changed 

Req: GET /bnd/
Res: 2.05 Content (application/link-format)
<coap://sensor.example.com/s/light>;
  rel="boundto";anchor="/a/light";bind="obs";pmin=10;pmax=60
~~~~
{: #figbindexp title="Binding Table Example"}

Additional operations on the Binding Table can be specified in future documents. Such operations can include, for example, the usage of the iPATCH or PATCH methods {{RFC8132}} for fine-grained addition and removal of individual bindings or binding subsets.

Implementation Considerations   {#Implementation}
=======================

The initiation of a Link Binding can be delegated from a client to a link state machine implementation, which can be an embedded client or a configuration tool. Implementation considerations have to be given to how to monitor transactions made by the configuration tool with regards to Link Bindings, as well as any errors that may arise with establishing Link Bindings in addition to established Link Bindings.

Security Considerations   {#Security}
=======================
Consideration has to be given to what kinds of security credentials the state machine of a configuration tool or an embedded client needs to be configured with, and what kinds of access control lists client implementations should possess, so that transactions on creating Link Bindings and handling error conditions can be processed by the state machine.

IANA Considerations
===================

Resource Type value 'core.bnd'
---------------------
This specification registers a new Resource Type Link Target Attribute 'core.bnd' in the Resource Type (rt=) registry established as per {{RFC6690}}.

Attribute Value:
: core.bnd

Description: See {{binding_table}}. This attribute value is used to discover the resource representing a binding table, which describes the link bindings between source and destination resources for the purposes of synchronizing their content.

Reference: This specification. Note to RFC editor: please insert the RFC of this specification.

Notes: None

Link Relation Type
-------------------
This specification registers the new "boundto" link relation type as per {{-link}}.

Relation Name: 
: boundto

Description: 
: The purpose of a boundto relation type is to indicate that there is a binding between a source resource and a destination resource for the purposes of synchronizing their content.

Reference: 
: This specification. Note to RFC editor: please insert the RFC of this specification.

Notes: 
: None

Application Data: 
: None

Acknowledgements
================
Acknowledgement is given to colleagues from the SENSEI project who were critical in the initial development of the well-known REST interface concept, to members of the IPSO Alliance where further requirements for interface types have been discussed, and to Szymon Sasin, Cedric Chauvenet, Daniel Gavelle and Carsten Bormann who have provided useful discussion and input to the concepts in this specification. Christian Amsuss supplied a comprehensive review of draft -06. Discussions with Ari Keränen led to the addition of an extra binding method supporting POST operations. 
 
Contributors
============

    Christian Groves
    Australia
    email: cngroves.std@gmail.com

    Zach Shelby
    ARM
    Vuokatti
    FINLAND
    phone: +358 40 7796297
    email: zach.shelby@arm.com

    Matthieu Vial
    Schneider-Electric
    Grenoble
    France
    phone: +33 (0)47657 6522
    eMail: matthieu.vial@schneider-electric.com

    Jintao Zhu
    Huawei
    Xi’an, Shaanxi Province
    China
    email: jintao.zhu@huawei.com 

Changelog
=========

draft-ietf-core-dynlink-14

* Conditional Atttributes section removed and submitted as draft-ietf-core-conditional-attributes-00


draft-ietf-core-dynlink-13

* Conditional Atttributes section restructured
* "edge" and "con" attributes added
* Implementation considerations, clarifications added when pmax == pmin
* rewritten to remove talk of server reporting values to clients

draft-ietf-core-dynlink-12

* Attributes epmin and epmax included
* pmax now can be equal to pmin

draft-ietf-core-dynlink-11

* Updates to author list

draft-ietf-core-dynlink-10

* Binding methods now support both POST and PUT operations for server push.

draft-ietf-core-dynlink-09

* Corrections in Table 1, Table 2, Figure 2. 
* Clarifications for additional operations to binding table added in section 5
* Additional examples in Appendix A  

draft-ietf-core-dynlink-08

* Reorganize the draft to introduce Conditional Notification Attributes at the beginning
* Made pmin and pmax type xs:decimal to accommodate fractional second timing
* updated the attribute descriptions. lt and gt notify on all crossings, both directions
* updated Binding Table description, removed interface description but introduced core.bnd rt attribute value


draft-ietf-core-dynlink-07

* Added reference code to illustrate attribute interactions for observations

draft-ietf-core-dynlink-06

* Document restructure and refactoring into three main sections
* Clarifications on band usage
* Implementation considerations introduced
* Additional text on security considerations

draft-ietf-core-dynlink-05

* Addition of a band modifier for gt and lt, adapted from draft-groves-core-obsattr
* Removed statement prescribing gt MUST be greater than lt

draft-ietf-core-dynlink-03

* General: Reverted to using "gt" and "lt" from "gth" and "lth" for this draft owing to concerns raised that the attributes are already used in LwM2M with the original names "gt" and "lt".

* New author and editor added. 

draft-ietf-core-dynlink-02

* General: Changed the name of the greater than attribute "gt" to "gth" and the name of the less than attribute "lt" to "lth" due to conlict with the core resource directory draft lifetime "lt" attribute.

* Clause 6.1: Addressed the editor's note by changing the link target attribute to "core.binding".

* Added Appendix A for examples.

draft-ietf-core-dynlink-01

* General: The term state synchronization has been introduced to describe the process of synchronization between destination and source resources.

* General: The document has been restructured the make the information flow better.

* Clause 3.1: The descriptions of the binding attributes have been updated to clarify their usage.

* Clause 3.1: A new clause has been added to discuss the interactions between the resources.

* Clause 3.4: Has been simplified to refer to the descriptions in 3.1. As the text was largely duplicated.

* Clause 4.1: Added a clarification that individual resources may be removed from the binding table.

* Clause 6: Formailised the IANA considerations.

draft-ietf-core-dynlink Initial Version 00:

* This is a copy of draft-groves-core-dynlink-00

draft-groves-core-dynlink Draft Initial Version 00:

* This initial version is based on the text regarding the dynamic linking functionality in I.D.ietf-core-interfaces-05.

* The WADL description has been dropped in favour of a thorough textual description of the REST API.

--- back
