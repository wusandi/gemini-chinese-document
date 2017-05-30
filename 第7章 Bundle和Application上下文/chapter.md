The unit of deployment (and modularity) in OSGi is the bundle (see section 3.2 of the OSGi Service Platform Core Specification). A bundle known to the OSGi runtime is in one of three steady states: installed, resolved, or active. Bundles may export services (objects) to the OSGi service registry, and by so doing make these services available for other bundles to discover and to use. Bundles may also export Java packages, enabling other bundles to import the exported types.

In Spring the primary unit of modularity is an application context, which contains some number of beans (objects managed by the Spring application context). Application contexts can be configured in a hierarchy such that a child application context can see beans defined in a parent, but not vice-versa. The Spring concepts of exporters and factory beans are used to export references to beans to clients outside of the application context, and to inject references to services that are defined outside of the application context.

There is a natural affinity between an OSGi bundle and a Spring application context. Using Gemini Blueprint, an active bundle may contain a Spring application context, responsible for the instantiation, configuration, assembly, and decoration of the objects (beans) within the bundle. Some of these beans may optionally be exported as OSGi services and thus made available to other bundles; beans within the bundle may also be transparently injected with references to OSGi services.

This chapter describes the lifecycle relationship between bundles and their application contexts, as imposed by Gemini Blueprint based on the events occurring at runtime, inside an OSGi environment.
7.1. The Gemini Blueprint Extender Bundle

Extender Pattern

A common pattern in OSGi applications is the extender, that (quoting Peter Kriens, OSGi Technical Director), ¡°allows other bundles to extend the functionality in a specific domain¡±. See this OSGi Alliance blog entry for an in-depth explanation.

The component responsible for detecting the Spring-powered bundles and instantiating their application context is the Gemini Blueprint extender. It serves the same purpose as the ContextLoaderListener does for Spring web applications. Once the extender bundle is installed and started it looks for any existing Spring-powered bundles that are already in the ACTIVE state and creates application contexts on their behalf. In addition, it listens for bundle starting events and automatically creates an application context for any Spring-powered bundle that is subsequently started. Section 8.1, ¡°Bundle Format And Manifest Headers¡± describes what the extender recognizes as a "Spring-powered bundle" while Section 8.3, ¡°Extender Configuration Options¡± how the extender can be configured. The extender monitors the lifecycle of the bundle it manages and will destroy automatically the contexts for bundles that are stopped. When the extender bundle itself is stopped, it will automatically close all the contexts that it manages, based on the service dependency between them. The extender bundle symbolic name is org.eclipse.gemini.blueprint.extender.
7.2. Application Context Creation

Once started, the extender analyses the existing started bundles and monitors any new bundle that will start. Once a Blueprint or Gemini Blueprint configuration is detected, the extender will create an application context for it in an asynchronous manner, on a different thread then the one starting the bundle (or delivering the STARTED event). This behaviour follows the OSGi specification recommendation and ensures that starting an OSGi Service Platform is fast and that bundles with service inter-dependencies do not cause deadlock (waiting for each other) on startup, as pictured below:
Application Context Sequence Diagram

The extender considers only bundles successfully started, that is, bundles in ACTIVE state; bundles in other states are ignored. Therefore a Spring-powered/Blueprint bundle will have its application context created after it has been fully started. It is possible to force synchronous/serialized creation of application contexts for started bundles, on a bundle-by-bundle basis. See Section 8.1, ¡°Bundle Format And Manifest Headers¡± for information on how to specify this behaviour.

If application context creation fails for any reason then the failure cause is logged. The bundle remains in the ACTIVE state; the application context lifecycle will not influence the bundle lifecycle in anyway. Naturally, since the context has failed, so will the functionality associated with it; for example there will be no services exported to the registry from the application context in this scenario.
7.2.1. Mandatory Service Dependencies

If an application context declares mandatory availability for certain imported OSGi services, the creation of the application context is blocked until all the mandatory dependencies can be satisfied through matching services available in the OSGi service registry. In practice, for most enterprise applications built using Gemini Blueprint services, the set of available services and bundles will reach a steady state once the platform and its installed bundles are all started. In such a world, the behaviour of waiting for mandatory dependencies simply ensures that bundles A and B, where bundle A depends on services exported by bundle B, may be started in any order.

A timeout applies to the wait for mandatory dependencies to be satisfied. By default the timeout is set to 5 minutes, but this value can be configured using the timeout directive. See Section 8.1, ¡°Bundle Format And Manifest Headers¡± for details.

