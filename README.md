# Key-Value Adjust Driver
## Overview

This driver represents our effort to give access to the developers to implement the process of querying and adjusting settings in their target application without writing a fully-fledged adjust driver on Python (which is our main approach).

The main idea of this driver is to give the developer an ability to implement two separate executables representing `querying` and `adjusting` processes.
Querying process means extracting current values of target settings from the application that is being optimized.
Adjusting process means applying given settings and their respective values to the application along with optional reporting of the progress.

You have two write two executables that must contain shebang line:
`adjust` and `query`. These are the names of the files we are going to execute at two different stages. In the section below you can find the expected stdin and stdout on each of those stages. They meant to be self-explanatory.

A set of components and their respective settings you want to optimize would have to be defined as the configuration file which we expect to be located in the current working directory with name `config.yaml`.

### Query stage
#### stdin
Your query executable would be given input in the form of a YAML structure. It would be an object where the root level keys represent `components` that reflect target applications (ex. canary-pg, canary-solr). Each `component` would be an array containing names of settings we would like to query for the current value.
#### stdout
Your query executable expected to retrieve current setting values in the form of a YAML structure where the key represents a setting name prefixed with a component name it is coming from joined with the dot (component.setting: value). And the value represents whatever you find current for the setting you would be querying.

### Adjust stage
#### stdin
At the adjust stage your executable would be given input in the form of a YAML structure. It would be an object where the root level keys represent `components`, which reflect target applications. Each `component` would be an object representing a set of settings to be adjusted. Each key in that object represents a setting name. And the value of each of those keys represents that particular setting value we would like to set.

Example:
```yaml
database:
   settings:
       memory: 2
       cpu: 3
       replicas: 2
```

#### stdout
(optional) You can report the progress of the adjustment process. You can do so by sending a line to stdout and flushing it immediately. We expect each line we read from stdout to represent a YAML object which has at least the key `progress` with the value being an integer. We use this value to reflect the progress of the adjustment process in Optune's web interface.

Example:
```yaml
progress: 1  # can go from 0 all the way to 100
```

#### stderr
Whenever there's an error you want to report or some other valuable information, such as a trace, debug info or an exception stack trace, you can dump all the contents into stderr. We will send that information to our backend server for debugging purposes. 

## Sample `config.yaml` along with notes
```yaml
kv:
  query: query  # path to the executable
  adjust: adjust  # path to the executable
  application:
    components:
        comp1:
          settings:
            memory:
              # Currently, we support two types of settings: range and enum.
              # In the case of enum, we would traverse all the possible values. Type enum does not support properties min, max and step.
              type: range
              # Min, max and step represent the boundaries of the setting. We are not going to go beyond those. Property step represents an increment of change. Note that the step value must be able to divide the difference between min and max values without a remainder.
              min: 1
              max: 5
              step: .1
              unit: GiB
            cpu:
              type: range
              min: .1
              max: 4
              step: .1
              unit: cores
            replicas:
              type: range
              min: 1
              max: 10
              step: 1
              unit: count
```
