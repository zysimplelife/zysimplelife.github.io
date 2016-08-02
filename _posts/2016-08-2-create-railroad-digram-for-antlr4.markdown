---
layout: post
title:  "How to create railroad digram for antlr 4 "
date:   2016-05-17 00:16:00 +0000
categories: JAVA
---

### Introduction ###
In order to defined a cli grammar, I selected ANTLR 4 as the library to help making analysis.  It was very helpful library saving me a lot of time.  I used Intellj IDE to compose my codes with ANTLR plug-in. All things went well until I wanted to provide provide grammar digram. I tried to find a command line way to generate railroad digram from antlr g4 file to integrate with my project. 

#### rrd-antlr4

After searching for a while, I could not find a one-click way to solve my problem, but rrd-antlr 4 seemed a acceptable option.  There was no off-shelf binary can be download directly. I needed compile the package from source code.  Following was what I did. 

Clone this repository:

```
git clone https://github.com/bkiers/rrd-antlr4.git

```

Then build it:

```
cd rrd-antlr4
mvn clean package

```

The JAR file containing all dependencies can be found in the `target` folder.

To give it a test by parsing the JSON grammar found in the official ANTLR 4repository, run the following command from the terminal:

```
cd target
java -jar rrd-antlr4-0.1.2.jar https://raw.github.com/antlr/grammars-v4/master/json/Json.g4
```



## summary ##

With method above, I can generate the digram and provide to my boss . 