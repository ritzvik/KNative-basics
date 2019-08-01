# KNative Basics

## Why do we need KNative

  - To run containerized apps which can scale infinitely on load, and can scale to zero when there is no traffic
  - To build apps inside the kubernetes cluster
  - To build CI/CD pipeline
  - To run conatiners which get created only on specific triggers

## Installation and demonstrations

  - [Basic installation and Hello World](knative-hello-world.md)
  - [Build component of KNative - basic](build/build.md)
  - [Build component of KNative - upload to a docker registry](build-and-upload/build-and-upload.md)
  - [KNative autoscaling and revisions](knative-revisions/knative-revisions.md)

## Sources and tutorials

  - [KNative official docs](https://knative.dev/docs/)
  - [Introducing Knative: Build, Deploy, and Manage Serverless Workloads on Kubernetes by David Currie](https://www.youtube.com/watch?v=AIDKDLxiCdk)
  - [KNative documentation repo](https://github.com/knative/docs)

## Limitations

  - Very unstable software (frequent API changes).
    - For example, configuration file written for `v0.6.0` won't work for `v0.7.0` and vice-versa.
  - No stable release (There is a long way to `v1.0`)
  - Very strict dependency requirements
    - See [Basic installation and Hello World](knative-hello-world.md)
  - Unintuitive.
  - The software seems promising but isn't mature enogh for production environment.

