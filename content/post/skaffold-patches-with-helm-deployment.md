---
date: "2022-10-10T12:00:00-00:00"
title: "Skaffold patches with helm deployment"
---

## Introduction

Skaffold is a tool that allows fast and easy deployments into a kubernetes cluster. It conveniently watches for file changes and redeploys a development setup whenever a change is detected. When a service needs a third party dependency (e.g. database or a proxy), skaffold can also deploy Helm charts into the cluster.

To differentiate between environments, skaffold provides profiles which can reconfigure the way a deployment behaves. A mighty feature for such manipulation is the `patches` keyword. Unfortunately the combination of skaffold, helm and patches is a dangerous fairway. In this blog post we will explore what can go wrong and how to avoid this.

## TL;DR

Always use the flat yaml form when using `SetValues` and `patches` on the helm deployer.

## Create a demo helm chart

Since we want to explore how skaffold patches behave when using helm deployments, we need a helm chart. Fortunately helm comes with it's own generator, so we can quickly create one with `helm create demo`. This generates a fully functional helm chart which comes with it's own prefilled `values.yaml`, a `_helpers.tpl` containing commonly used templates and a number of configurable kubernetes manifests.

## Write the skaffold file

To deploy the freshly generated helm chart, we need to create a module file telling skaffold to use the helm chart. In the same directory, where we created the helm chart, we will create a file called `skaffold.yaml`. The contents of this file are:

```yaml
apiVersion: skaffold/v2beta28
kind: Config
metadata:
  name: demo

deploy:
  helm:
    releases:
      - name: demo
        chartPath: ./demo
        setValues:
          resources:
            limits:
              cpu: 100m
              memory: 128Mi
        upgradeOnChange: true
```

For this demonstration, we configure the helm chart to limit the resource usage on the cpu and memory. The way it is written here is usually the form used when writing the `values.yaml`. Let's call this the *expanded* form.

To see that the helm values are applied correctly, we will render the skaffold module and grep for the keys.

```shell
$ skaffold render -m demo | grep -E "(resources|limits|cpu|memory)"

          resources:
            limits:
              cpu: 100m
              memory: 128Mi
```

## Add patches the intuitive way

Now that we saw the correct configuration of the helm chart with skaffold, we want to prepare the deployment for the production environment. For this we add another profile to the `skaffold.yaml`. Since we expect the production environment to have a lot more requests, we also increase the resource limits.

Under the profiles key, we add this profile:

```yaml
deploy:
  ...

profiles:
  - name: prod
      patches:
        - op: replace
          path: /deploy/helm/releases/0/setValues/resources/limits/cpu
          value: 1000m
        - op: replace
          path: /deploy/helm/releases/0/setValues/resources/limits/memory
          value: 1Gi
```

When we now render the skaffold module with the prod profile activated, the resource values should adapt to the respective values.

```shell
$ skaffold render -m demo -p prod | grep -E "(resources|limits|cpu|memory)"

parsing skaffold config: failed to apply profiles to config "demo" defined in file "/tmp/skaffold.yaml": applying profile "prod": invalid path: /deploy/helm/releases/0/setValues/resources/limits/cpu. There's an issue with one of the profiles defined in config "demo" in file "/tmp/skaffold.yaml"; refer to the documentation on how to author valid profiles: https://skaffold.dev/docs/environment/profiles/.
```

That's a bummer. Skaffold complains that the path, we want to replace does not exist.

## Add patches the correct way

The way skaffold [interprets](https://github.com/GoogleContainerTools/skaffold/issues/6908) the helm deployers `SetValues` key is through a `FlatMap`. This means that both keys and values are stored as flat strings. Knowing this, we update the prod profile with the correct paths.

```yaml
profiles:
  - name: local
    ...

  - name: prod
      patches:
        - op: replace
          path: /deploy/helm/releases/0/setValues/resources.limits.cpu
          value: 1000m
        - op: replace
          path: /deploy/helm/releases/0/setValues/resources.limits.memory
          value: 1Gi
```

```shell
$ skaffold render -m demo -p prod | grep -E "(resources|limits|cpu|memory)"

          resources:
            limits:
              cpu: 1000m
              memory: 1Gi
```

This time the values are applied correctly.

Let's update the helm deploers `SetValues` content to also use the flat form to

1. prevent confusion due to the usage of two different forms
2. work as a guidance for us to remember that we must use the flat form for patches

```yaml
deploy:
  ...
        setValues:
          resources.limits.cpu: 100m
          resources.limits.memory: 128Mi
```

## Fazit

We saw that skaffold requires the paths for patches that target helm values must be written in a flat form to work correctly. Since it can be confusing using the *expanded* form and the *flat* form of yaml at the same time, we also use the flat form for the helm value keys.

The whole `skaffold.yaml` looks like this in the end:

```yaml
apiVersion: skaffold/v2beta28
kind: Config
metadata:
  name: demo

deploy:
  helm:
    releases:
      - name: demo
        chartPath: ./demo
        setValues:
          resources.limits.cpu: 100m
          resources.limits.memory: 128Mi
        upgradeOnChange: true

profiles:
  - name: prod
    patches:
      - op: replace
        path: /deploy/helm/releases/0/setValues/resources.limits.cpu
        value: 1000m
      - op: replace
        path: /deploy/helm/releases/0/setValues/resources.limits.memory
        value: 1Gi
```
