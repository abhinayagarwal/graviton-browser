Introduction
************

We would like a competitor to the web browser for JVM application distribution and deployment, to act as an alternative
to the now deprecated Java Web Start, applets, and javapackager style bundled distribution. Initial target markets:

* Internal apps in industrial and enterprise scenarios, e.g. finance and trading applications where web apps are often
  not preferred for productivity/usability/security and other reasons. Slick updates and a deploy-once runtime are
  useful here.
* Hobbyist and learner apps where users do not want the overhead of complex deployment, but do want to publish their
  apps. They aren't attempting to maximise the user acquisition funnel so don't mind telling users to download an app
  browser component. These are people who distribute JARs or only source code today.
* Eventually, the hobbyist Java gaming community.
* IoT devices that struggle to provide secure administration interfaces, given the demands of web browser makers to use
  web PKI SSL and the poor fit of SSL with embedded web servers that do not have domain names.

We feel these market segments are under-served by web browser developers.

The resulting platform should use the JVM for platform independence, sandboxing, code streaming and other services but
should otherwise be language and UI toolkit agnostic to the extent possible. That is, Graviton Browser is not a Kotlin
or JavaFX specific project, even though it may use these tools for its own implementation.

FAQ
===

**Why the JVM?** We don't want to repeat the web's mistake of restricting us to a single language. The JVM has the best
support for running multiple languages and having them interoperate. It also has some of the best cross platform user
interface frameworks, many excellent libraries and modules, good tools and a relatively clean design. Through Graal
and Truffle languages we can combine not only scripting languages like JavaScript, Python and Ruby but also C/C++
modules via Sulong. This gives us a direct equivalent to WebAssembly.

**Aren't JVMs bloated and sluggish?** Historically yes. We aren't worried about this though for three reasons. (1) Our competition
isn't expertly written C++ apps but web apps, which are even worse. (2) The bulk of the JVM's reputation for sluggishness
comes from startup time and memory usage, all of which are being tackled by the JVM team through recently added features like ahead of time
compilation, AppCDS (class data pre-computation and sharing), GCs that don't pause the application and other such
features. (3) The JVM's reputation largely dates from the late 1990's and early 2000's when hardware wasn't as good as
it is now. Over time hardware got bigger and the JVM got more efficient. So we aren't worried about this so much anymore.

**Is this real?** Not at the moment. Why not help us make it real? Check out the
`task list <https://github.com/mikehearn/graviton-browser/labels/task>`_.

User experience sketch
======================

Graviton exposes a dual user experience. For GUI apps it is somewhat analagous to a web browser, but with a greater
focus on allowing apps to open top level windows (escape the tab). It features:

* A tabbed UI.
* An "address bar", albeit with some quirks as outlined below.
* Streaming, instant-on apps with no permission requests or signing requirements for the default case.
* Automatic and silent app updates, in the same manner as a web app.

However, Graviton has a stronger notion of an app than the web does. Apps are not defined by origins but by modules. The
browser understands module dependencies and will resolve them on the fly in the background.

Users can pin, downgrade and otherwise control the version of an app they are running from the Graviton settings UI,
restricted by optional developer specified policy. This means that botched app upgrades can be worked around by end users,
that offline support can work well, that demos are not unnecessarily affected by bad wifi or an unexpected app upgrade
between preparation and presentation.

Graviton may understand and use mDNS to enable network administrators to publish discoverable apps, as an alternative to
intranet/default-page style deployment.

A basic page / Markdown rendering feature is supported. This enables the root UI of an app that is embedded in a tab to
be basic instructions or release notes, if the app really doesn't want to be confined to a tab and would rather open its
own windows.

Command line apps are first class entities in Graviton. There is a command line interface analogous to apt-get, brew,
and so on that allows you to install command line apps and keep background updated::

    $ graviton alias sketch https://foobar.com/apps/sketch
    Downloading https://foobar.com/apps/sketch and linking to /usr/local/bin/sketch ...
    Linked version 3.2

    $ sketch --help
    Welcome to sketch 3.2

Some time later::

    $ sketch --help
    <silent delay as the app is updated>
    Welcome to sketch 3.4


Implementation sketch
=====================

**Top level UI**.  The root tabbed window is a JavaFX app that may use TornadoFX.

It uses the SubScene component to allow embedding of sub-apps within a tab view. This implies that embedded JavaFX apps
do not start from an Application object, as they would if they were bundled standalone. Rather, Graviton provides its
own equivalent that exposes host services.

