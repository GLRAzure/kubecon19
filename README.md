# KubeCon 2019 Highlights

This repo contains a script file (scripts.md) and a number of YAML files to illustrate how various services, apps, and resources were provisioned.

The scripts.md file walks through steps from provisioning an AKS cluster, to associating it with an ACR, to setting up GitOps with Flux, Linkerd as a Service Mesh, Kong and nginx ingress controllers, Flagger for canary and blue/green deploys, and KEDA for killer HPA stuff.

The majority of the YAML files involve the deployment of a sample app called Hipster Store, which can be found here (https://github.com/GoogleCloudPlatform/microservices-demo). Hipster is a microservice app, each microservice written in a different language, using gRPC for service-to-service communication.

These files are still a work in progress and can change over time as additional services are added/removed. But it should provide a sense of what's involved in setting up services like service mesh, gitops, KEDA, etc.
