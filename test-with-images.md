<img href="image5.png">
![alt_text](images/OpenSSL-Strategic0.png "image_tooltip")


# OpenSSL Strategic Architecture


## 2018-09-17

****

OpenSSL Management Committee (OMC)




# Introduction

This document outlines the OpenSSL strategic architecture. It will take multiple releases to move the architecture from the current "as-is" (1.1.1) to the future "to-be" architecture. 


# As-is architecture

Currently, OpenSSL is split into three principle components:



1.  libcrypto. This is the core library for providing implementations of numerous cryptographic primitives. In addition it provides a set of supporting services which are used by libssl and libcrypto, as well as implementations of protocols such as CMS and OCSP.
1.  libssl. This library depends on libcrypto and implements the TLS and DTLS protocols.
1.  Applications. The applications are a set of command line tools that use the underlying libssl and libcrypto components to provide a set of cryptographic and other features such as:
    1.  Key and parameter generation and inspection
    1.  Certificate generation and inspection
    1.  SSL/TLS test tools
    1.  ASN.1 inspection
    1.  etc

In addition to the above components the functionality can be extended through the Engine API. Typically engines are dynamically loadable modules that are registered with libcrypto and use the available hooks to provide cryptographic algorithm implementations. Usually these are alternative implementations of algorithms already provided by libcrypto (e.g. to enable hardware acceleration of the algorithm), but they may also include algorithms not implemented in default OpenSSL (e.g. the GOST engine provides an implementation of the Russian GOST ciphers which are not available by default). Some engines are provided as part of the OpenSSL distribution, and some are provided by external third parties.


## Conceptual Component View

The existing architecture is a simple 3 level layering with the crypto layer at the bottom. The SSL/TLS layer depends on the crypto layer, and the applications depend on both the SSL/TLS and crypto layers. Within each layer the sub-component structure is flat.



<img href="image1.pmg">
![alt_text](images/OpenSSL-Strategic1.png "image_tooltip")



## Packaging View

The components described above are packaged into two shared libraries (libcrypto and libssl) as well as an "openssl" command line executable for running the various utilities. This is illustrated in the diagram below.



<img href="image4.png">
![alt_text](images/OpenSSL-Strategic2.png "image_tooltip")



# To-be Architecture

The to-be architecture has the following features:



*   All the implementation of cryptographic algorithms is moved conceptually into "Providers". A provider has implementations of one or more of the following: 
    *   The cryptographic primitives for an algorithm, e.g. how to encrypt/decrypt/sign/hash etc
    *   Serialisation for an algorithm, e.g. how to convert a private key into a PEM file. Serialisation could be to or from formats that are not currently supported.
    *   Store loader back ends. OpenSSL currently has a store loader that reads keys, parameters and other items from files. Providers could have a loader to load from another location (e.g. LDAP directory)
    *   (etc … i.e. other future components that allow implementations by module)
*   A provider may be entirely self-contained or it may interact with services provided by different providers. For example an application may use the cryptographic primitives for an algorithm implemented by a hardware accelerated provider, but use the serialisation services of a different provider in order to export keys into PKCS#12 format.
*   The EVP layer becomes a thin wrapper for services implemented in the providers. Most calls are passed straight through with little/no pre or post processing.
*   New APIs will be provided to find the implementation of an algorithm in the Method Dispatcher to be used for any given EVP call.
*   The Method Dispatcher will allow providers to register their services with it. It will implement a property based look-up feature for finding algorithms, e.g. it might allow you find an algorithm where "fips=true", or "keysize=128 and constant_time=true". The details of this will be determined in later design documents.
*   Information will be passed between the libcrypto core library and the providers in an implementation agnostic way. This is expected to be in the format of a set of properties.
*   A default provider (which contains the core of the current OpenSSL cryptographic algorithm implementations) will be "built-in" but other providers will be able to be dynamically loadable at runtime
*   There will be a new "legacy" provider module which will also be able to be dynamically loadable. This will provide cryptographic implementation for legacy algorithms. We will put in place a policy for how and when algorithms are transitioned from the default provider to the legacy provider.
*   Legacy APIs (e.g. low level cryptographic APIs that do not go via the EVP layer) will be moved out into a new libcrypto-legacy. Note that there are legacy APIs to non legacy algorithms (e.g. AES).
*   The new FIPS module will be a dynamically loadable module (there will not be a static version of the FIPS module). It's implementation will be entirely self-contained such that it will never use services of another provider - it can only have dependencies on the system runtime libraries. However the caller is able to pass in functionality (so long as it is not cryptographic) via methods that allow for improved  interactions. There are limits as to how far the concepts of method passing can be stretched and what sort of capability is permitted (in a FIPS context). 
*   Other interfaces may also be transitioned to use the Method Dispatcher over time (for example OSSL_STORE might be considered for this).


