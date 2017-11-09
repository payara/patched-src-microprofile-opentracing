/*
 * Copyright (c) 2017 Contributors to the Eclipse Foundation
 *
 * See the NOTICE file(s) distributed with this work for additional
 * information regarding copyright ownership.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * You may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

# OpenTracing

* Proposal: [MP-0007](0007-DistributedTracing.md)
* Authors: [Akihiko Kuroda](https://github.com/akihikokuroda), [Steve Fontes](https://github.com/Steve-Fontes)
* Status: **Awaiting review**
* Decision Notes: [Discussion thread topic covering the  Rationale](https://groups.google.com/forum/#!topic/microprofile/YxKba36lye4)

## Introduction

Distributed tracing allows you to trace the flow of a request across service boundaries.
This is particularly important in a microservices environment where a request typically flows through multiple services.
To accomplish distributed tracing, each service must be instrumented to log messages with a correlation id that may have been propagated from an upstream service.
A common companion to distributed trace logging is a service where the distributed trace records can be stored. ([Examples](http://opentracing.io/documentation/pages/supported-tracers.html)).
The storage service for distributed trace records can provide features to view the cross service trace records associated with particular request flows.

It will be useful for services written in the microprofile.io framework to be able to integrate well with a distributed trace system that is part of the larger microservices environment.

This proposal defines an API and microprofile.io behaviors that allow services to easily participate in an environment where distributed tracing is enabled.

Mailinglist thread: [Discussion thread topic for that proposal](https://groups.google.com/forum/#!topic/microprofile/YxKba36lye4)

## Motivation

In order for a distributed tracing system to be effective and usable, two things are required
1. The different services in the environment must agree on the mechanism for transferring correlation ids across services.
2. The different services in the environment should produce their trace records in format that is consumable by the storage service for distributed trace records.

Without the first, some services will not be included in the trace records associated with a request.
Without the second, custom code would need to be written to present the information about a full request flow.

There are existing distributed tracing systems that provide a server for distributed trace record storage and viewing, and application libraries for instrumenting microservices.
The problem is that the different distributed tracing systems use implementation specific mechanisms for propagating correlation IDs and for formatting trace records,
so once a microservice chooses a distributed tracing implementation library to use for its instrumentation, all other microservices in the environment are locked into the same choice.

The [opentracing.io project's](http://opentracing.io/) purpose is to provide a standard API for instrumenting microservices for distributed tracing.
If every microservice is instrumented for distributed tracing using the opentracing.io API, then (as long as an implementation library exists for the microservice's language),
the microservice can be configured at deploy time to use a common system implementation to perform the log record formatting and cross service correlation id propagation.
The common implementation ensures that correlation ids are propagated in a way that is understandable to all services,
and log records are formatted in a way that is understandable to the server for distributed trace record storage.

In order to make microprofile.io distributed tracing friendly, it will be useful to allow distributed tracing to be enabled on any microprofile.io application, without having to explicitly add distributed tracing code to the application.

In order to make microprofile.io as flexible as possible for adding distributed trace log records, microprofile.io should expose whatever objects are necessary for an application to use the opentracing.io API.

This proposal specifically addresses the problem of making it easy to instrument services with distributed tracing function, given an existing distributed tracing system in the environment.

This proposal specifically does not address the problem of defining, implementing, or configuring the underlying distributed tracing system. The proposal assumes an environment where all services use a common opentracing.io implementation (all zipkin compatible, all jaeger compatible, ...). At some point it would be beneficial to define another specification that allows inter-operation between different opentracing.io implementations.

## Proposed Solution

This specification defines an easy way to allow an application running
in a microprofile.io container to take advantage of distributed tracing
by using an opentracing.io Tracer implementation.

Rationale
---------

Distributed tracing allows you to trace the flow of a request across
service boundaries. This is particularly important in a microservices
environment where a request typically flows through multiple services.
To accomplish distributed tracing, each service must be instrumented to
log messages with a correlation id that may have been propagated from an
upstream service. A common companion to distributed trace logging is a
service where the distributed trace records can be stored.
(\[Examples\](<http://opentracing.io/documentation/pages/supported-tracers.html)>).
The storage service for distributed trace records can provide features
to view the cross service trace records associated with particular
request flows.

OpenTracing defines a Span as the conceptual artifact
that is propagated between services. The Span information can also be
communicated to a central server for storage.

The microprofile.io Opentracing proposal will describe how a developer
can easily specify methods to be traced.

The proposal will also describe how a microprofile.io implementation
will assist with Span creation and propagation. It is particularly
useful to be able to enable distributed tracing for an application
without having to add any code at all to the application.

Enabling distributed tracing with no code instrumentation {#_enabling_distributed_tracing_with_no_code_instrumentation}
---------------------------------------------------------

The microprofile.io implementation will allow JAX-RS applications to
participate in distributed tracing, without requiring developers to add
any distributed tracing code into their applications, and without
requiring developers to know anything about the distributed tracing
environment that their JAX-RS application will be deployed into.

1.  The microprofile.io implementation must provide a mechanism to
    configure an io.opentracing.Tracer implementation for use by each
    JAX-RS application.

2.  The microprofile.io implementation must provide a mechanism to
    automatically extract Span information from any incoming JAX-RS
    request.

3.  The microprofile.io implementation must provide a mechanism to
    automatically start a Span for any incoming JAX-RS request.

4.  The microprofile.io implementation must provide a mechanism to
    automatically inject Span information into any outgoing JAX-RS
    request.

5.  The microprofile.io implementation must provide a mechanism to
    automatically start a Span for any outgoing JAX-RS request.

All automatically created Spans must have the correct parent/child
relationships when creating either synchronous or asynchronous JAX-RS
incoming and outgoing requests.

The information about a Span that is propagated between services is
typically called a SpanContext. It is not the intent of this
specification to define the exact format for how SpanContext information
is stored or propagated. Our use case is for applications running in an
environment where all applications use the same Tracer implementation.
The Tracer implementation will create SpanContexts that are consistent
across all applications that use the same Tracer implementation.

A second use case would be to support SpanContext propagation between
different Tracer implementations. This second use case is outside the
scope of this specification.

### Tracer configuration {#_tracer_configuration}

An implementation of an io.opentracing.Tracer must be made available to
each application. Each application will have its own Tracer instance.
The Tracer must be configurable outside of the application to match the
distributed tracing environment where the application is deployed. For
example, it should be possible to take the exact same application and
deploy it to an environment where Zipkin is in use, and to deploy the
application without modification to a different environment where Jaeger
is in use, and the application should report Spans correctly in either
environment.

### Span extraction {#_span_extraction}

When a request arrives at a JAX-RS endpoint, an attempt is made to use
the configured Tracer to extract a Span from the arriving request. If a
Span is extracted, it is used as the parent Span for the new Span that
is created for the endpoint.

### Span creation for inbound requests {#_span_creation_for_inbound_requests}

When a request arrives at a JAX-RS endpoint, a new Span is created. The
new Span will be a child of the Span extracted from the incoming
request, if the extracted Span exists.

#### Span name {#_span_name}

The default operation name of the new Span for the incoming request is

    <HTTP method>:<package name>.<Class name>.<method name>

#### Span tags {#_span_tags}

Spans created for incoming requests will have the following tags added
by default:

-   Tags.SPAN\_KIND = Server

-   Tags.HTTP\_METHOD

-   Tags.HTTP\_URL

-   Tags.HTTP\_STATUS

-   Tags.ERROR (if true)

### Span creation and injection for outbound requests {#_span_creation_and_injection_for_outbound_requests}

When a request is sent from a JAX-RS javax.ws.rs.client.Client, a new
Span will be created that is injected in the outbound request for
propagation downstream. The new Span will be a child of the current Span
if a current Span exists. The new Span will be finished when the
outbound request is completed.

#### Span name {#_span_name_2}

The default operation name of the new Span for the outgoing request is

    <HTTP method>

#### Span tags {#_span_tags_2}

Spans created for outgoing requests will have the following tags added
by default:

-   Tags.SPAN\_KIND = Client

-   Tags.HTTP\_METHOD

-   Tags.HTTP\_URL

-   Tags.HTTP\_STATUS

-   Tags.ERROR (if true)

Enabling explicit distributed tracing code instrumentation {#_enabling_explicit_distributed_tracing_code_instrumentation}
----------------------------------------------------------

An annotation is provided to define explicit Span creation.

-   @Traced: Specify a class or method to be traced.

### The @Traced annotation {#_the_traced_annotation}

The @Traced annotation, applies to a Class or a method. When applied to
a Class, the @Traced annotation is applied to all methods of the Class.
The annotation starts a Span at the beginning of the method, and
finishes the Span at the end of the method.

The @Traced annotation has two optional arguments.

-   value=\[true\|false\]. Defaults to true. If @Traced is specified at
    the Class level, then @Traced(false) is used to annotate specific
    methods to disable creation of a Span for those methods. By default
    all JAX-RS endpoint methods are traced. To disable Span creation of
    a specific JAX-RS endpoint, the @Traced(false) annotation can be
    used.

    Even when the @Traced(false) annotation is used for a JAX-RS
    endpoint method, the upstream Spancontext will still be extracted if
    it exists. The extracted Spancontext will be used to create a
    current Span.

-   operationName=\<Name for the Span\>. Default is \"\". If the @Traced
    annotation finds the operationName as \"\", the operationName is
    assigned as \<package name\>.\<Class name\>.\<method name\>

Example:

``` {.java}
@InterceptorBinding
@Target({ TYPE, METHOD })
@Retention(RUNTIME)
public @interface Traced {
    @Nonbinding
    boolean value() default true;
    @Nonbinding
    String operationName() default "";
}
```

### io.opentracing.Tracer access {#_io_opentracing_tracer_access}

This proposal also specifies that the underlying opentracing.io Tracer
object configured instance is available for developer use. The
microprofile.io implementation will make the configured Tracer available
with CDI injection.

The configured Tracer object is accessed by injecting the Tracer class
that has been configured for the particular application for this
environment. Each application gets a different Tracer instance.

Example:

``` {.java}
@Inject
io.opentracing.Tracer configuredTracer;
```

Access to the configured Tracer gives full access to opentracing.io
functions.

The Tracer object enables support for the more complex tracing
requirements, such as when a Span is started in one method, and finished
in another.

Access to the Tracer also allows tags, logs and baggage to be added to
Spans with, for example:

``` {.java}
configuredTracer.activeSpan().setTag(...);
configuredTracer.activeSpan().log(...);
configuredTracer.activeSpan().setBaggage(...);
```
