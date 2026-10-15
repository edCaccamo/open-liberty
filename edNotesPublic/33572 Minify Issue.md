---
tags:
  - issue
  - minify
  - package
  - client
Date: 2026-10-14
Status: In Progress
Issue Link: https://github.com/OpenLiberty/open-liberty/issues/33572
Github Branch/PR:
---
---
# Minify Issue 33572

## The Issue

When packaging client with minify option. The following messages are produced:
```bash
Packaging client <client name>.
Querying server <client name> for content.
                    Java home:  <JAVA_HOME>
                    Install root:  <WLP_INSTALL_ROOT>
                    System libraries:  <WLP_INSTALL_ROOT>/lib/
                    User root:  <WLP_INSTALL_ROOT>/usr/
                    Config:  <WLP_INSTALL_ROOT>/usr/servers/<client name>/
                    Output:  <WLP_INSTALL_ROOT>/usr/clients/<client name>/

CWWKE0076E: Unable to query server <client name> to determine features to retain.
Client <client name> package failed. Check client logs for details.
```

Changing the config dir to `clients/<client name>` does not resolve the issue. suspect it goes to read server.xml, instead of client.xml

## Notes

### What is Minify?

The package command in OL creates an archive (.zip, .jar, .pmax etc) of a server or application client runtime. By default (--include=all) packaging includes the entire Liberty installation (wlp/ directory) which contains every feature, bundle, and library shipped with Liberty.

``--include=minify`` option create a minimal trimmed down runtime tailored specifically to the configuration and applications of that instance.
- Headless Provisioning Query: Liberty temporarily starts an embedded headless kernel instance in query mode by setting the property. ``REQUEST_SERVER_CONTENT_PROPERTY=1.0.0`` 
- Feature tree resolution: The kernel boots up and parses the config file (server.xml or client.xml). The `FeatureManager` resolves dependancy of OSGi bundles, feature manifests etc. but prevents raising the OSGi start level to full application execution.
- Content whitelist generation: The `ServerContentHelper` registers a `ServerContent` service, returning the complete list of required file paths within `wlp/`.
- Targeted Packaging: The `PackageProcessor` assembles the archive containing only those whitelisted runtime files plus the user config directory dramatically reducing image/archive size.

### Example of Failure:

SW Engineer using Liberty is building a standalone Java application client that communicates with an Open Liberty server.

They create the client
```zsh
./client create orderClient
```

They configure features in `client.xml` to include what they want.

They try to package it for distribution with `--include=minify` enabled. 
```zsh
./client package orderClient --include=minify --archive=orderClient.zip
```

Instead of producing a compact zip as expected, the command immediately crashes with:
```bash
Packaging client orderClient.
Querying server orderClient for content.
    Config:  <WLP_INSTALL_ROOT>/usr/servers/orderClient/
    Output:  <WLP_INSTALL_ROOT>/usr/clients/orderClient/

CWWKE0076E: Unable to query server orderClient to determine features to retain.
Client orderClient package failed. Check client logs for details.
```

#### Why

To create a minified package, Liberty needs to figure out which jar files and features are actually being used. It does this by temporarily booting a shadow embedded process in memory to inspect the config.

