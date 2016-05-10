---
layout: post
title:  "Why Jaxb need Object factory"
date:   5/10/2016 9:50:01 AM 
categories: WebServices
---


Oracle docs say:

*When XML element information can not be inferred by the derived Java representation of the XML content, a JAXBElement object is provided. This object has methods for getting and setting the object name and object value. Link here*

There are a few use cases where a JAXBElement is required:



- An element is both nillable="true" and minOccurs="0". In this case what does null on the mapped field/property mean? When the property is JAXBElement a null value means the element isn’t present and a JAXBElement wrapping null means an XML element with xsi:nil="true".


- There are 2 global elements with the same named complex type. Since in JAXB classes correspond to complex types a way is needed to capture which root element was encountered.
	http://blog.bdoughan.com/2012/07/jaxb-and-root-elements.html


- There is a choice structure where either foo or bar elements can occur and they are the same type. Here a JAXBElement is required because simply encountering a String value isn’t enough to indicate which element should be marshalled.


- An element with xsi:nil is encountered in the document that contains attributes. In this example the object corresponding to that element can still be unmarshalled to hold the attribute values, but JAXBElement can stil indicate that the element was null.