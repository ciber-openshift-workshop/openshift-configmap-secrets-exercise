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
        --param=POSTGRESQL_PASSWORD=$POSTGRESLQ_PASSWORD \
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

This is going to fail, as the Spring Boot properties for the DB connection are not properly set up yet.


## Bonus
If time permits, you may try one or both of the following:
1. Build the Spring Boot app from source in Openshift, that is, use the Git repo of the app as an input
   source to a BuildConfig.
