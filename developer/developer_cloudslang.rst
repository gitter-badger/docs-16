CloudSlang
++++++++++

Embedded CloudSlang
===================

CloudSlang content can be run from inside an existing Java application
using Maven and Spring by embedding the CloudSlang Orchestration Engine
(Score) and interacting with it through the `Slang API <#slang-api>`__.

Embed CloudSlang in a Java Application
--------------------------------------

-  Add the Score and CloudSlang dependencies to the project's pom.xml
   file in the ``<dependencies>`` tag.

.. code-block:: xml

    <dependency>
      <groupId>io.cloudslang</groupId>
      <artifactId>score-all</artifactId>
      <version>0.3.28</version>
    </dependency>

    <dependency>
      <groupId>io.cloudslang.lang</groupId>
      <artifactId>cloudslang-all</artifactId>
      <version>0.9.60</version>
    </dependency>

    <dependency>
      <groupId>com.h2database</groupId>
      <artifactId>h2</artifactId>
      <version>1.3.175</version>
    </dependency>

-  Add Score and CloudSlang configuration to your Spring application
   context xml file.

.. code-block:: xml

    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xmlns:score="http://www.cloudslang.io/schema/score"
           xsi:schemaLocation="http://www.springframework.org/schema/beans
                            http://www.springframework.org/schema/beans/spring-beans.xsd
                            http://www.cloudslang.io/schema/score
                            http://www.cloudslang.io/schema/score.xsd">

    <bean class="io.cloudslang.lang.api.configuration.SlangSpringConfiguration"/>

      <score:engine />

      <score:worker uuid="-1"/>

    </beans>

-  Get the Slang bean from the application context xml file and interact
   with it using the `Slang API <#slang-api>`__.

.. code-block:: java

    ApplicationContext applicationContext =
                    new ClassPathXmlApplicationContext("/META-INF/spring/cloudSlangContext.xml");

    Slang slang = applicationContext.getBean(Slang.class);

    slang.subscribeOnAllEvents(new ScoreEventListener() {
        @Override
        public void onEvent(ScoreEvent event) {
            System.out.println(event.getEventType() + " : " + event.getData());
        }
    });


.. _slang_api:

Slang API
=========

The Slang API allows a program to interact with the CloudSlang
Orchestration Engine (Score) using content authored in CloudSlang. What
follows is a brief discussion of the API using a simple example that
compiles and runs a flow while listening for the events that are fired
during the run.

Example
-------

Code
~~~~

**Java Class - CloudSlangEmbed.java**

.. code-block:: java

    package io.cloudslang.example;

    import io.cloudslang.score.events.ScoreEvent;
    import io.cloudslang.score.events.ScoreEventListener;
    import io.cloudslang.lang.api.Slang;
    import io.cloudslang.lang.compiler.SlangSource;
    import org.springframework.context.ApplicationContext;
    import org.springframework.context.support.ClassPathXmlApplicationContext;

    import java.io.File;
    import java.io.IOException;
    import java.io.Serializable;
    import java.net.URISyntaxException;
    import java.util.HashMap;
    import java.util.HashSet;
    import java.util.Set;

    public class CloudSlangEmbed {
        public static void main(String[] args) throws URISyntaxException, IOException{
            ApplicationContext applicationContext =
                    new ClassPathXmlApplicationContext("/META-INF/spring/cloudSlangContext.xml");

            Slang slang = applicationContext.getBean(Slang.class);

            slang.subscribeOnAllEvents(new ScoreEventListener() {
                @Override
                public void onEvent(ScoreEvent event) {
                    System.out.println(event.getEventType() + " : " + event.getData());
                }
            });

            File flowFile = getFile("/content/hello_world.sl");
            File operationFile = getFile("/content/print.sl");

            Set<SlangSource> dependencies = new HashSet<>();
            dependencies.add(SlangSource.fromFile(operationFile));

            HashMap<String, Serializable> inputs = new HashMap<>();
            inputs.put("input1", "Hi. I'm inside this application.\n-CloudSlang");

            slang.compileAndRun(SlangSource.fromFile(flowFile), dependencies, inputs,
                    new HashMap<String, Serializable>());
        }

        private static File getFile(String path) throws URISyntaxException {
            return new File(CloudSlangEmbed.class.getResource(path).toURI());
        }
    }

**Flow - hello_world.sl**

