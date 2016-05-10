---
layout: post
title:  "Jaxb Basic "
date:   2016-05-10 00:16:00 +0000
categories: WebService
---



JAXB is a very very basic and old fashion topic, but it seem I do no understand it enough.  It make me somehow confusing when I need solve some details issues.  So, I want to summary some experience of it to organize thought.

At Beginning of the background 

My understanding of JAXB is to use Java Class with annotation to stand for a XML schema. It very useful to handle a stable structured data model link SOAP protocol which make it very popular in enterprise application rather than web app.

But  because of the difference between  xml and java class,   there are a lot of limitation which need complex extension configuration  to adapt that.

My project is trying to translate from ASN.1 to and JAXB defined class for migration from old system to next generation.  I need to fix a lot of problem to make the translation work.

Before make details job,  it is good to review Jaxb basic knowledge with this web page of JAXB Tutorial 

Unmarshalling

```java
public <T> T unmarshal( Class<T> docClass, InputStream inputStream )
    throws JAXBException {
    String packageName = docClass.getPackage().getName();
    JAXBContext jc = JAXBContext.newInstance( packageName );
    Unmarshaller u = jc.createUnmarshaller();
    JAXBElement<T> doc = (JAXBElement<T>)u.unmarshal( inputStream );
    return doc.getValue();
}

```


Marshalling

Usually hidden in the middle of the list of the classes derived from the types defined in an XML schema there will be one class called ObjectFactory. It’s convenient to use the methods of this class because they provide an easy way of creating elements that have to be represented by a JAXBElement<?> object. Given that the top-level element of a document is represented as a JAXBElement<RulebaseType> with the tag “rulebase”, one such doument object can be created by code as shown below.

```java

ObjectFactory objFact = new ObjectFactory();
RulebaseType rulebase = objFact.createRulebaseType();
JAXBElement<RulebaseType> doc = objFact.createRulebase( rulebase );

```