# Optune Key-Value Adjust Driver
## Overview

This driver calls two user-provided executables: (a) an executable to query current settings of a target application, and (b) an executable to adjust settings of a target application.

## Adjust executable

In the adjust executable the user is given a serialized YAML object on `stdin` in the following format:
```yaml
component1.memory: 2
component1.cpu: 3
component1.replicas: 2
```

Where `component` stands for the name of a target application, and section `settings` is a `key: primitive-value` object that defines a set of settings with their respective requested values.

The user has to adjust the application's settings to the values given on `stdin`.

If the adjustment process takes more than 30 seconds, you can report the progress of the adjustment process by writing to `stdout` a number ranging from 1 to 100, that represents percentage of completeness of the process.

If there's an error or any other tracing/debugging information that is important, you can write it to `stderr`. All the contents of `stderr` will be retained and then either truncated according to the truncation settings of the driver and or kept in a full form and sent to our backend server. Make sure to sanitize the output from sensitive information.


## Query executable

In the query executable the user is given a serialized YAML object on `stdin` in the following format:
```yaml
components:
    component1:
        - memory
        - cpu
        - replicas
    component2:
        - setting1
        - setting2
```

The user has to query respective applications to get the requested settings' current values.

When all the requested settings' values have been acquired – they have to be written to `stdout` in the following form:
```yaml
component1.setting1: value
component2.setting2: value
```

## Sample `config.yaml`

```yaml
kv:
  components:
    component1:
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
