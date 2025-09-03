---
title: "What's FHIR?"
date: 2025-08-28T12:59:43-07:00
description: "Part 1: Data and application standards for Healthcare"
---

# Introduction to FHIR

Currently, Electronic health records live in siloed systems and sharing patient information between these systems often involves custom integrations that are expensive. Reporting data between agencies or healthcare providers often uses HL7v2 which requires specialist knowledge and healthcare specific software.

FHIR (Fast Healthcare Interoperability Resources) represents a modern approach to solving this problem. Developed by HL7, FHIR leverages familiar web technologies like REST APIs and JSON to create a standardized way for healthcare systems to exchange data. The healthcare system, government and vendors can all implement to this common and open standard, reducing the integration work required and improving the interoperability of new systems. Vendors building EHR software, Cloud Services and Interoperability solutions are all starting to (or already) support FHIR.

# FHIR Resources Explained

FHIR Resources are the JSON representations of the entities (e.g. Patients, Providers) and events (e.g. Encounters, Procedures) we come across in the healthcare system. These resources are all defined by the HL7 FHIR standard and include standard fields, data types and naming conventions. FHIR is also a relational model, Encounters will contain a reference to the relevant Patient Resource and relevant Provider Resource(s).

For example, a patient resource looks like:

```json
{
  "resourceType": "Patient",
  "id": "john-smith-123",
  "name": [
    {
      "use": "official",
      "family": "Smith",
      "given": ["John"]
    }
  ],
  "gender": "male",
  "birthDate": "1985-03-15",
  "address": [
    {
      "use": "home",
      "line": ["456 Oak Avenue"],
      "city": "Boston",
      "state": "MA",
      "postalCode": "02101",
      "country": "US"
    }
  ]
}
```

and an encounter might look like:

```json
{
  "resourceType": "Encounter",
  "id": "john-checkup-visit",
  "status": "finished",
  "class": {
    "system": "http://terminology.hl7.org/CodeSystem/v3-ActCode",
    "code": "AMB",
    "display": "ambulatory"
  },
  "subject": {
    "reference": "Patient/john-smith-123",
    "display": "John Smith"
  },
  "period": {
    "start": "2025-08-28T10:00:00Z",
    "end": "2025-08-28T10:30:00Z"
  },
  "reasonCode": [
    {
      "text": "Annual wellness checkup"
    }
  ]
}
```
These resources highlight a couple of interesting points. In the `subject.reference` field we have `Patient/john-smith-123`, this is a foreign key reference to the John Smith patient record. We can also see the `class` object has a system which defines the valid codes to use (other common example codings are SNOMED and LOINC). Generally within FHIR Resources we are either capturing data or referencing it (via static defintions like CodeSystem or other FHIR Resources).

I would recommend checking the complete list of FHIR resources on the [HL7 website](https://build.fhir.org/resourcelist.html). The specification is incredibly comprehensive, the resources (or combinations of) can represent the vast majority of use cases within healthcare. FHIR also supports the definition of extensions where you can define your own (or use somebody elses custom) valid value list or modifications to standard resources. For instance, the extension for [birthPlace](https://build.fhir.org/ig/HL7/fhir-extensions/StructureDefinition-patient-birthPlace.html) supports capturing birth place on the patient record. These are extremely useful for extending FHIR in a transparent way as you can share your extension definition publicly allowing others to use it to interface with your application.

# FHIR APIs

Standardizing the data great, but for true operability we need standardization of the way we access that data. REST APIs are the most common way of access data between web applications and HL7 defined it's [RESTful API](https://build.fhir.org/http.html) for data exchange. Let's assume we have a FHIR server at fhir.example.com.

To get a specific resource by id (e.g. a clinical observation):
```bash
GET https://fhir.example.com/Observation/john-blood-pressure
```

To get a list of all encounters for a specific patient:
```bash
GET https://fhir.example.com/Encounter?subject=Patient/john-smith-123
```

To search all patients:
```bash
GET https://fhir.example.com/Patient?name=John%20Smith&birthdate=1960
```

A typical response from the search API would look like:
```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 1,
  "entry": [
    {
      "resource": {
        "resourceType": "Patient",
        "id": "john-smith-1960",
        "name": [
          {
            "use": "official",
            "family": "Smith",
            "given": ["John"]
          }
        ],
        "gender": "male",
        "birthDate": "1960-03-15"
      }
    }
  ]
}
```

These are just a couple of basic examples of supported API operations. The complete list is comprehensive and includes create, read, update and delete operations as well as various search operations. 

You'll notice that the resource type is a Bundle. Bundles are just a collection of FHIR Resources. We can build applications and processes that sit upon FHIR APIs to construct bundles to support our use cases. A notable example of this would be eICR (the way we report diseases to Public Health on FHIR) which is a bundle with a particular set of resources including the patient, confirmatory lab test, relevant encounters and messages from the provider detailing the diagnosis and other relevant information.


# Key Advantages

FHIR is will make development significantly quicker as we can see that we are no longer discussing how to map a Patient (or other data for that matter) from one system onto a Patient into another system. Instead, we can focus on business problems such as what information should be included in this exchange or how can we automate workflows like detemining case status or further reporting requirements from a collection of FHIR resources.

In my view, the industry should move towards seperating out healthcare data storage (i.e. FHIR servers) from applications and solutions (EHR UIs, Case Management Solutions, Analytics Systems). Being able to flexibly interchange front-ends would massively reduce effort for the healthcare system and Public Health to move or integrate new solutions or applications into their organisations. We could even get to the stage where a common FHIR server supports multiple applications (EHR and Case Management). Now, vendors and public health can distribute software to [run alongside a FHIR Server](https://docs.smarthealthit.org), which allows us all to tackle eICR and other reporting use cases in a completely new way - at the data source.

To summarise, FHIR provides:
* Modern web technologies with mature development communities
* Standardized data format
* Rich Open Source community
* Ability for healthcare industry to speak a common language
* Framework for implementing extensions and complex data collections

# Appendix - HL7v2

To show traditional HL7v2 messaging, here's what a patient admission message looks like in the older format:

```
MSH|^~\&|SENDING_APPLICATION|SENDING_FACILITY|RECEIVING_APPLICATION|RECEIVING_FACILITY|20250828101500||ADT^A01|12345|P|2.5
EVN||20250828101500|||^SMITH^JOHN^^^^^L
PID|1||123456^^^FACILITY^MR||SMITH^JOHN^M^^^^L||19850315|M|||456 OAK AVE^^BOSTON^MA^02101^USA||(617)555-1234|||EN|M|||123-45-6789
NK1|1|SMITH^JANE^^^^^L|SPO|456 OAK AVE^^BOSTON^MA^02101^USA|(617)555-5678
PV1|1|I|ICU^101^1|||^DUCK^DONALD^MD^^^^L||||19|||^DUCK^DONALD^MD^^^^L|EMR|||||||||||||||||||||20250828101500|20250828120000
```

FHIR is a great improvement as working with JSON and REST APIs are commonly held skills, HL7v2 (as you can imagine) is a little bit of a more niche skillset!

# References

[HL7 FHIR Resource List](https://build.fhir.org/resourcelist.html) - Complete catalog of all FHIR resources

[FHIR RESTful API Specification](https://build.fhir.org/http.html) - Official API documentation

[SMART on FHIR Documentation](https://docs.smarthealthit.org) - Framework for healthcare applications

[Patient Birth Place Extension](https://build.fhir.org/ig/HL7/fhir-extensions/StructureDefinition-patient-birthPlace.html) - Example FHIR extension







