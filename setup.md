# Setting up the Lambda Engine (using docker) #
Using the Lambda Engine docker image requires a running instance of docker (see https://docs.docker.com/get-docker/)

## Getting the image ##
A docker image can either be pulled from a remote image repository or be distributed manually as an archive file. 

To pull the image from a remote image repository use:

```docker pull [image_repository]/[image_name]:[tag] ```

Change the url to the image repository, the name of the specific image and the release tag of the wanted version.
The command then may look like:

``` docker pull ghcr.io/morgena/lambda-engine:latest  ```

Alternatively, to manually load an archive into your local docker image repository use the following command and replace the path with the path of your locally stored image archive file.

``` docker load -i /path/lambda-engine.tar ```

## Starting the container ##

``` docker container run -d --name "lambda-engine" --volume /host/path:/workspace/data -p 8080:8080 gitlab.uni-marburg.de:5050/fb12/ag-seeger/lambda-engine:latest  ```

This command creates, based on the image "../lambda-engine:latest" that was loaded into the local image repository in the previous step, a container with the name "lambda-engine" and runs it. In this container the internal folder `/workspace/data` gets mounted to the specified path in the host file system and the `internal port 8080` of the image gets published to the `local port 8080`.

{{< hint type=note >}}
Data inserted into the Lambda Engine is not stored in the container itself. All data like created streams or inserted events are stored in a folder `data` that is always a mounted volume. If no volume is specified at the first run, per default an unnamed volume is created.
{{< /hint >}}
 
As a quick test with a browser, visit `http://localhost:8080/chronicledb/streams`. There should be a (for now) empty list of existing streams displayed.

{{< hint type=note >}}
Instead of mounting a path in the file system a named virtual volume managed by docker can be bound to the internal folder.
Use `--volume lambda_data:/workspace/data` to create a virtual volume named `lambda_data`. In general, its best to use on Windows a mounted volume, while on Linux a mounted folder will result in the best performance.
{{< /hint >}}

{{< hint type=note >}}
For simplicity it is recommended to use the same internal port 8080 as the locally published port. Nonetheless any available local port can be published, for example port 8081 by using `-p 8081:8080`.
{{< /hint >}}

## Using the container ##
With the commands `start`, `stop` or `rm` the container and by that the lambda-rest-service can be started, stopped and deleted.

``` docker container start|stop|rm lambda-engine ```

{{< hint type=note >}}
Even if the container is removed, a bound volume or the contents of a mounted folder will not be deleted. This allows the deployment of a new version of the Lambda Engine without losing any stored data.
{{< /hint >}}

For more ressources see https://docs.docker.com/get-started

# Setting up the Lambda Engine (locally) #

## Installation ##
1. Apply for access to the event processing git repositories.
2. Install **Event Processing Commons**
	- Create a local copy of the Git Repository ("git clone ...").
	- Install locally ("mvn clean install")
3. Install **Java Event Processing Connectivity (JEPC)**
	- Create a local copy of the Git Repository ("git clone ...").
	- Install locally ("mvn clean install").
4. Install **ChronicleDB**
	- Create a local copy of Git Repository ("git clone ...").
	- Install locally ("mvn clean install")
5. Install **Lambda Engine**
	- Create a local copy of the Git Repository ("git clone ...").
	- Install locally ("mvn clean install")

{{< hint type=note >}}
All git repositories can be found at the [GitLab](https://gitlab.uni-marburg.de) of the University of Marburg.
{{< /hint >}}

## Starting the REST Server ##
1. The rest server is started by executing the class `en.umr.lambda.rest.LambdaRestApp` from the project `lambda-engine/lambda-rest`
2. The application looks for the configuration file 'application.properties' in the following paths:
	- $CURRENT_DIRECTORY/application.properties
  	- $CLASSPATH/application.properties

This file contains settings such as the port on which the server listens and the directory where the saved streams are stored. An example configuration can be found in the directory `lambda-engine/lambda-rest/examples`.

{{< hint type=note >}}
For logging via log4j2 the path to the configuration file (usually `log4j2.xml`) must be passed as argument to the JVM:
`java -Dlogging.config="log4j2.xml" -jar $PATH/lambda-rest-service-$VERSION.jar`
{{< /hint >}}