.. code-block:: yaml

    namespace: resources.content

    imports:
      ops: resources.content

    flow:
      name: hello_world

      inputs:
        - input1

      workflow:
        - sayHi:
            do:
              ops.print:
                - text: "input1"

**Operation - print.sl**

.. code-block:: yaml

    namespace: resources.content

    operation:
      name: print

      inputs:
        - text

      python_action:
        script: print text

      results:
        - SUCCESS

Discussion
~~~~~~~~~~

The program begins by creating the Spring application context and
getting the Slang bean. In general, most of the interactions with Score
are transmitted through the reference to this bean.

.. code-block:: java

    ApplicationContext applicationContext =
              new ClassPathXmlApplicationContext("/META-INF/spring/cloudSlangContext.xml");

    Slang slang = applicationContext.getBean(Slang.class);

Next, the ``subscribeOnAllEvents`` method is called and passed a new
``ScoreEventListener`` to listen to all the
:ref:`Score <score_events>` and `Slang events <#slang-events>`__ that are fired.

.. code-block:: java

    slang.subscribeOnAllEvents(new ScoreEventListener() {
        @Override
        public void onEvent(ScoreEvent event) {
            System.out.println(event.getEventType() + " : " + event.getData());
        }
    });

The ``ScoreEventListener`` interface defines only one method, the
``onEvent`` method. In this example the ``onEvent`` method is overridden
to print out the type and data of all events it receives.

The API also contains a method ``subscribeOnEvents``, which takes in a
set of the event types to listen for and a method
``unSubscribeOnEvents``, which unsubscribes the listener from all the
events it was listening for.

Next, the two content files, containing a flow and an operation
respectively, are loaded into ``File`` objects.

.. code-block:: java

    File flowFile = getFile("/content//hello_world.sl");
    File operationFile = getFile("/content/print.sl");

These ``File`` objects will be used to create the two ``SlangSource``
objects needed to compile and run the flow and its operation.

A ``SlangSource`` object is a representation of source code written in
CloudSlang along with the source's name. The ``SlangSource`` class
exposes several ``static`` methods for creating new ``SlangSource``
objects from files, URIs or arrays of bytes.

Next, a set of dependencies is created and the operation is added to the
set.

.. code-block:: java

    Set<SlangSource> dependencies = new HashSet<>();
    dependencies.add(SlangSource.fromFile(operationFile));

A flow containing many operations or subflows would need all of its
dependencies loaded into the dependency set.

Next, a map of input names to values is created. The input names are as
they appear under the ``inputs`` key in the flow's CloudSlang file.

.. code-block:: java

    HashMap<String, Serializable> inputs = new HashMap<>();
    inputs.put("input1", "Hi. I'm inside this application.\n-CloudSlang");

Finally, the flow is compiled and run by providing its ``SlangSource``,
dependencies, inputs and an empty map of system properties.

.. code-block:: java

    slang.compileAndRun(SlangSource.fromFile(flowFile), dependencies,
            inputs, new HashMap<String, Serializable>());

An operation can be compiled and run in the same way.

Although we compile and run here in one step, the process can be broken
up into its component parts. The ``Slang`` interface exposes a method to
compile a flow or operation without running it. That method returns a
``CompliationArtifact`` which can then be run with a call to the ``run``
method.

A ``CompilationArtifact`` is composed of a Score ``ExecutionPlan``, a
map of dependency names to their ``ExecutionPlan``\ s and a list of
CloudSlang ``Input``\ s.

A CloudSlang ``Input`` contains its name, expression and the state of
all its input properties (e. g. required).

.. _slang_events:

Slang Events
============

CloudSlang uses :ref:`Score <score_events>` and
its own extended set of Slang events. Slang events are comprised of an
event type string and a map of event data that contains all the relevant
event information mapped to keys defined in the
``org.openscore.lang.runtime.events.LanguageEventData`` class. All fired
events are logged in the :ref:`execution log <execution_log>` file.

Events that contain ``SensitiveValue`` s will have the sensitive data replaced
by the ``********`` placeholder.

Event types from CloudSlang are listed in the table below along with the
event data each event contains.

All Slang events contain the data in the following list. Additional
event data is listed in the table below alongside the event type. The
event data map keys are enclosed in square brackets - [KEYNAME].

-  [DESCRIPTION]
-  [TIMESTAMP]
-  [EXECUTIONID]
-  [PATH]
-  [STEP_TYPE]
-  [STEP_NAME]
-  [TYPE]

