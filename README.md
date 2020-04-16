# openshift-configmap-secrets-exercise
A simple exercise in configuring a microservice with a DB dependency through config maps and secrets.

## Goals
* Deploy a PostgreSQL database from a built-in Openshift template.
* Deploy a micro service, a Spring Boot app from a prebuilt Docker image.
* Configure the micro service's dependencies to the database with config maps and secrets.


## Steps
### Background
Briefly review the source of the Spring Boot app we are going to deploy from a prebuilt Docker image:
  https://github.com/svejk-ciber/yaapp

This is a simple Spring Boot app using Spring Data JDBC and Spring Data Rest to supply a minimalistic 
CRUD interface to a single table. It builds with Maven, and uses the `jib`plug-in to produce an executable Docker image.

The prebuilt Docker image we are going to deploy is published at:
svejkciber/yaapp:9af679e1258e3ca9a1baca4a4cf2f11da9374c09 (at Docker Hub)

### Setup
1. Log in with the CLI to Openshift as a developer user.
2. Create a new project, e.g. `yaaap`.
3. Establish a couple of environment variables for the DB credentials you set up PostgreSQL with:
   ```
   $ export POSTGRESQL_USER=postgresql
   $ export POSTGRESQL_PASSWORD=<YOUR CHOICE>
   ```

### Deploy a PostgreSQL instance our micro service can use from a built-in Openshift template
1. Run the `oc new-app` command on a built-in template with three parameters:
   ```
   $ oc new-app postgresql-ephemeral --param=POSTGRESQL_USER=$POSTGRESQL_USER \
        --param=POSTGRESQL_PASSWORD=$POSTGRESQL_PASSWORD \
        --param=POSTGRESQL_DATABASE=yaaap-db
   ...
   --> Creating resources ...
    secret "postgresql" created
    service "postgresql" created
    deploymentconfig.apps.openshift.io "postgresql" created
    --> Success
        Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose svc/postgresql'
    Run 'oc status' to view your app.
   ```
   There is no need to expose a HTTP port for PostreSQL.
1. Check that PostgreSQL is running
  1. Run `$ oc status` as suggested.
     This should not indicate errors. The _info_ identified is about missing health checks, 
     and can be safely ignored at this point.
  1. Check the logs of the running pod (`oc get pods`, ` oc logs <POD-NAME>`) 

### Deploy the Spring Boot app from a Docker image
1. Deploy the app from an existing Docker image
   ```
   $ oc new-app svejkciber/yaapp:9af679e1258e3ca9a1baca4a4cf2f11da9374c09
   --> Found container image 100bad7 (50 years old) from Docker Hub for "svejkciber/yaapp:9af679e1258e3ca9a1baca4a4cf2f11da9374c09"
   ...
   --> Success
    Run 'oc status' to view your app.
   ```

This is going to fail, as the Spring Boot properties for the DB connection are not properly set up yet.
Verify that the deployment failed by checking the logs of the pod being created and, and the project 
status (`oc status`). After a while the project status and pod information (`oc get pods`) indicates 
that the pod is crash-looping.

### Configure DB connection with config maps and secrets
We are going to fix the failed start-up of the micro service by adding the required configuration as `ConfigMaps` and `Secrets`. These are the preferred methods in Openshift and Kubernetes of configuring applications without redeployment.
From looking at `application.properties` in the source and the logs of the failing pods, we find that the settings of the micro service are:
* spring.datasource.url
* spring.datasource.username
* spring.datasource.password

1. Specify the data source url in a configmap:
   ```
   $ oc create configmap yaapp --from-literal=SPRING_DATASOURCE_URL=jdbc:postgresql://postgresql:5432/postgres
   configmap/yaapp created
   ```
   Explanation of the URL modification: We just changed the host name part of the data source url, from the default
   `localhost` to `postgresql` above. The host name `postgresql` is not a real DNS name, it is just a Kubernetes service 
   name, corresponding to the one create from the Postgresql template above. Kubernetes/Openshift uses _service discovery_ 
   to map such names into cluster-internal IP addresses, when pods refer to each other.
   
1. Inspect the newly create config map:
   ```
   $ oc describe configmap yaapp
   Name:         yaapp
   ...
   Data
   ====
   SPRING_DATASOURCE_URL:
   ...
   ```
1. Attach the new config map as environment variables of the deployment config:
   ```
   $ oc set env dc/yaapp --from configmap/yaapp
   deploymentconfig.apps.openshift.io/yaapp updated
   ```
   This configuration change will trigger a redeploy of the pod (and also recreation of the replication controller).
   Verify that by checking the output of `oc get pod -w`.
   
1. Verify progress in failing application logs.
After a while this deployment also fails with `ChrashLoopBackState`. Verify that we made some progress in the 
application log. The logs should indicate that we got past the previous connection errors, but are now left with authentication errors against the database.

1. Create a secret with the DB credentials
   ```
   $ oc create secret generic yaapp --from-literal=SPRING_DATASOURCE_USERNAME=$POSTGRESQL_USER \
        --from-literal=SPRING_DATASOURCE_PASSWORD=$POSTGRESQL_PASSWORD
   secret/yaapp created    
   ```
1. Inspect the newly created secret:
   ```
   $ oc describe secret yaapp
   Name:         yaapp
   ...
   ```
   It is possible to view more resource details with `oc get secret <NAME> -o json`
1. Attach the secret contents as environment variables in the deployment config:
   ```
   $ oc set env dc/yaapp --from secret/yaapp
   deploymentconfig.apps.openshift.io/yaapp updated
   ```
1. Verify that this redeployment finally works (`oc status`, `oc get pods -w`, `oc logs <POD-NAME`).
   The application log should not show any errors anymore.

### Test the micro service
1. Expose the HTTP interface of the app:
   ```
   $ oc expose dc/yaapp --port=8080
   service/yaapp exposed
   $ oc expose svc yaapp
   route.route.openshift.io/yaapp exposed
   ```
   _NB_: The first of these commands is a workaround for a deficiency in the prebuilt Docker image:
   it was built without metainformation required by Openshift to describe exposed ports. It is normally 
   not necessary to create a default HTTP service manually.
   
 1. Open the web page of the app, and verify that you see a REST API generated by Spring Data.
    The URL of the app can be found by getting or describing the HTTP service route:
    ```
    $ oc get route
    NAME    HOST/PORT                                                        PATH   SERVICES   PORT   TERMINATION   WILDCARD
    yaapp   yaapp-default.2886795321-80-host03nc.environments.katacoda.com          yaapp      8080      None
    ```
   
## Bonus
If time permits, you may try one or more of the following:
1. How secret are Openshift secrets? Run base64encode to decode values from `oc get secret ... -o json`
   and judge for yourself.
1. Build the Spring Boot app from source in Openshift, that is, use the Git repo of the app as an input
   source to a BuildConfig.
1. A build like the above, will probably not use the existing jib configuration in the `pom.xml` file. 
   It will likely build a default Docker image from the Spring Boot JAR. You can try to fix that by forking the 
   repo, and figure out how to configure Jib to publish to an internal Docker registry when building on Openshift. 
1. Add missing metainformation to the Docker image.   
