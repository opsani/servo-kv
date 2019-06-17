# Optune Key-Value Adjust Driver
## Overview

This repo contains a so-called "adjust driver". The main responsibility of `adjust driver` is to apply settings provided by `Optune AI` backend to the target application. This adjust driver is split into two parts:

1. Query - this part is responsible for acquiring current application settings, so that we can define the baseline for the settings of the target application.
2. Adjust - this part is responsible for applying the setting values that were provided by the Optune AI backend.

This particular adjust driver is built with one goal in mind - to be as simple as possible. In essence, this driver expects the user to provide two executable files that are responsible for the two respective parts of the process: querying and adjusting. In the respective sections below you will find the requirements for those executables. 

## Driver configuration

This driver expects the user to provide configuration file `config.yaml` in the same folder as the driver `adjust` Python file.

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
* `query_exec` is the shell command that is going to be called on the query request. It is executed under the same user as the `adjust` executable itself.
* `adjust_exec` is the shell command that is going to be called on the adjust request. It is executed under the same user as the `adjust` executable itself.
* `components` defines a set of target components that are going to be optimized. It is certain there will be components that would only report their current settings, and not adjust them in any way. Those components would only participate in the calculation of the score of improvement.
* `settings` defines a set of target component settings that are going to be optimized. Properties `min`, `max` and `step` correspond to boundaries of allowed change. Property `step` defines the size of a setting change and should allow the difference between the properties `min` and `max` to be divided by its value without a remainder, so that we can reach the upper boundary in equal `step` increments.

## Query executable

Briefly, the query executable is responsible for reaching out to the target component (which can be an application, a deployment, a container, etc) to get the current setting values. It would let us define the baseline for the optimization.

In the query executable the user is expected to request current values of settings for components provided in the `config.yaml`. A list of those setting names and its components should be hard-coded in the executable.

User has to output current setting values to `stdout` in the following form:
```yaml
opsani-canary.memory: 3
opsani-canary.cpu: 2
opsani-canary.replicas: 1
```

The first part (before the dot) signifies the component name, and a part after the dot corresponds to a particular setting name and its current value. We expect no progress to be reported while performing querying.

## Adjust executable

Briefly, the adjust executable is responsible for adjusting particular component settings to the values provided by the `Optune AI` backend.

In the adjust executable the user is expected to adjust the target component settings to the ones provided in `stdin`. The input values are provided in the form of a YAML object like in the following example.
```yaml
opsani-canary.memory: 2.5
opsani-canary.cpu: 1.8
opsani-canary.replicas: 1
```

The input reflects the same format that is expected on the output of `query` executable which you can refer to in section `Query executable` above.

In case the adjustment is not immediate and takes some time to complete, you can report the progress by writing to `stdout` a line with a whole number ranging from 1 to 100, which represents the percentage. It is important to flush the output on every progress report.

If you stumbled upon an exception or any other important tracing/debugging information, you can report its sanitized version to the backend by writing it to `stderr`. All or part of the contents of `stderr` will be sent to the backend for debugging purposes.