+---------------------------+------------------------+----------------------------------+
| Type [TYPE]               | Usage                  | Event Data                       |
+===========================+========================+==================================+
| EVENT_INPUT_START         | | Input binding        | [INPUTS]                         |
|                           | | started for          |                                  |
|                           | | flow or operation    |                                  |
+---------------------------+------------------------+----------------------------------+
| EVENT_INPUT_END           | | Input binding        | [BOUND_INPUTS]                   |
|                           | | finished for         |                                  |
|                           | | flow or operation    |                                  |
+---------------------------+------------------------+----------------------------------+
| EVENT_STEP_START          | Step started           |                                  |
+---------------------------+------------------------+----------------------------------+
| EVENT_ARGUMENT_START      | | Argument binding     | [ARGUMENTS]                      |
|                           | | started for step     |                                  |
+---------------------------+------------------------+----------------------------------+
| EVENT_ARGUMENT_END        | | Step arguments       | [BOUND_ARGUMENTS]                |
|                           | | resolved             |                                  |
+---------------------------+------------------------+----------------------------------+
| EVENT_OUTPUT_START        | | Output binding       | | [executableResults]            |
|                           | | started for          | | [executableOutputs]            |
|                           | | flow or operation    | | [actionReturnValues]           |
+---------------------------+-----------------------+-----------------------------------+
| EVENT_OUTPUT_END          | | Output binding       | | [OUTPUTS]                      |
|                           | | finished for         | | [RESULT]                       |
|                           | | flow or operation    | | [EXECUTABLE_NAME]              |
+---------------------------+-----------------------+-----------------------------------+
| EVENT_OUTPUT_START        | | Output binding       | | [operationReturnValues]        |
|                           | | started for step     | | [stepNavigationValues]         |
|                           |                        | | [stepPublishValues]            |
+---------------------------+------------------------+----------------------------------+
| EVENT_OUTPUT_END          | | Output binding       | | [nextPosition]                 |
|                           | | finished for step    | | [RESULT]                       |
|                           |                        | | [OUTPUTS]                      |
+---------------------------+------------------------+----------------------------------+
| EVENT_EXECUTION_FINISHED  | | Execution finished   | | [RESULT]                       |
|                           | | running              | | [OUTPUTS]                      |
+---------------------------+------------------------+----------------------------------+
| EVENT_ACTION_START        | | Before action        | | [TYPE]                         |
|                           | | invocation           | | [CALL_ARGUMENTS]               |
+---------------------------+------------------------+----------------------------------+
| EVENT_ACTION_END          | | After successful     | [RETURN_VALUES]                  |
|                           | | action invocation    |                                  |
+---------------------------+------------------------+----------------------------------+
| EVENT_ACTION_ERROR        | | Exception in         | [EXCEPTION]                      |
|                           | | action execution     |                                  |
+---------------------------+------------------------+----------------------------------+
| EVENT_SPLIT_BRANCHES      | | parallel loop        | [BOUND_PARALLEL_LOOP_EXPRESSION] |
|                           | | expression bound     |                                  |
+---------------------------+------------------------+----------------------------------+
| EVENT_BRANCH_START        | | Parallel loop        | | [splitItem]                    |
|                           | | branch created       | | [refId]                        |
+---------------------------+------------------------+----------------------------------+
| EVENT_BRANCH_END          | | Parallel loop        | [branchReturnValues]             |
|                           | | branch ended         |                                  |
+---------------------------+------------------------+----------------------------------+
| EVENT_JOIN_BRANCHES_START | | Parallel loop output | | [stepNavigationValues]         |
|                           | | binding started      | | [stepAggregateValues]          |
+---------------------------+------------------------+----------------------------------+
| EVENT_JOIN_BRANCHES_END   | | Parallel loop output | | [nextPosition]                 |
|                           | | binding finished     | | [RESULT]                       |
|                           |                        | | [OUTPUTS]                      |
+---------------------------+------------------------+----------------------------------+
| SLANG_EXECUTION_EXCEPTION | | Exception in         | | [IS_BRANCH]                    |
|                           | | previous step        | | [executionIdContext]           |
|                           |                        | | [systemContext]                |
|                           |                        | | [EXECUTION_CONTEXT]            |
+---------------------------+------------------------+----------------------------------+
