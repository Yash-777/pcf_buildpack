# pcf_buildpack

Buildpack is a mechanism  used by cloud foundry (CF) to create a droplet for a standalone application.
CF droplet is a binary artifact that contains the application to be deployed and all its dependencies.

This repository provides a quick introduction to buildpacks and sample code showing implementation of a custom buildpack.

##  What is covered?

-   What is a buildpack

-   Buildpack v/s Docker approach 

-   Structure of a buildpack

-   Accessing buildpacks

-   Using multiple buildpacks during deployment

-   Useful links

-   Code sample 

## What is a buildpack?

-   Mechanism to create a droplet
-   Droplet is a binary artifact created to stage the pushed application
-   Droplet = application code + runtime components + dependencies
-   Droplet + file system = A container similar to any OCI compatible container like Docker

#### List available buildpacks

You can run following command to list all available build packs.

- cf buildpacks it will show an output something like below

```aidl

cf buildpacks

Getting buildpacks...

buildpack                              position   enabled   locked   filename
meta_buildpack                         1          true      false    meta_buildpack-v3.0.0.zip
java_buildpack_offline                 2          true      false    java-buildpack-offline-cflinuxfs2-v4.12.1.zip
java_buildpack_offline-3_15            6          true      false    java-buildpack-offline-v3.15.zip
java_buildpack_offline-4_4             7          true      false    java-buildpack-offline-v4.4.zip
ruby_buildpack                         9          true      false    ruby_buildpack-cached-cflinuxfs2-v1.7.19.zip
ruby_buildpack_test                    10         true      false    ruby_buildpack-cached-v1.6.47.zip
nodejs_buildpack                       11         true      false    nodejs_buildpack-cached-cflinuxfs2-v1.6.25.zip
staticfile_buildpack                   13         true      false    staticfile_buildpack-cached-cflinuxfs2-v1.4.28.zip
nodejs_buildpack_cached-1_6_6          14         true      false    nodejs_buildpack-cached-v1.6.6_1502722293.zip
python_buildpack                       15         true      false    python_buildpack-cached-cflinuxfs2-v1.6.17.zip
staticfile                             16         true      false    staticfile_buildpack-cached-v1.0.0.zip
java_buildpack_offline-3_1             17         true      false    java-buildpack-offline-v3.1.1.zip
binary_buildpack                       18         true      false    binary_buildpack-cached-v1.0.18.zip
dotnet-core_buildpack-cached-v1_0_25   19         true      false    dotnet-core_buildpack-cached-v1.0.25+1503436559.zip
go_buildpack                           20         true      false    go_buildpack-cached-cflinuxfs2-v1.8.23.zip
php_buildpack                          21         true      false    php_buildpack-cached-cflinuxfs2-v4.3.56.zip
dotnet_core_buildpack                  22         true      false    dotnet-core_buildpack-cached-cflinuxfs2-v2.0.7.zip
microgateway_decorator                 24         true      false    microgateway_decorator-v3.0.0.zip
staticfile_buildpack_1_3_18            25         true      false    staticfile_buildpack-cached-v1.3.18+1487975051.zip
tibco-bwce-buildpack                   26         true      false    bwce233buildpackwithMECL.zip.

.......
.......

```
 
Above is also an ordered list in  which the CF will try to find a match using detect script in each BP.

## Buildpack v/s Docker approach 

CF supports both approaches to application deployment 
-   First where developer creates a container ready with application and all its run time and dependencies prepackaged and reay to deploy.
-   Second where developer only supplies and application to deploy and CF creates a container.

However there are few trade-offs between the two approaches as described below,


