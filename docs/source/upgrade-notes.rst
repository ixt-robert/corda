.. highlight:: kotlin
.. raw:: html

   <script type="text/javascript" src="_static/jquery.js"></script>
   <script type="text/javascript" src="_static/codesets.js"></script>

Upgrading apps to Corda 4
=========================

These notes provide instructions for upgrading your CorDapps from previous versions. Corda provides backwards compatibility for public,
non-experimental APIs that have been committed to. A list can be found in the :doc:`corda-api` page.

This means that you can upgrade your node across versions *without recompiling or adjusting your CorDapps*. You just have to upgrade
your node and restart.

However, there are usually new features and other opt-in changes that may improve the security, performance or usability of your
application that are worth considering for any actively maintained software. This guide shows you how to upgrade your app to benefit
from the new features in the latest release.

.. contents::
   :depth: 3

Step 1. Adjust the version numbers in your Gradle build files
-------------------------------------------------------------

Alter the versions you depend on in your Gradle file like so:

.. sourcecode:: groovy

    ext.corda_release_version = '4.0'
    ext.corda_gradle_plugins_version = '4.0.36'
    ext.kotlin_version = '1.2.71'
    ext.quasar_version = '0.7.10'

.. important:: Apps targeting Corda 4 may not at this time use Kotlin 1.3, as it was released too late in the development cycle
   for us to risk an upgrade. Sorry! Future work on app isolation will make it easier for apps to use newer Kotlin versions than
   the node itself uses.

You should also ensure you're using Gradle 4.10 (but not 5). If you use the Gradle wrapper, run::

    ./gradlew wrapper --gradle-version 4.10.3

Otherwise just upgrade your installed copy in the usual manner for your operating system.

Step 2. Add a "cordapp" section to your Gradle build file
---------------------------------------------------------

This is used by the Corda Gradle build plugin to populate your app JAR with useful information. It should look like this::

    cordapp {
        targetPlatformVersion 4
        minimumPlatformVersion 4
        contract {
            name "MegaApp Contracts"
            vendor "MegaCorp"
            license "A liberal, open source license"
            versionId 1
        }
        workflow {
            name "MegaApp flows"
            vendor "MegaCorp"
            license "A really expensive proprietary license"
        }
    }

Name and vendor can be set to any string you like, they don't have to be Corda identities. Target versioning is a new concept
introduced in Corda 4. Learn more by reading :doc:`versioning`. Setting a target version of 4 disables workarounds for various
bugs that may exist in your app, so by doing this you are promising that you have thoroughly tested your app on the new version.
Using a high target version is a good idea because some features and improvements are only available to apps that opt in.

The duplication between ``contract`` and ``workflow`` blocks exists because you should split your app into two separate JARs/modules,
one that contains on-ledger validation code like states and contracts, and one for the rest (called by convention the "workflows"
module although it can contain a lot more than just flows: services would also go here, for instance). For simplicity, here we
use one JAR for both, but this is in general an anti-pattern and can result in your flow logic code being sent over the network to
arbitrary third party peers, even though they don't need it.

Step 3. Upgrade your use of FinalityFlow
----------------------------------------

The previous ``FinalityFlow`` API is insecure. It doesn't have a receive flow, so requires counterparty nodes to accept any and
all signed transactions that are sent to it, without checks. It is **highly** recommended that existing CorDapps migrate
away to the new API, as otherwise things like business network membership checks won't be reliably enforced.

This is a two step process:

1. Change the flow that calls ``FinalityFlow``
2. Change or create the flow that will receive the finalised transaction.

As an example, let's take a very simple flow that finalises a transaction without the involvement of a counterpart flow:

.. container:: codeset

    .. literalinclude:: example-code/src/main/kotlin/net/corda/docs/kotlin/FinalityFlowMigration.kt
        :language: kotlin
        :start-after: DOCSTART SimpleFlowUsingOldApi
        :end-before: DOCEND SimpleFlowUsingOldApi

    .. literalinclude:: example-code/src/main/java/net/corda/docs/java/FinalityFlowMigration.java
        :language: java
        :start-after: DOCSTART SimpleFlowUsingOldApi
        :end-before: DOCEND SimpleFlowUsingOldApi
        :dedent: 4

To use the new API, this flow needs to be annotated with ``InitiatingFlow`` and a ``FlowSession`` to the participant of the transaction must be
passed to ``FinalityFlow`` :