Blueprint users could achieve the same result through the blueprint.timeout attribute declared on the Bundle-SymbolicName

It is possible to change the application context creation semantics so that application context creation fails if all mandatory services are not immediately available upon startup (see the aforementioned section for more information). Again, note that regardless of the configuration chosen, the failure of the application context will not change the bundle state.

For more information on the availability of imported services, see Section 9.2.1, ¡°Imported Service Availability¡±
7.2.2. Application Context Service Publication

Once the application context creation for a bundle has completed, the application context object is automatically exported as a service available through the OSGi Service Registry. The context is published under the interface org.springframework.context.ApplicationContext (and also all of the visible super-interfaces and types implemented by the context). The published service has a service property named org.springframework.context.service.name whose value is set to the bundle symbolic name of the bundle hosting the application context. In case of a Blueprint bundle, the container will be published under org.osgi.service.blueprint.container.BlueprintContainer while the bundle symbolic name will be published under osgi.blueprint.container.symbolicname property.

It is possible to prevent publication of the application context as a service using a directive in the bundle's manifest. See Section 8.1, ¡°Bundle Format And Manifest Headers¡± for details.

Note: the application context is published as a service primarily to facilitate testing, administration, and management. Accessing this context object at runtime and invoking getBean() or similar operations is discouraged. The preferred way to access a bean defined in another application context is to export that bean as an OSGi service from the defining context, and then to import a reference to that service in the context that needs access to the service. Going via the service registry in this way ensures that a bean only sees services with compatible versions of service types, and that OSGi platform dynamics are respected.
7.3. Bundle Lifecycle

OSGi is a dynamic platform: bundles may be installed, started, updated, stopped, and uninstalled at any time during the running of the framework.

When an active bundle is stopped, any services it exported during its lifetime are automatically unregistered and the bundle returns to the resolved state. A stopped bundle should release any resources it has acquired and terminate any threads. Packages exported by a stopped bundle continue to be available to other bundles.

A bundle in the resolved state may be uninstalled: packages that were exported by an uninstalled bundle continue to be available to bundles that imported them (but not to newly installed bundles).A bundle in the resolved state may also be updated. The update process migrates from one version of a bundle to another version of the same bundle.

Finally of course, a resolved bundle can be started, which transitions it to the active state.

The diagram below represents the bundle states and its transitions:
Bundle States

The OSGi PackageAdmin refreshPackages operation refreshes packages across the whole OSGi framework or a given subset of installed bundles. During the refresh, an application context in an affected bundle will be stopped and restarted. After a refreshPackages operation, packages exported by older versions of updated bundles, or packages exported by uninstalled bundles, are no longer available. Consult the OSGi specifications for full details.

When a Spring-powered or Blueprint bundle is stopped, the application context created for it is automatically destroyed. All services exported by the bundle will be unregistered (removed from the service registry) and the normal application context tear-down life-cycle is observed (org.springframework.beans.factory.DisposableBean implementors and destroy-method callbacks are invoked on beans in the context).

If a Spring-powered bundle that has been stopped is subsequently re-started, a new application context will be created for it.
7.4. The Resource Abstraction