| No. | Buildpacks | Docker |
|:---:|:---|:---|
|1    | Provides clean separation of responsibilities between platform operator and developer. Operator provides the runtime versions for OS, Languages and other dependencies where as developer supplies only the application compatible with the platform.    | In case of docker either the developer or operator may provide the container image and all application dependencies. In such cases standardization and control may become bigger challenge than the buildpack approach |
|2    | Developers can truly focus on application business logic and business requirements increasing velocity of app's business code.   | Developers need to also work on undifferentiated tasks like building containers and tinkering with middleware |
|3    | Buildpack approach provides for additional security and operational benefits. E.g. as and when common vulnerabilities and exposures (CVE) are detected then operators can patch the build packs once and all applications need to be only re-staged a task that can be automated.| In case of Docker approach each container image will need to be individually patched, recreated and redeployed.|
|4    | Building and running of a droplet happens on two separate containers. The container created to run the droplet dose not need to include any build tools further reducing the attack surface.| It may become difficult to achive in case of docker approach.|
|5    | CF Push to each individual dev test stage and prod env technically creates a new droplet every time and  which means there is a small risk of dependency updates and changes between the environments.| In case of docker same exact image can be promoted to all environments.|
|6    | This approach more difficult to test locally than docker way.| Docker images work well in desktop environment and easier to work with and test locally.|
|7    | It is easier for applications to consume the declarative services installed on the platform. Just declaring a dependency by service name in manifest is sufficient and the discovery and  connection happens on platform. | Where as in case of docker image app consuming platform services some additional work may be needed on part of apps.
 
## Structure of a buildpack

Structurally a buildpack contains a bin directory with either 3 ( detect|release|compile ), (detect|release|supply)
OR 4 (detect|release|supply|finalize) of following 5 scripts.

|Script|Purpose|Arguments|Execution|Return value|
|:----|:----|:----|:----|:----|
|bin/detect   | Determines whether or not a buildpack applies to the app being pushed.|'build' dir for the app. | Called when application is pushed without specifying a buildpack. build dir contains app files uploaded with cf push.|Returns 0 if match i.e. can positively identify the app. Nonzero on mismatch. E.g. if it is Java buildpack and there are no .java files in app it returns a non zero exit status indicating java build is not applicable for the app. And detection proceeds in predertermined order to other build packs on CF.|
|bin/supply   | Downloads or otherwise provides the dependencies for an app. This script is part of non-final buildpacks.|Takes 4 arguments, 'build' dir as above, 'cache' dir to store assets during build, 'deps' dir where dependencies from all BPs in chain are kept, and index representing position of this BP in chain. | Supply stores it dependencies in deps/index, can read other deps/index but should not try to modify anything outside its own deps/index dir. cache of only the final BP is preserved even if BP changes. For others cache is cleared if BP changes or is updated.|All output is sent to STDOUT and shown to user through CF CLI.|
|bin/finalize | Prepares an application for launch.|Same 4 args as supply script above with same purpose.|same as supply.|All output is sent to STDOUT and shown to user through CF CLI.|
|bin/release  | Provides feedback and metadata to CF indicating how the app is to be executed. This script is part of build pack run as final or the last buildpack in order. |only one arg 'build' dir|Generates output YAML in following format, default_process_type indicating type of app being run and start command for the app.Currently only web type apps are supported. |All output is sent to STDOUT and shown to user through CF CLI.|
|bin/compile  | Deprecated alternative for supply and finalize going forward from release (TBD). This script is part of final buildpack if present.|takes only two args 'buid' and 'cache'| Does not support multi buildpacks. Does function of both supply and finalize.|All output is sent to STDOUT and shown to user through CF CLI.|

Only one buildpack can be a final and it has commands to start the app.
 
Environment vars fr BP can be supplied in a script under profile.d dir in root folder of app 

Web type commands run as bash -c \<COMMAND\> 

## Accessing buildpacks

Access it 3 ways which present trade-offs between control/governance and convenience.

1. Online : where BP and dependencies are both remote.

2. Minimal-package : where minimal packaged version of BP is uploaded to CF and all dependencies are accessed remotely

3. Offline-package: where BP can run without network access, all dependencies and BP is uploaded to CF and remote download is disabled.


## Using multiple buildpacks during deployment

Following commad shows how a cf cli (CLI v6.38 or later) is used to push an app with multiple BPs

```aidl
cf push YOUR-APP -b BUILDPACK-NAME-1 -b BUILDPACK-NAME-2 ... -b FINAL-BUILDPACK-NAME

```


## Useful links

- https://docs.cloudfoundry.org/buildpacks/

- https://docs.cloudfoundry.org/buildpacks/understand-buildpacks.html#droplet-filesystem

- https://docs.cloudfoundry.org/buildpacks/understand-buildpacks.html#buildpack-detection

- https://docs.cloudfoundry.org/buildpacks/understand-buildpacks.html#interactions

## Code sample 
  