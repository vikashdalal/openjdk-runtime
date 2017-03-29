# Google Cloud Platform OpenJDK Docker Image

This repository contains the source for the Google-maintained OpenJDK [docker](https://docker.com) image. This image can be used as the base image for running Java applications on [Google App Engine Flexible Environment](https://cloud.google.com/appengine/docs/flexible/java/) and [Google Container Engine](https://cloud.google.com/container-engine).

This image is mirrored at both `launcher.gcr.io/google/openjdk8` and `gcr.io/google-appengine/openjdk`.

## App Engine Flexible Environment
When using App Engine Flexible, you can use the runtime without worrying about Docker by specifying `runtime: java` in your `app.yaml`:
```yaml
runtime: java
env: flex
```
The runtime image `gcr.io/google-appenine/openjdk` will be automatically selected if you are attempting to deploy a JAR (`*.jar` file).

If you want to use the image as a base for a custom runtime, you can specify `runtime: custom` in your `app.yaml` and then
write the Dockerfile like this:

```dockerfile
FROM gcr.io/google-appengine/openjdk
COPY your-application.jar app.jar
```
      
That will add the JAR in the correct location for the Docker container.
      
Once you have this configuration, you can use the Google Cloud SDK to deploy this directory containing the 2 configuration files and the JAR using:
```
gcloud app deploy app.yaml
```

##Container Engine & other Docker hosts
For other Docker hosts, you'll need to create a Dockerfile based on this image that copies your application code and installs dependencies. For example:

```dockerfile
FROM gcr.io/google-appengine/openjdk
COPY your-application.jar app.jar
```
You can then build the docker container using `docker build` or [Google Cloud Container Builder](https://cloud.google.com/container-builder/docs/).
By default, the CMD is set to run the application JAR. You can change this by specifying your own `CMD` or `ENTRYPOINT`.

## The Default Entry Point
Any arguments passed to the entry point that are not executable are treated as arguments to the java command:
```
$ docker run openjdk -jar /usr/share/someapplication.jar
```

Any arguments passed to the entry point that are executable replace the default command, thus a shell could
be run with:
```
> docker run -it --rm openjdk bash
root@c7b35e88ff93:/# 
```

## Entry Point Features
The entry point for the openjdk8 image is [docker-entrypoint.bash](https://github.com/GoogleCloudPlatform/openjdk-runtime/blob/master/openjdk8/src/main/docker/docker-entrypoint.bash), which does the processing of the passed command line arguments to look for an executable alternative or arguments to the default command (java).

If the default command (java) is used, then the entry point sources the [setup-env.bash](https://github.com/GoogleCloudPlatform/openjdk-runtime/blob/master/openjdk8/src/main/docker/setup-env.bash), which looks for supported features to be enabled and/or configured.  The following table indicates the environment variables that may be used to enable/disable/configure features, any default values if they are not set: 

|Env Var           | Description         | Type     | Default                                     |
|------------------|---------------------|----------|---------------------------------------------|
|`DBG_ENABLE`      | Stackdriver Debugger| boolean  | `true`                                      |
|`TMPDIR`          | Temporary Directory | dirname  |                                             |
|`JAVA_TMP_OPTS`   | JVM tmpdir args     | JVM args | `-Djava.io.tmpdir=${TMPDIR}`                |
|`GAE_MEMORY_MB`   | Available memory    | size     | Set by GAE or `/proc/meminfo`-400M          |
|`HEAP_SIZE_MB`    | Available heap      | size     | 80% of `${GAE_MEMORY_MB}`                   |
|`JAVA_HEAP_OPTS`  | JVM heap args       | JVM args | `-Xms${HEAP_SIZE_MB}M -Xmx${HEAP_SIZE_MB}M` |
|`JAVA_GC_OPTS`    | JVM GC args         | JVM args | `-XX:+UseG1GC` plus configuration           |
|`JAVA_USER_OPTS`  | JVM other args      | JVM args |                                             |
|`JAVA_OPTS`       | JVM args            | JVM args | See below                                   |

If not explicitly set, `JAVA_OPTS` is defaulted to 
```
JAVA_OPTS:=-showversion \
           ${JAVA_TMP_OPTS} \
           ${DBG_AGENT} \
           ${JAVA_HEAP_OPTS} \
           ${JAVA_GC_OPTS} \
           ${JAVA_USER_OPTS}
```

The command line executed is effectively (where $@ are the args passed into the docker entry point):
```
java $JAVA_OPTS "$@"
```

## The Default Command
The default command will attempt to run `app.jar` in the current working directory.
It's equivalent to:
```
$ docker run openjdk java -jar app.jar
Error: Unable to access jarfile app.jar
```
The error is normal because the default command is designed for child containers that have a Dockerfile definition like this:
```
FROM openjdk
ADD my_app_0.0.1.jar app.jar
```

# Stackdriver Logging
The [Google Cloud Stackdrive Logging](https://cloud.google.com/logging/) service may be used
by applications running in this image either from Google Cloud Platforms or any other platform with network access to the Google Cloud services. 

## Stackdriver Logging Dependencies
The application deployed on this image must provide the stackdriver logging classes and their 
dependencies.  This can be done easily in maven or gradle following [these instructions](https://cloud.google.com/logging/docs/reference/libraries#client-libraries-install-java).

## Stackdriver Logging Authentication
### Local Development
When running locally, the [Google Cloud SDK](https://cloud.google.com/sdk/) along with [Application Default Credentials](https://developers.google.com/identity/protocols/application-default-credentials) can be used to authenticate the logging client library with stackdriver logging service.  This ultimately means that after the SDK is installed, the following command is all that is required prior to running the logging client library:
```shell
gcloud auth application-default login
```
### Google Cloud Platforms
No additional authentication is required when the stackdriver logging mechanism is deployed on a Google Cloud Platform.

### Other environments
To use the stackdriver logging client from other platforms see the [Google Cloud Platform Authentication Guide](https://cloud.google.com/docs/authentication#getting_credentials_for_server-centric_flow)

## Stackdriver Logging Configuration
The Stackdriver API may be [used directly](https://cloud.google.com/logging/docs/reference/libraries#using_the_client_library) by the application, however it is often more convenient to use the [Java Util Logging](https://docs.oracle.com/javase/8/docs/api/java/util/logging/package-summary.html) handler that is provided by the libraries.

### Java Util Logging Overview

When using the [Java Util Logging](https://docs.oracle.com/javase/8/docs/api/java/util/logging/package-summary.html)(JUL) API, the configuration steps necessary to use the stack driver logging are:
 1. Instantiate and configure a [LoggingHandler](http://googlecloudplatform.github.io/google-cloud-java/0.10.0/apidocs/com/google/cloud/logging/LoggingHandler.html) instance that will send [JUL LogRecord](https://docs.oracle.com/javase/8/docs/api/java/util/logging/LogRecord.html) to the Stackdriver Logging Service.
 2. Instantiate and configure a [Formatter](https://docs.oracle.com/javase/8/docs/api/java/util/logging/Formatter.html) instance to format text messages from a LogRecord.
 3. If deploying to the Flex Google Cloud Platform environment, instantiate and configure a  [GaeFlexLoggingEnhancer](http://googlecloudplatform.github.io/google-cloud-java/0.10.0/apidocs/com/google/cloud/logging/GaeFlexLoggingEnhancer.html) to add additional information to each log entry (eg. traceId, projectId etc.).  Note that developers may provide their own [Logging.Enhancer](http://googlecloudplatform.github.io/google-cloud-java/0.10.0/apidocs/com/google/cloud/logging/LoggingHandler.Enhancer.html) implementation to enhance log entries for other environments. 

Whilst JUL configuration can be done to a limited extent via programmatic APIs, for most purposes it is far simpler to achieve the above configuration by providing a `logging.properties` file like:

```properties
.level=INFO

handlers=com.google.cloud.logging.LoggingHandler
com.google.cloud.logging.LoggingHandler.level=FINE
com.google.cloud.logging.LoggingHandler.log=my_app.log
com.google.cloud.logging.LoggingHandler.formatter=java.util.logging.SimpleFormatter
java.util.logging.SimpleFormatter.format=%3$s: %5$s%6$s

## uncomment the following lines if running of GCP Flex
# com.google.cloud.logging.LoggingHandler.resourceType=gae_app
# com.google.cloud.logging.LoggingHandler.enhancers=com.google.cloud.logging.GaeFlexLoggingEnhancer
```

### Providing `logging.properties` via a custom image
If this image is being used as the base of a custom image, then the following `Dockerfile` commands can be used to add a `logging.properties` file and to set the system property to detect it:
```Dockerfile
FROM gcr.io/google-appengine/openjdk
ADD logging.properties /etc/logging.properties
ENV JAVA_USER_OPTS -Djava.util.logging.config.file=/etc/logging.properties
...
```

### Providing `logging.properties` via docker run 
A `logging.properties` file may be added to an existing images using the `docker run` command if the deployment environment allows for the run arguments to be modified. The `-v` option can be used to bind a new `logging.properties` file to the running instance and the `-e` option can be used to set the system property to point to it:
```shell 
docker run -it --rm \
-v /mylocaldir/logging.properties:/etc/logging.properties \
-e JAVA_USER_OPTS="-Djava.util.logging.config.file=/etc/logging.properties" \
...
```

### Providing `logging.properties` via the classpath 
If this image is being used by tools that automatically bundle a java application, then a `logging.properties` file may be added to the image as a resource within a jar file on the JVM classpath.   To read the configuration file from the classpath, the following class needs to be instantiated:
```java
import java.io.InputStream;
import java.util.logging.LogManager;
import java.util.logging.Logger;

public class ConfigJUL {
  public ConfigJUL() {
    try (final InputStream is = getClass().getResourceAsStream("/logging.properties")) {
      LogManager.getLogManager().readConfiguration(is);
    }
    catch (Exception ex) {
      ex.printStackTrace();
    }
  }  
}

```
This class needs to be instantiated early so that logging is not accidentally initialized with default configuration by other components within the application. The best way to achieve this is by setting the `java.util.logging.config.file` system property to the fully qualified class name (`ConfigJUL` in this case), which for GCP can be done in `app.yaml`:
```yaml
env_variables:
  JAVA_USER_OPTS="-Djava.util.logging.config.class=ConfigJUL
```
Setting the `JAVA_USER_OPTS` can also be done within a `Dockerfile` or via `docker run` command as shown above.


### Enhanced Stackdriver Logging
When running on the Google Cloud Platform Flex environment, the Stackdriver logging can be enhanced with additional information about the environment by adding the following lines to the `logging.properties`: 
```
com.google.cloud.logging.LoggingHandler.resourceType=gae_app
com.google.cloud.logging.LoggingHandler.enhancers=com.google.cloud.logging.GaeFlexLoggingEnhancer
```
This enables the [GaeFlexLoggingEnhancer](http://googlecloudplatform.github.io/google-cloud-java/0.10.0/apidocs/com/google/cloud/logging/GaeFlexLoggingEnhancer.html).  The logging generated can then be linked to the `nginx` request log in the logging console by calling [setCurrentTraceId](http://googlecloudplatform.github.io/google-cloud-java/0.10.0/apidocs/com/google/cloud/logging/GaeFlexLoggingEnhancer.html#setCurrentTraceId-java.lang.String-) for any thread handling a request.  The traceId for a request on a Google Cloud Platform is obtained from the `setCurrentTraceId` HTTP header as the first field of the `'/'` delimited value.

# Development Guide

* See [instructions](DEVELOPING.md) on how to build and test this image.

# Contributing changes

* See [CONTRIBUTING.md](CONTRIBUTING.md)

## Licensing

* See [LICENSE.md](LICENSE)
