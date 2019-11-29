---
layout: post
title:  "Function test in maven"
date: 2019-10-08 15:10:00 +0000
categories: java, maven, CICD
---

### Maven Phase Concept

A Maven phase represents a stage in the Maven build lifecycle. Each * phase is responsible for a specific task.
 Here are some of the most important phases in the default build * lifecycle:
*
* validate: check if all information necessary for the build is available
* compile: compile the source code
* test-compile: compile the test source code
* test: run unit tests
* package: package compiled source code into the distributable format * (jar, war, …)
* integration-test: process and deploy the package if needed to run * integration tests
* verify: run any checks to verify the package is valid and meets quality criteria.
* install: install the package to a local repository
* deploy: copy the package to the remote repository

For the full list of each lifecycle's phases, check out the Maven [Reference](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html#Lifecycle_Reference).


### Function test with maven

There is no any thing that prevent you to run function test in unit test. but to make the life easier, it would be better to run function test in phase "integration-test" or phase "verify". In most of the project, there is however already several unit test. It can easily become mess when people put more and more function test into the same project. it quite confusing to mix them together. thing become even more complex when people want to run some test from command line.

After rebuilding several time function test framework for different project. We end up with some kind of best practice that I would like to summaries like this. Of cause people need follow other basic function test rule. Things here is only specific to maven projects.

#### Use separated project for function test
The product code should be store together with unit test but not function test.  The major reason is function test might involve more than one component from the product. With the separated project we don't need create very complex code structure either.

#### Put function test code in src directory
The reason there is very simple, for that we can put unit test of "test code" in test directory. It is also very important to test "test code" from our experience. Otherwise it is might easily end up with the useless trouble shooting task which turn out to be a "stupid" test code bug.

#### Use maven-surefire-plugin and maven-failsafe-plugin in the right way
It is quite normal to mix maven-surefire-plugin and maven-failsafe-plugin. In briefly, they are for different purpose.

* maven-surefire-plugin is designed for running unit tests and if any of the tests fail then it will fail the build immediately.
* maven-failsafe-plugin is designed for running integration tests, and decouples failing the build if there are test failures from actually running the tests.

It is naturally to use maven-surefire-plugin to run unit test and maven-failsafe-plugin to run function test. just notice that people can bind goals of maven-surefire-plugin to integration-test. we prefer however to keep the default value, at least it can reduce the size of pom file.

#### Use Test framework to handle the process of setting up and tearing down environment
It should also work to use pre-integration-test and post-integration-test. the only problem is how to run many test in parallel. Since testNg or even junit test have provided similar function, it is reasonable to use it to run many different test in parallel

#### Use Profile to apply different parameters
We don't need download all the dependency for just run a small group of test. with the help of "Profile", it is quite easy to create different "recept" for different test cases.

### Example
With those consideration in mind, the pom file might end up with following example

```xml

...
 <plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.0.0-M4</version>
    <configuration>
        <excludes>
        <exclude>**/ManualTest.java</exclude>
        </excludes>
    </configuration>
</plugin>

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <executions>
        <execution>
        <id>integration-test</id>
        <goals>
            <goal>integration-test</goal>
        </goals>
        </execution>
        <execution>
        <id>verify</id>
        <goals>
            <goal>verify</goal>
        </goals>
        </execution>
    </executions>
</plugin>


</profile>
    <profile>
      <id>ncft</id>
      <build>
       <configuration>
              <suiteXmlFiles>
                <suiteXmlFile>${project.build.outputDirectory}/suites/ncft.xml</suiteXmlFile>
              </suiteXmlFiles>
              <properties>
                <property>
                  <name>testnames</name>
                  <value>${testNames}</value>
                </property>
              </properties>
            </configuration>
          </plugin>
        </plugins>
        </build>
</profile>

 </profile>
    <profile>
      <id>ft</id>
      <build>
       <configuration>
              <suiteXmlFiles>
                <suiteXmlFile>${project.build.outputDirectory}/suites/ft.xml</suiteXmlFile>
              </suiteXmlFiles>
              <properties>
                <property>
                  <name>testnames</name>
                  <value>${testNames}</value>
                </property>
              </properties>
            </configuration>
          </plugin>
        </plugins>
        </build>
</profile>
...

```



### Reference
1.