The Spring Framework defines a resource abstraction for loading resources within an application context (see Spring's resource abstraction). All resource loading is done through the org.springframework.core.io.ResourceLoader associated with the application context. The org.springframework.core.io.ResourceLoader is also available to beans wishing to load resources programmatically. Resource paths with explicit prefixes - such as classpath: - are treated uniformly across all application context types (for example, web application contexts and classpath-based application contexts). Relative resource paths are interpreted differently based on the type of application context being created. This enables easy integration testing outside the ultimate deployment environment.

OSGi 4.0.x specification defines three different spaces from which a resource can be loaded. Gemini Blueprint supports all of them through its dedicated OSGi-specific application context and dedicated prefixes:

Table 7.1. OSGi resource search strategies
OSGi Search Strategy	Prefix	Explanation
Class Space	classpath:	Searches the bundle classloader (the bundle, all imported packages and required bundles). Forces the bundle to be resolved. This method has similar semantics to Bundle#getResource(String)
Class Space	classpath*:	Searches the bundle classloader (the bundle and all imported packages and required bundles). Forces the bundle to be resolved. This method has similar semantics to Bundle#getResources(String)
JAR File (or JarSpace)	osgibundlejar:	Searches only the bundle jar. Provides low-level access without requiring the bundle to be resolved.
Bundle Space	osgibundle:	Searches the bundle jar and its attached fragments (if there are any). Does not create a class loader or force the bundle to be resolved.

Please consult section 4.3.12 of the OSGi specification for an in depth explanation of the differences between them.
[Note]	Note
If no prefix is specified, the bundle space (osgibundle:) will be used.
[Note]	Note
Due to the OSGi dynamic nature, a bundle classpath can change during its life time (for example when dynamic imports are used). This might cause different classpath Resources to be returned when doing pattern matching based on the running environment or target platform.

All of the regular Spring resource prefixes such as file: and http: are also supported, as are the pattern matching wildcards. Resources loaded using such prefixes may come from any location, they are not restricted to being defined within the resource-loading bundle or its attached fragments.

OSGi platforms may define their own unique prefixes for accessing bundle contents. For example, Equinox defines the bundleresource: and bundlentry: prefixes. These platform specific prefixes may also be used with Gemini Blueprint, at the cost, of course, of tying yourself to a particular OSGi implementation.
7.5. Bundle Scope

Gemini Blueprint introduces a new bean scope named bundle. This scope is relevant for beans exported as an OSGi service and can be described as one instance per bundle. Beans exported as OSGi service, that have bundle scope, will result in a different instance created for each unique bundle that imports the service through the OSGi service registry. Consumers of the same bundle (whether defined through Gemini Blueprint or not) will see the same bean instance. When a bundle has stopped importing the bundle (for whatever reason), the bean instance is disposed. To the declaring bundle, a bundle-scoped bean behaves just like a singleton (i.e. there is only one instance per bundle, including the declaring one).This contract lifecycle is similar to that of the org.osgi.framework.ServiceFactory interface.

For more information regarding service publication and consumption, see Chapter 9, The Service Registry.
[Important]	Important
The bundle scope is relevant, only if the declaring bean is consumed through the OSGi service registry. That is, instances are created and destroyed (tracked) only when the bean exported as a service, is requested or released as an OSGi service by other bundles.
7.6. Accessing the BundleContext

In general there is no need to depend on any OSGi APIs when using the Gemini Blueprint support. If you do need access to the OSGi BundleContext object for your bundle, then Spring makes this easy to do.

The OSGi application context created by the Spring extender will automatically contain a bean of type BundleContext and with name bundleContext. You can inject a reference to this bean into any bean in the application context either by-name or by-type. In addition, Gemini Blueprint defines the interface org.eclipse.gemini.blueprint.context.BundleContextAware:

public interface BundleContextAware {
  public void setBundleContext(BundleContext context);
}

Any bean implementing this interface will be injected with a reference to the bundle context when it is configured by Spring. If you wish to use this facility within a bundle, remember to import the package org.eclipse.gemini.blueprint.context in your bundle manifest since otherwise the interface will not be visible to your bundle.
7.7. Application Context Destruction

The application context is bound to the bundle in which it lives. Thus, if the declaring bundle is being shutdown (for whatever reasons), the application context will be destroyed as well, all exported services being unregistered and all service imported dispose of.

As opposed to the application creation, the application context is destroyed in a synchronized manner, on the same thread that stops the bundle. This is required since once stopped, a bundle can not longer be used (even for class loading) preventing the application context shutdown from executing correctly.
Application Context Sequence Diagram

Note that a bundle can be closed individually or as part of a bigger event such as shutting down the entire OSGi platform. In this case or when the extender bundle is being closed down, the application contexts will be closed in a managed manner, based on the service dependencies between them. Please see the next section for more details.
7.8. Stopping the Extender Bundle

Shutdown algorithm change in 2.x

The shutdown algorithm implementation in Gemini Blueprint 1.0 has been revised to be better aligned with the Blueprint Container spec. Namely, the previous implementation performed ordering in only one pass while the latter performs multiple steps to accommodate the service changes in the OSGi space. Users should not discover any differences at runtime however, if that's not the case, please let us know.

If the extender bundle is stopped, then all the application contexts created by the extender will be destroyed. The algorithm described here is identical to that used by the Blueprint specification (section 121.3.11). Application contexts are shutdown in the following order:

    Application contexts that do not export any services, or that export services that are not currently referenced, are shutdown in reverse order of bundle id. (Most recently installed bundles have their application contexts shutdown first).

    Shutting down the application contexts in step 1 may have released references these contexts were holding such that there are now additional application contexts that can be shutdown. If so, repeat step 1 again.

    If there are no more active application contexts, we have finished. If there are active application contexts then there must be a cyclic dependency of references. The circle is broken by determining the highest ranking service exported by each context: the bundle with the lowest ranking service in this set (or in the event of a tie, the highest service id), is shut down. Repeat from step 1.