This implies apps must be designed to run in Graviton. Existing apps can be easily adapted to run inside the sandbox,
but it isn't as simple as "drop in a JAR". This allows us to impose browser style just-in-time permissioning, PowerBoxes
and other security approaches that have been tried and tested in other platforms, but which may not be an exact fit for
the desktop application model. A GravitonApp base class may be the way to go here.

**App streaming.** Apps are modules compressed with pack200 (Jigsaw or OSGi, tbd). They load and run in an isolated sandbox
using the standard JVM security management and classloading APIS. They are intended to load fast so a module should be,
on average, not larger than the size of the average web page. There is tooling such as a Gradle plugin that checks this
and warns the developer if their module graphs exceed this size. That ensures app loading speed can be competitive with
the web. An async version of ServiceLoader or a similar API exposed via the GravitonApp entrypoint can be used to get a
similar experience to the page-style incremental loading experience - that is, whilst one part of the app has loaded,
other parts may be loading asynchronously.

It would once have been hard to get an incremental loading experience with Java because JVM apps tended to be larger
than web pages (bytecode is slightly more verbose than JavaScript despite being binary). In practice network speeds are
now very high and web pages have become so bloated that it should be easy to match their size with pack200 compression.

**Network connectivity and discovery.** HTTP caching and loading is used for module fetch. We do not attempt to replace
HTTP(/2) for this project. OkHttp has a local cache implementation that may be useful, or the new HTTP API in Java 10
may be sufficient. If the remote server supports HTTP2 then an open long-lived connection is built by the browser and
provided to the app, which it can then use for communication back to the server.

Client/server communications is left out of scope for this project, but must be able to run over HTTP 1 and 2. It is
expected that apps will bring in their own abstractions over HTTP as SPA web apps do.

mDNS/Bonjour discovery is used to locate domain names and apps that are advertising themselves on the local network.
Whilst this would be ineffective for very large enterprise networks that are multi-segment and do not support broadcast,
it is sufficient for IoT devices to advertise themselves (e.g. printers, wifi hotspots, etc). It is also sufficient for
factory floor applications, smaller offices, and so on. mDNS/Bonjour names look like this "foobar.local" where the name
is chosen by the app itself.

For larger networks ordinary domain names can be used to identify and start apps.

A server may present a self signed certificate if it was reached via mDNS. This certificate is then remembered and may
not change in future without generating a giant red scary warning page. This is different to how web browsers use SSL
and is intended to make life easier for internal app developers and embedded devices that struggle to obtain a web PKI
certificate today.

Given a domain name a "well known" URL is constructed and then fed to an HTTP library to initiate module fetch.

In future the DAT protocol may be interesting as an additional protocol, if consumer use cases turn out to be more
popular than hoped for.

**Module repository storage.** A Maven repository format is probably sufficient. Whilst not designed for app streaming and
with improvable efficiency, it is probably not required given the general bloat of the modern web, and a better
repository format can be left out of scope.

An ideal module system would hard-code the SHA checksums of the dependent modules. This would allow global deduplication
in the HTTP cache, independent of origin, meaning widely used frameworks would impose essentially no overhead and
avoiding the problem web apps have of triggering requests to dozens of servers and CDNs whose only real purpose is to
provide a widely agreed name for the same resource that isn't the app origin. Jigsaw embeds checksums of the modules a
module was compiled against, but I'm not sure it's a secure hash.

**Online update of Graviton itself.** Enterprises are getting more accepting of what they sometimes call "evergreen"
software i.e. software that silently updates itself outside of IT control. Graviton by default reuses the Google Chrome
auto-update engines, Omaha on Windows and the equivalent on macOS. Therefore, users are never aware of the update
process of the browser itself.

The browser-style UI design and silent auto update implies that apps may be exposed to breaking changes in the Java
platform as it evolves. Is this a critical problem? Perhaps. With a "pause" feature as outlined above for app updates,
Graviton updates that are known to break a specific app can be avoided by the app developers telling the user that they
need to temporarily opt out of updates for a while - giving them time to fix their apps. The always-evolving model is
harder work for developers, but is what they're used to and forced to accept from the web anyway, so it's unclear this
is a competitive disadvantage. On the other hand, offering a more stable underlying platform would be a competitive
advantage and it can be obtained by simply sacrificing the tabbed UI in favour of a Java Web Start approach whereby apps
get their own top level windows. This would allow old JVMs to be kept around and run in parallel.

Overall I'd rather go for the evergreen model to start with and see how much pain underlying platform churn really
causes. If starting from Java 10 or any post Jigsaw release, it would be impossible for Graviton apps to access JVM or
Graviton internals as the module system, classloaders and security system would forbid it. So whilst a few apps may be
impossible to distribute in this way, we should be in a much better situation w.r.t. evolution than Java has been in the
past.

