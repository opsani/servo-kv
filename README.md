# Optune Key–Value Adjust Driver
## Overview

This repo contains a so-called "adjust driver". The main responsibility of the `adjust driver` is to apply settings provided by the `Optune AI` backend to the target applications. This particular adjust driver is split into two parts:

1. `Query` — this part is responsible for acquiring current application settings, so that we can define the baseline for the settings of the target application.
2. `Adjust` — this part is responsible for applying the setting values that were provided by the `Optune AI` backend.

The driver is built with one goal in mind — to be as simple as possible. In essence, it expects the user to provide two executable files that are responsible for the two respective parts of the process: querying and adjusting. In the following sections you can find the requirements for those executables.

## Driver configuration

This driver expects the user to provide a configuration file named `config.yaml` in the same folder as the driver itself. You can follow the example and its description below to create the config file for your own setup.

### Example `config.yaml`
```yaml
kv:
  query_exec: sh query.sh --abc
  adjust_exec: sh adjust.sh
  components:
    canary:
      settings:
        memory:
          type: range
          min: 1
          max: 5
          step: .1
          unit: GiB
        cpu:
          type: range
          min: .1
          max: 4
          step: .1
          unit: Cores
        replicas:
          type: range
          min: 1
          max: 10
          step: 1
          unit: Count
```

In detail:
* `query_exec` is the shell command that is going to be called on the query request from the backend. It is executed under the same user as the `adjust` executable itself.
* `adjust_exec` is the shell command that is going to be called on the adjust request from the backend. It is executed under the same user as the `adjust` executable itself.
* `components` defines a set of target components that are going to be optimized. Component is an abstract word that implies any type of entity, such as deployment, application, ingress controller or anything else. Usually it is a deployment or an application. Note, that it is certain we will have components that would only report their current settings, and not adjust them in any way. Those components would only participate in the calculation of the score of improvement. Usually, those would represent `production` setup.
* `settings` defines a set of target component settings that are going to be optimized. Properties `min`, `max` and `step` correspond to the boundaries of allowed change. Property `step` defines the size of a setting value increment and its value should allow us to go from value of `min` all the way to the value of `max` in the whole steps, i.e. without fractioning.

## Query executable

Briefly, the query executable is responsible for reaching out to the target component (which can be an application, a deployment, a container, etc) to get the current setting values. It would let us define the baseline for the optimization.

In the query executable the user is expected to request current setting values for the components provided in `config.yaml`. A list of those settings and their components should be hard-coded in the query executable.

User has to output current setting values to `stdout` in the following form:
```yaml
opsani-canary.memory: 3
opsani-canary.cpu: 2
opsani-canary.replicas: 1
```

The first part (before the dot) signifies a component name, and a part after the dot corresponds to a particular setting name and its current value. We expect no progress to be reported while performing querying.

## Adjust executable

Briefly, the adjust executable is responsible for adjusting particular component settings to the values provided by the `Optune AI` backend.

In the `adjust` executable the user is expected to adjust the target component settings to the values provided in `stdin`. The input values are sent in the form of a YAML object. See example below.

### Example input in `stdin`

```yaml
opsani-canary.memory: 2.5
opsani-canary.cpu: 1.8
opsani-canary.replicas: 1
```

### Progress reporting

The input reflects the same format that is expected on the output of a `query` executable which you can refer to in the section [`Query executable`](#query-executable) above.

In case the adjustment is not immediate and takes some time to complete, you can report the progress by writing to `stdout` a line with a whole number ranging from 1 to 100, which represents percentage. It is important to flush the output on every progress report.

### Error reporting

If you stumbled upon an exception or any other important tracing/debugging information, you can report its sanitized version to the backend by writing it to `stderr`. All or part of the contents of `stderr` will be sent to the backend for debugging purposes. You can provide an environment variable that will define in what form the log is going to be discharged to the backend. For that, please, refer to https://github.com/opsani/servo#environment-variables.