.. container:: codeset

    .. literalinclude:: example-code/src/main/kotlin/net/corda/docs/kotlin/FinalityFlowMigration.kt
        :language: kotlin
        :start-after: DOCSTART SimpleFlowUsingNewApi
        :end-before: DOCEND SimpleFlowUsingNewApi

    .. literalinclude:: example-code/src/main/java/net/corda/docs/java/FinalityFlowMigration.java
        :language: java
        :start-after: DOCSTART SimpleFlowUsingNewApi
        :end-before: DOCEND SimpleFlowUsingNewApi
        :dedent: 4

If there are more than one transaction participants then a session to each one must be initiated, excluding the local party
and the notary.

A responder flow has to be introduced, which will automatically run on the other participants' nodes, which will call ``ReceiveFinalityFlow``
to record the finalised transaction:

.. container:: codeset

    .. literalinclude:: example-code/src/main/kotlin/net/corda/docs/kotlin/FinalityFlowMigration.kt
        :language: kotlin
        :start-after: DOCSTART SimpleNewResponderFlow
        :end-before: DOCEND SimpleNewResponderFlow

    .. literalinclude:: example-code/src/main/java/net/corda/docs/java/FinalityFlowMigration.java
        :language: java
        :start-after: DOCSTART SimpleNewResponderFlow
        :end-before: DOCEND SimpleNewResponderFlow
        :dedent: 4

For flows which are already initiating counterpart flows then it's a simple matter of using the existing flow session.
Note however, the new ``FinalityFlow`` is inlined and so the sequence of sends and receives between the two flows will
change and will be incompatible with your current flows. You can use the flow version API to write your flows in a
backwards compatible way.

Here's what an upgraded initiating flow may look like:

.. container:: codeset

    .. literalinclude:: example-code/src/main/kotlin/net/corda/docs/kotlin/FinalityFlowMigration.kt
        :language: kotlin
        :start-after: DOCSTART ExistingInitiatingFlow
        :end-before: DOCEND ExistingInitiatingFlow

    .. literalinclude:: example-code/src/main/java/net/corda/docs/java/FinalityFlowMigration.java
        :language: java
        :start-after: DOCSTART ExistingInitiatingFlow
        :end-before: DOCEND ExistingInitiatingFlow
        :dedent: 4

For the responder flow, insert a call to ``ReceiveFinalityFlow`` at the location where it's expecting to receive the
finalised transaction. If the initiator is written in a backwards compatible way then so must the responder.

.. container:: codeset

    .. literalinclude:: example-code/src/main/kotlin/net/corda/docs/kotlin/FinalityFlowMigration.kt
        :language: kotlin
        :start-after: DOCSTART ExistingResponderFlow
        :end-before: DOCEND ExistingResponderFlow
        :dedent: 8

    .. literalinclude:: example-code/src/main/java/net/corda/docs/java/FinalityFlowMigration.java
        :language: java
        :start-after: DOCSTART ExistingResponderFlow
        :end-before: DOCEND ExistingResponderFlow
        :dedent: 12

The responder flow may be waiting for the finalised transaction to appear in the local node's vault using ``waitForLedgerCommit``.
This is no longer necessary with ``ReceiveFinalityFlow`` and the call to ``waitForLedgerCommit`` can be removed.

Step 4. Upgrade your use of SwapIdentitiesFlow
----------------------------------------------

The :ref:`confidential_identities_ref` API is experimental in Corda 3 and remains so in Corda 4. In this release, the ``SwapIdentitiesFlow``
has been adjusted in the same way as ``FinalityFlow`` above, to close problems with confidential identities being injectable into a node
outside of other flow context. Old code will still work, but it is recommended to adjust your call sites so a session is passed into
the ``SwapIdentitiesFlow``.

Step 5. Possibly, adjust unit test code
---------------------------------------

``MockNodeParameters`` and functions creating it no longer use a lambda expecting a ``NodeConfiguration`` object.
Use a ``MockNetworkConfigOverrides`` object instead. This is an API change we regret, but unfortunately in Corda 3 we accidentally exposed
large amounts of the node internal code through this one API entry point. We have now insulated the test API from node internals and
reduced the exposure.

