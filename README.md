# serverless-workflow-hello-world Project

## My setup from scratch

```shell script
# https://kiegroup.github.io/kogito-docs/serverlessworkflow/latest/getting-started/create-your-first-workflow-service.html

# create project
quarkus create app \
    -x=kogito-quarkus-serverless-workflow \
    -x=quarkus-resteasy-jackson \
    -x=quarkus-smallrye-openapi \
    --no-code \
    org.acme:serverless-workflow-hello-world:1.0.0-SNAPSHOT

cd serverless-workflow-hello-world

# add Kogito Serverless Workflow tools extension
quarkus ext add org.kie.kogito:kogito-quarkus-serverless-workflow-devui

# to avoid podman network conflict
echo "quarkus.devservices.enabled=false" >> ././src/main/resources/application.properties

# create service flow
cat <<EOF > ./src/main/resources/hello.sw.json
{
  "id": "hello_marco",
  "version": "1.0",
  "specVersion": "0.8",
  "name": "Hello Marco Workflow",
  "description": "JSON based hello Marco workflow",
  "start": "Inject Hello Marco",
  "states": [
    {
      "name": "Inject Hello Marco",
      "type": "inject",
      "data": {
        "greeting": "Hello from Marco"
      },
      "transition": "Inject Mantra"
    },
    {
      "name": "Inject Mantra",
      "type": "inject",
      "data": {
        "mantra": "Just started to study Serverless Workflow!"
      },
      "end": true
    }
  ]
}
EOF

quarkus build

quarkus dev

curl -s -X POST -H 'Content-Type:application/json' http://localhost:8080/hello_marco | jq .

# http://localhost:8080/q/dev/
```

## installing the Serverless Workflow plug-in for Knative CLI

```shell script
curl -LO https://github.com/kiegroup/kie-tools/releases/download/0.26.0/kn-workflow-linux-amd64-0.26.0
chmod a+x kn-workflow-linux-amd64-0.26.0
sudo cp ./kn-workflow-linux-amd64-0.26.0 /usr/bin/kn-workflow
kn-workflow --help

curl -LO https://storage.googleapis.com/knative-nightly/client/latest/kn-linux-amd64
chmod a+x ./kn-linux-amd64
sudo cp ./kn-linux-amd64 /usr/bin/kn
kn plugin list
kn --help
```

## building a workflow project using Knative CLI
```shell script
# https://kiegroup.github.io/kogito-docs/serverlessworkflow/latest/tooling/kn-plugin-workflow-overview.html

kn workflow create --name my-project-hello --extension quarkus-jsonp,quarkus-smallrye-openapi
cd my-project-hello

# to avoid podman network conflict
echo "quarkus.devservices.enabled=false" >> ././src/main/resources/application.properties

# use dev.local for local tests ( --image=[registry]/[repository]/[name]:[tag] )
# kn workflow build --image my-user/my-project:1.0.0 --image-repository other-user --image-tag 1.0.1
kn workflow build --image dev.local/my-project-hello

mvn quarkus:dev

curl -s -X POST -H 'Content-Type:application/json' http://localhost:8080/hello | jq .
```

## deploy Red Hat OpenShift Serverless operator
```shell script

oc login ...

cat <<EOF | oc create -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-serverless
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: serverless-operators
  namespace: openshift-serverless
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: serverless-operator
  namespace: openshift-serverless
spec:
  channel: stable
  name: serverless-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# wait for Succeeded state
oc get csv

cat <<EOF | oc create -f -
apiVersion: operator.knative.dev/v1beta1
kind: KnativeServing
metadata:
    name: knative-serving
    namespace: knative-serving
---
apiVersion: operator.knative.dev/v1beta1
kind: KnativeEventing
metadata:
  name: knative-eventing
  namespace: knative-eventing
EOF

# wait for Ready=True
oc get knativeserving.operator.knative.dev/knative-serving -n knative-serving --template='{{range .status.conditions}}{{printf "%s=%s\n" .type .status}}{{end}}'

# wait for Ready=True
oc get knativeeventing.operator.knative.dev/knative-eventing -n knative-eventing --template='{{range .status.conditions}}{{printf "%s=%s\n" .type .status}}{{end}}'
```

## build, push, deploy the service
```shell script

kn workflow build --image quay.io/marco_antonioni/my-project-hello-sw --push

oc new-project my-project-hello-sw

kn workflow deploy

SRVURL=$(oc get services.serving.knative.dev my-project-hello-sw -o jsonpath='{.status.url}')
curl -s -X POST -H 'Content-Type:application/json' ${SRVURL}/hello | jq .

# apache bench
ab -k -n 10000 -c 5 -T 'Content-Type:application/json' ${SRVURL}/hello
```

## Serverless Logic Web Tools
https://start.kubesmarts.org

## remove Serverless crd
oc get crd -oname | grep 'knative.dev' | xargs oc delete

# ===============================================================================================

This project uses Quarkus, the Supersonic Subatomic Java Framework.

If you want to learn more about Quarkus, please visit its website: https://quarkus.io/ .

## Running the application in dev mode

You can run your application in dev mode that enables live coding using:
```shell script
./mvnw compile quarkus:dev
```

> **_NOTE:_**  Quarkus now ships with a Dev UI, which is available in dev mode only at http://localhost:8080/q/dev/.

## Packaging and running the application

The application can be packaged using:
```shell script
./mvnw package
```
It produces the `quarkus-run.jar` file in the `target/quarkus-app/` directory.
Be aware that it’s not an _über-jar_ as the dependencies are copied into the `target/quarkus-app/lib/` directory.

The application is now runnable using `java -jar target/quarkus-app/quarkus-run.jar`.

If you want to build an _über-jar_, execute the following command:
```shell script
./mvnw package -Dquarkus.package.type=uber-jar
```

The application, packaged as an _über-jar_, is now runnable using `java -jar target/*-runner.jar`.

## Creating a native executable

You can create a native executable using: 
```shell script
./mvnw package -Pnative
```

Or, if you don't have GraalVM installed, you can run the native executable build in a container using: 
```shell script
./mvnw package -Pnative -Dquarkus.native.container-build=true
```

You can then execute your native executable with: `./target/serverless-workflow-hello-world-1.0.0-SNAPSHOT-runner`

If you want to learn more about building native executables, please consult https://quarkus.io/guides/maven-tooling.

## Related Guides

- Kogito - Serverless Workflow ([guide](https://quarkus.io/guides/kogito)): Add Kogito Serverless Workflows (SW) capabilities - Includes the Process engine and Knative Eventing capabilities
- SmallRye OpenAPI ([guide](https://quarkus.io/guides/openapi-swaggerui)): Document your REST APIs with OpenAPI - comes with Swagger UI