**Multi-language support.** Graviton is not Kotlin or JavaFX specific. It should come with the Graal compiler and Truffle
backends, as GraalVM itself does. In this way apps should be authorable in JavaScript, Python, Ruby and so on, if they
depend on the right runtime modules that Graal can recognise.

Graal is on the verge of offering several features that are of particular interest:

* Support for NodeJS modules. Node apps already work, but when loading GraalJS via their API you don't get that API
  included - this is silly and I expect it will be fixed soon. Thus a subset of the Node API would be available to
  Graviton apps, and NPM modules could be easily made to work, providing a smooth on ramp for JavaScript developers who
  wish for a more rigorous API and better UI toolkit than HTML.

* Ability to impose execution time limits and interrupt execution asynchronously, to break infinite loops. This is
  effectively a compiler-supported version of the deprecated Thread.stop()  and is useful for browser style code
  sandboxing. In early versions it is acceptable for Graviton to hang in the face of a DoS attack by a malicious app - it
  is unlikely to matter for the initial use cases.

* Support for Python, Ruby and LLVM. Thus Graviton programs could conceivably utilise sandboxed modules written in C/C++,
  offering an alternative to WebAssembly.

**EGL and advanced graphics.** The Java game dev community is surprisingly large and successful - consider that Minecraft
came out of it. They would be a great market segment to target and a potentially enthusiastic userbase. For this to work
they need access to OpenGL contexts. An initial version of Graviton could offer an API to open a new OpenGL window and
expose the handle back such that it could be combined with JMonkeyEngine, LWJGL and other popular game engines. Chrome
uses an open source layer to implement EGL on top of Direct3D which improves support on Windows, and it could be
integrated into Graviton. Once this work is done an eGL surface for JavaFX apps should be relatively straightforward for
experienced systems/graphics programmers.

**Active Directory and other SSO integration.** Internet Explorer and some other browsers allow for automatic remote
sign-in based on local credentials, when the network is properly configured. It'd be nice to have this too.

Implementation plan
===================

Because none of us have any spare time, project planning and small iterations are critical. The goal is to reach the
above featureset eventually, but maybe not fast.

Many tasks can be done in parallel. Tasks are tracked using GitHub issues with labels indicating top level parallelism.
Here are some proposed tracks.

Browser and runtime updates
---------------------------

Silent background upgrades of the runtime (JVM+app browser) itself. See ":doc:`browser-update`" for more detailed design
discussion.


Module loading
--------------

Iteration 1: Write a command line tool that given a domain name, downloads a set of modules with a local HTTP cache.
Use ModuleLayer to load them, isolated from the browser internals, and initiate the app via a GravitonApp service.
If the remote modular JARs change, they are redownloaded. For now a simple manifest file can be used to list all the
JARs but this is not intended to be a long term solution. It's just a quick way to get started.

Iteration 2: Support for module streaming and execution of partially downloaded applications.

Iteration 3: Experiment with pack200 compression, with Jigsaw modules and with cached dependency resolution using
secure hashes for deduplication. That is, the module cache should not be the HTTP cache anymore when the right
metadata is present.

App shell
---------

Iteration 1: Create a top level window with a basic ``GravitonApp`` API, to allow applications to provide a ``JavaFX``
Scene. If they want to use Swing they can write a wrapper that embeds a SwingNode for their main window (or open other
windows, or both). At this point the app has to be on the classpath together with the app shell.

Iteration 2: Implement a simple address bar type UI that allows the user to specify a domain name (not a full URL), which
then looks up the app in an internal hard-coded hashmap to initiate it. The goal at this point is UI exploration and not
module loading or anything like that (this is a parallel track that can be integrated later).

Sandboxing
----------

Iteration 1: A basic sandbox is integrated into the module loader subsystem. Unrestricted TCP/IP sockets to the origin
is granted automatically. File access is provided to two app-private directories, which are located in the correct file
locations for local temporaries/caches and replicated home directories, respectively. Storage quota at this time is
uncapped. Access to files outside the private areas are forbidden.

Iteration 2: One of the benefits of getting away from the web is better integration with the file system. Direct access
to local files and directories can be granted via the PowerBox pattern (a file/directory directory chooser dialog that is
controlled by the browser and grants access to whatever is selected). Access rights to files are remembered.

Samples
-------

Iteration 1: Sample apps showing Swing, JavaFX would be nice to have and can be developed in parallel as the browser develops.

Iteration 2: Sample app showing how to use TruffleRuby and/or GraalJS.

Iteration 3: Command line apps.