## Conceptual Component View

An overview of the conceptual components we expect to see in the new OpenSSL to-be architecture is as shown in the diagram below.



<img href="image2.png">
![alt_text](images/OpenSSL-Strategic3.png "image_tooltip")


The components shown here are as follows:



*   Applications: Command line applications, e.g. ca, ciphers, cms, dgst etc
*   Protocols: Provides capabilities for communicating between endpoints according to standard protocols
    *   SSL/TLS Protocols: An implementation of all supported SSL/TLS/DTLS protocols
        *   SSL BIO: A BIO for communication using SSL/TLS
        *   Statem: The SSL/TLS state machine
        *   Record: The SSL/TLS record layer
    *   Other Protocols
        *   CMS: An implementation of the Cryptographic Message Syntax standard
        *   OCSP: An implementation of Online Certificate Status Protocol
        *   TS: An implementation of the Timestamp Protocol
        *   etc
    *   Supporting Services: Components specifically designed to support the implementation of protocol code
        *   Packet: Internal component for reading protocol messages
        *   Wpacket: Internal component for writing protocol messages
*   OSSL Method Dispatcher: This is a fundamental component that connects requests for a service (such as encryption) to a provider of that service. It implements the ability for providers to register their services along with the properties of those services. It also provides the ability to locate a service given a set of properties that the service must fulfil. For example, properties of an encryption service might include "aead", "aes-gcm", "fips", "security-bits=128", etc.
*   Default Provider: Implements a set of default services that are registered with the OSSL Method Dispatcher.
    *   Supporting Services
        *   Low Level Implementations: This is the set of components that actually implement the cryptographic primitives.
        *   Engine: Support for the legacy Engine API
*   FIPS Provider: Implements a set of services that are FIPS validated and are registered with the OSSL Method Dispatcher.
    *   FIPS Dispatcher. An instance of the OSSL Method Dispatcher that only has services implemented by the FIPS Provider registered with it. If an application invokes a service, that itself depends on another service, this dependant service will be located via the FIPS Dispatcher. In this way the FIPS module will be entirely self-contained.
    *   FIPS Supporting Services
        *   POST: Power On Self Test
        *   KAT: Known Answer Tests
        *   Low Level Implementations: Alternative instances of the same low level implementation that are present in the default provider. These may have FIPS specific code paths based on conditional logic.
*   Legacy Provider: Provides implementations of legacy algorithms that will be exposed via EVP.
*   Third-Party Provider: Not part of the OpenSSL distribution. Third Parties may implement their own providers.
*   Services. Implements a set of supporting services that may be used directly by applications or by other components within OpenSSL.
    *   X509: X509 certificates
    *   CT: Certificate Transparency
    *   EVP: Algorithm independent crypto APIs
    *   etc
*   Legacy APIs. The "low-level" APIs. The "legacy" here refers to the API - not necessarily the algorithm itself. For example AES is not a legacy algorithm but it has a legacy API.


## Packaging View

The various components described in the conceptual component view above are physically packaged into three types of container:



*   Executable application for running the command line utilities
*   Shared libraries for loading directly by applications
*   Loadable modules for loading at runtime as required by the Method Dispatcher.

Some loadable modules may be available from third-party providers. The libraries and modules provided by a default OpenSSL instance are as shown below.



<img href="image3.png">
![alt_text](images/OpenSSL-Strategic4.png "image_tooltip")


The containers shown here are:



*   Openssl executable. The command line application.
*   Libssl. This contains everything directly related to SSL, TLS and DTLS. Its contents will be largely the same as libssl in the as-is architecture, although some supporting services may be moved to libcrypto.
*   Libcrypto. This library contains the following components:
    *   Implementations of the core services such as X509, ASN1, EVP, STORE etc
    *   The Method Dispatcher
    *   Protocols not related to SSL, TLS or DTLS
    *   Protocol supporting services (e.g. Packet and Wpacket)
    *   The default provider containing implementations of all the default algorithms
*   Libcrypto-legacy. Provides the legacy "low-level" APIs. Implementations of the algorithms for these APIS may come from any provider.
*   FIPS module. This is a dynamically loadable module which will be entirely self-contained (i.e. it will not link to libcrypto). Some components will exist both in the FIPS module and also in libcrypto. For example the AES implementation will be in both the FIPS module and the default provider. Where this is the case there will be one copy of the source code in order to ensure maintainability. The details of how this will work will be covered by later design documentation.
*   Legacy module. This is a dynamically loadable module containing the legacy provider. It is expected that this module will be linked with libcrypto in order to benefit from the various services provided by libcrypto.
*   Engine modules. These are engines conforming to the legacy engine 

<!-- GD2md-html version 1.0β13 -->
