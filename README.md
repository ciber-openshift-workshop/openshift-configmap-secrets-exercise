# openshift-configmap-secrets-exercise
A simple exercise in configuring a microservice with a DB dependency through config maps and secrets.

## Goals
# Deploy a PostgreSQL database from a built-in Openshift template.
# Deploy a micro service, a Spring Boot app from a prebuilt Docker image.
# Configure the micro service's dependencies to the database with config maps and secrets.


## Steps
### Background
Briefly review the source of the Spring Boot app we are going to deploy from a prebuilt Docker image.
The source is at:
https://github.com/svejk-ciber/yaapp
This is a simple Spring Boot app using Spring Data JDBC and Spring Data Rest to supply a minimalistic 
CRUD interface to a single table. It builds with Maven, and uses the `jib`plug-in to produce an executable Docker image.

The prebuilt Docker image we are going to deploy is published at:
svejkciber/yaapp:9af679e1258e3ca9a1baca4a4cf2f11da9374c09 (at Docker Hub)

### Setup
1. Log in with the CLI to Openshift as a developer user.
2. Create a new project, e.g. `yaaap`.

### Deploy a PostgreSQL instance our micro service can use from a built-in Openshift template


### Deploy the Spring Boot app from a Docker image

This is going to fail, as the Spring Boot properties for the DB connection are not properly set up yet.


## Bonus
If time permits, you may try one or both of the following:
1. Build the Spring Boot app from source in Openshift, that is, use the Git repo of the app as an input
   source to a BuildConfig.