If you are constructing a MockServices for testing contracts, and your contract uses the Cash contract from the finance app, you
now need to explicitly add ``net.corda.finance.contracts`` to the list of ``cordappPackages``. This is a part of the work to disentangle
the finance app (which is really a demo app) from the Corda internals. Example::

    val ledgerServices = MockServices(
        listOf("net.corda.examples.obligation", "net.corda.testing.contracts"),
        identityService = makeTestIdentityService(),
        initialIdentity = TestIdentity(CordaX500Name("TestIdentity", "", "GB"))
    )

becomes::

    val ledgerServices = MockServices(
        listOf("net.corda.examples.obligation", "net.corda.testing.contracts", "net.corda.finance.contracts"),
        identityService = makeTestIdentityService(),
        initialIdentity = TestIdentity(CordaX500Name("TestIdentity", "", "GB"))
    )

Step 6. Security: refactor to avoid violating sealed packages
-------------------------------------------------------------

Hardly any apps will need to do anything in this step.

App isolation has been improved. Version 4 of the finance CorDapp (*corda-finance.jar*) is now built as a sealed and signed JAR file.
This means classes in your own CorDapps cannot be placed under the following package namespace:  ``net.corda.finance``

In the unlikely event that you were injecting code into ``net.corda.finance.*`` package namespaces from your own apps, you will need to move them
into a new package, e.g. ``net/corda/finance/flows/MyClass.java`` can be moved to ``com/company/corda/finance/flows/MyClass.java``.
As a consequence your classes are no longer able to access non-public members of finance CorDapp classes.

When recompiling your JARs for Corda 4, your own apps will also become sealed, meaning other JARs cannot place classes into your own packages.
This is a security upgrade that ensures package-private visibility in Java code works correctly.

Step 7. Security: Add BelongsToContract annotations
---------------------------------------------------

In versions of the platform prior to v4, it was the responsibility of contract and flow logic to ensure that ``TransactionState`` objects
contained the correct class name of the expected contract class. If these checks were omitted, it would be possible for a malicious counterparty
to construct a transaction containing e.g. a cash state governed by a commercial paper contract. The contract would see that there were no
commercial paper states in a transaction and do nothing, i.e. accept.

In Corda 4 the platform takes over this responsibility from the app, if the app has a target version of 4 or higher. A state is expected
to be governed by a contract that is either:

1. The outer class of the state class, if the state is an inner class of a contract. This is a common design pattern.
2. Annotated with ``@BelongsToContract`` which specifies the contract class explicitly.

Learn more by reading ":ref:`implicit_constraint_types`". If an app targets Corda 3 or lower (i.e. does not specify a target version),
states that point to contracts outside their package will trigger a log warning but validation will proceed.

Step 8. Consider adopting signature constraints
-----------------------------------------------

:doc:`design/data-model-upgrades/signature-constraints` are a new data model feature introduced in Corda 4. They make it much easier to
deploy application upgrades smoothly and in a decentralised manner. We strongly recommend all apps move to using signature constraints
as soon as feasible, as they represent the best tradeoff between the different upgrade control models.

.. important:: You will be able to use this feature if the compatibility zone you plan to deploy on has raised its minimum platform version
   to 4. Otherwise attempting to use signature constraints will throw an exception, because other nodes would not understand it or be able
   to check the correctness of the transaction. Please take this into account for your own schedule planning.

You can read more about signature constraints and what they do in :doc:`api-contract-constraints`. The ``TransactionBuilder`` class will
automatically use them if your application JAR is signed. We recommend all JARs are signed. To start signing your JAR files, read
:ref:`cordapp_build_system_signing_cordapp_jar_ref`.

Step 9. Consider adding extension points to your flows
------------------------------------------------------

In Corda 4 it is possible for flows in one app to subclass and take over flows from another. This allows you to create generic, shared
flow logic that individual users can customise at pre-agreed points (protected methods). For example, a site-specific app could be developed
that causes transaction details to be converted to a PDF and sent to a particular printer. This would be an inappropriate feature to put
into shared business logic, but it makes perfect sense to put into a user-specific app they developed themselves.

If your flows could benefit from being extended in this way, read ":doc:`flow-overriding`" to learn more.

Step 10. Explore other new features that may be useful
------------------------------------------------------

Corda 4 adds several new APIs that help you build applications. Why not explore:

* The `new withEntityManager API <api/javadoc/net/corda/core/node/ServiceHub.html#withEntityManager-block->`_ for using JPA inside your flows and services.
* :ref:`reference_states`, that let you use an input state without consuming it.
* :ref:`state_pointers`, that make it easier to 'point' to one state from another and follow the latest version of a linear state.