- CLI knows it's a client.
	- The `client package` correctly knows it is handling a client (`orderClient` in `usr/clients/)
- The minify helper assumes everything is a server:
	- When the code asks the background helper (`EmbeddedServerImpl`) to boot and check dependencies, the helper hardcodes server behavior. It ignores the fact that it was called from a client command.
- It looks in the wrong place
	- It searches `usr/servers/orderClient/server.xml`
	- It should search `usr/clients/orderClient/client.xml`
- Because the first doe snot exist, the embedded process fails to start leading to the `CWWKE0076E`

Essentially EmbeddedServer is hardcoded despite being client. It also lacks the code for client, as 'PackageProcessor.execute' purposely excludes clients from minify via a guard `bootProps.getProcessType*() != BootstrapConstants.LOC_PROCESS_TYPE_CLIENT` . 


## The Fix

Changes Made:

Deferred Initialization of `bootProps` in `EmbeddedServerImpl.java`
- This is so it doesnt default to standard servers.xml and can use client.xml. 

`EmbeddedServiceImpl` added an overloaded Constructor so it now accepts a `BootstrapConfig` instance. Null defaults to what it was before.

In `PackageCommand.java` pass `bootProps` in so that when launched in client package it uses `ClientBootstrapConfig` to resolve proper paths to client.xml instead of server.xml.

In `PackageProcessor.java`, removed the client exclusion guard (`bootProps.getProcessType() != BootstrapConstants.LOC_PROCESS_TYPE_CLIENT`) so client packages can assemble minified runtime archives.

Added unit test in `PackageProcessorTest.java` (`testClientMinifyPackageProcessor`) to verify client minify packaging.

## Verification if applicable

First Test run (after building gradlew)
```zsh
BUILD SUCCESSFUL in 3m 42s
5427 actionable tasks: 2182 executed, 3245 up-to-date
$ cd dev/build.image/wlp
$ ./bin/client create testClient
Client testClient created.
$ ./bin/client package testClient --include=minify --archive=testClient-min.zip
Packaging client testClient.
Querying server testClient for content.
Launching testClient (Open Liberty 26.0.0.11/wlp-1.0.119.202610141532) on Eclipse OpenJ9 VM, version 21.0.12+8-LTS (en_US)
[AUDIT   ] CWWKE0907I: The client testClient has been launched.
[AUDIT   ] CWWKF0034I: The client installed the following features: [beanValidation-1.1, cdi-1.2, el-3.0, javaMail-1.5, javaeeClient-7.0, jaxb-2.2, jdbc-4.1, jndi-1.0, jpa-2.1, jpaContainer-2.1, jsonp-1.0, managedBeans-1.0, wasJmsClient-2.0].
[AUDIT   ] CWWKF0026I: The server testClient is ready to build a smaller package.
[ERROR   ] CWWKE0916E: CWWKE0916E: The client feature was not enabled. Review the error messages.
[AUDIT   ] CWWKE1103I: Waiting for up to 30 seconds for the client to quiesce.
[AUDIT   ] CWWKE0908I: The client testClient stopped after 1.913 seconds.
Building archive for server testClient.
Client testClient package complete in <WLP_INSTALL_ROOT>/usr/clients/testClient/testClient-min.zip.
$ cd usr/clients/testClient
$ ls
client.xml              logs                    testClient-min.zip      workarea
$ unzip -l testClient-min.zip | grep "usr/clients/testClient"
        0  10-14-2026 15:43   wlp/usr/clients/testClient/
        0  10-14-2026 15:43   wlp/usr/clients/testClient/workarea/.sLock
      170  10-14-2026 15:43   wlp/usr/clients/testClient/client.xml
$ cd ../../..
$ ./bin/server create testServer

Server testServer created.
$ ./bin/server package testServer --include=minify --archive=testServer-min.zip

Packaging server testServer.
Querying server testServer for content.
Launching testServer (Open Liberty 26.0.0.11/wlp-1.0.119.202610141532) on Eclipse OpenJ9 VM, version 21.0.12+8-LTS (en_US)
[AUDIT   ] CWWKE0001I: The server testServer has been launched.
[AUDIT   ] CWWKF0012I: The server installed the following features: [el-3.0, jsp-2.3, servlet-3.1].
[AUDIT   ] CWWKF0026I: The server testServer is ready to build a smaller package.
[AUDIT   ] CWWKE1100I: Waiting for up to 30 seconds for the server to quiesce.
[AUDIT   ] CWWKE0036I: The server testServer stopped after 1.665 seconds.
Building archive for server testServer.
Server testServer package complete in <WLP_INSTALL_ROOT>/usr/servers/testServer/testServer-min.zip.
```

What this says

- Original `CWWKE0076E` is completely resolved:
```zsh
Packaging client tesetClient.
Querying server tesetClient for content.
[AUDIT ] CWWKE0907I: The client tesetClient has been launched.
```

Instead of searching for server.xml and crashing, it launches correctly.

- Feature Manager resolved client features
```zsh
[AUDIT ] CWWKF0034I: The client installed the following features: [beanValidation-1.1, cdi-1.2, el-3.0, javaMail-1.5, javaeeClient-7.0, jaxb-2.2, jdbc-4.1, jndi-1.0, jpa-2.1, jpaContainer-2.1, jsonp-1.0, managedBeans-1.0, wasJmsClient-2.0].
[AUDIT ] CWWKF0026I: The server tesetClient is ready to build a smaller package.
```
Meaning it configured feature tree from client.xml.

- Archives were assembled and verified, and Server minify verified with zero regressions.

#### Additional Local Tests.

Run outside of wlp:
```bash
[AUDIT ] CWWKE0907I: The client tesetClient has been launched.
[AUDIT ] CWWKZ0058I: Monitoring dropins for applications.
[AUDIT ] CWWKF0034I: The client installed the following features: [...]
[AUDIT ] CWWKF0035I: The client tesetClient is running.
```

minify zip is actually smaller than includes=all
```zsh
$ ls -lh testClient-min.zip all.zip
-rw-r-----@ 1 user  staff   448M Oct 14 16:03 all.zip
-rw-r-----@ 1 user  staff    71M Oct 14 15:43 testClient-min.zip
```

Other includes tags still work
```zsh
$ ./bin/client package testClient --include=usr --archive=usr.zip
$ ./bin/client package testClient --include=all --archive=all.zip
Packaging client testClient.
Client testClient package complete in <WLP_INSTALL_ROOT>/usr/clients/testClient/usr.zip.
Packaging client testClient.
Client testClient package complete in <WLP_INSTALL_ROOT>/usr/clients/testClient/all.zip.
```

gradlew com.ibm.ws.kernel.boot_fat:buildandrun
```zsh
BUILD SUCCESSFUL in 36.5s
```
