---
id: schedulers-k8s-persistent-volume-claims
title: Kubernetes Persistent Volume Claims via CLI
sidebar_label: Kubernetes Persistent Volume Claims (CLI)
---
<!--
    Licensed to the Apache Software Foundation (ASF) under one
    or more contributor license agreements.  See the NOTICE file
    distributed with this work for additional information
    regarding copyright ownership.  The ASF licenses this file
    to you under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License.  You may obtain a copy of the License at
      http://www.apache.org/licenses/LICENSE-2.0
    Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied.  See the License for the
    specific language governing permissions and limitations
    under the License.
-->

> This document demonstrates how you can utilize both static and dynamically backed [Persistent Volume Claims](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/) in the `Executor` containers. You will need to enable Dynamic Provisioning in your Kubernetes cluster to proceed to use the dynamic provisioning functionality.

<br/>

It is possible to leverage Persistent Volumes with custom Pod Templates but the Volumes you add will be shared between all Pods in the topology.

The CLI commands allow you to configure a Persistent Volume Claim (dynamically or statically backed) which will be unique and isolated to each Pod and mounted in a single `Executor` when you submit your topology with a Claim name of `OnDemand`. Using any Claim name other than on `OnDemand` will permit you to configure a shared Persistent Volume without a custom Pod Template which will be specific to an individual Pod. The CLI commands override any configurations you may have present in the Pod Template, but Heron's configurations will take precedence over all others.

Some use cases include process checkpointing, caching of results for later use in the process, intermediate results which could prove useful in analysis (ETL/ELT to a data lake or warehouse), as a source of data enrichment, etc.

**Note:** Heron ***will*** remove any dynamically backed Persistent Volume Claims it creates when a topology is terminated. Please be aware that Heron uses the following `Labels` to locate the claims it has created:
```yaml
metadata:
  labels:
    topology: <topology-name>
    onDemand: true
```

<br>

> ***System Administrators:***
>
> * You may wish to disable the ability to configure dynamic Persistent Volume Claims specified on the CLI. To achieve this, you must pass the define option `-D heron.kubernetes.persistent.volume.claims.cli.disabled=true` to the Heron API Server on the command line during boot. This command has been added to the Kubernetes configuration files to deploy the Heron API Server and is set to `false` by default.
> * If you have a custom `Role`/`ClusterRole` for the Heron API Server you will need to ensure the `ServiceAccount` attached to the API server has the correct permissions to access the `Persistent Volume Claim`s:
>
>```yaml
>rules:
>- apiGroups: 
>  - ""
>  resources: 
>  - persistentvolumeclaims
>  verbs: 
>  - create
>  - delete
>  - get
>  - list
>  - deletecollection
>```

<br>

## Usage

To configure a Persistent Volume Claim you must use the `--config-property` option with the `heron.kubernetes.volumes.persistentVolumeClaim.` command prefix. Heron will not validate your Persistent Volume Claim configurations, so please validate them to ensure they are well-formed. All names must comply with the [*lowercase RFC-1123*](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/) standard.

The command pattern is as follows:
`heron.kubernetes.volumes.persistentVolumeClaim.[VOLUME NAME].[OPTION]=[VALUE]`

The currently supported CLI `options` are:

* `claimName`
* `storageClass`
* `sizeLimit`
* `accessModes`
* `volumeMode`
* `path`
* `subPath`

***Note:*** A `claimName` of `OnDemand` will create unique Volumes for each `Executor` as well as deploy a Persistent Volume Claim for each Volume. Any other Claim name will result in a shared Volume being created between all Pods in the topology.

***Note:*** The `accessModes` must be a comma separated list of values *without* any white space. Valid values can be found in the [Kubernetes documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes).

***Note:*** If a `storageClassName` is specified and there are no matching Persistent Volumes then [dynamic provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/) must be enabled. Kubernetes will attempt to locate a Persistent Volume that matches the `storageClassName` before it attempts to use dynamic provisioning. If a `storageClassName` is not specified there must be [Persistent Volumes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/) provisioned manually with the `storageClassName` of `standard`.

<br>

### Example

An example series of commands and the `YAML` entries they make in their respective configurations are as follows.

***Dynamic:***

```bash
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.claimName=OnDemand
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.storageClassName=storage-class-name-of-choice
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.accessModes=comma,separated,list
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.sizeLimit=555Gi
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.volumeMode=volume-mode-of-choice
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.path=/path/to/mount
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.subPath=/sub/path/to/mount
```

Generated `Persistent Volume Claim`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  labels:
    app: heron
    onDemand: "true"
    topology: <topology-name>
  name: volumenameofchoice-<topology-name>-[Ordinal]
spec:
  accessModes:
  - comma
  - separated
  - list
  resources:
    requests:
      storage: 555Gi
  storageClassName: storage-class-name-of-choice
  volumeMode: volume-mode-of-choice
```

Pod Spec entries for `Volume`:

```yaml
volumes:
  - name: volumenameofchoice
    persistentVolumeClaim:
      claimName: volumenameofchoice-<topology-name>-[Ordinal]
```

`Executor` container entries for `Volume Mounts`:

```yaml
volumeMounts:
  - mountPath: /path/to/mount
    subPath: /sub/path/to/mount
    name: volumenameofchoice
```

<br>

***Static:***

```bash
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.claimName=OnDemand
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.accessModes=comma,separated,list
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.sizeLimit=555Gi
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.volumeMode=volume-mode-of-choice
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.path=/path/to/mount
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.subPath=/sub/path/to/mount
```

Generated `Persistent Volume Claim`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  labels:
    app: heron
    onDemand: "true"
    topology: <topology-name>
  name: volumenameofchoice-<topology-name>-[Ordinal]
spec:
  accessModes:
  - comma
  - separated
  - list
  resources:
    requests:
      storage: 555Gi
  storageClassName: standard
  volumeMode: volume-mode-of-choice
```

Pod Spec entries for `Volume`:

```yaml
volumes:
  - name: volumenameofchoice
    persistentVolumeClaim:
      claimName: volumenameofchoice-<topology-name>-[Ordinal]
```

`Executor` container entries for `Volume Mounts`:

```yaml
volumeMounts:
  - mountPath: /path/to/mount
    subPath: /sub/path/to/mount
    name: volumenameofchoice
```

<br>

## Submitting

An example of sumbitting a topology using the *dynamic* example CLI commands above:

```bash
heron submit kubernetes \
  --service-url=http://localhost:8001/api/v1/namespaces/default/services/heron-apiserver:9000/proxy \
  ~/.heron/examples/heron-api-examples.jar \
  org.apache.heron.examples.api.AckingTopology acking \
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.claimName=OnDemand \
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.storageClassName=storage-class-name-of-choice \
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.accessModes=comma,separated,list \
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.sizeLimit=555Gi \
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.volumeMode=volume-mode-of-choice \
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.path=/path/to/mount \
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumenameofchoice.subPath=/sub/path/to/mount
```

## Required and Optional Configuration Items

The following table outlines CLI options which are either ***required*** ( &#x2705; ), ***optional*** ( &#x2754; ), or ***not available*** ( &#x274c; ) depending on if you are using dynamic/statically backed or shared `Volume`.

| Option | Dynamic | Static | Shared
|---|---|---|---|
| `VOLUME NAME` | &#x2705; | &#x2705; | &#x2705;
| `claimName` | `OnDemand` | `OnDemand` | A valid name
| `path` | &#x2705; | &#x2705; | &#x2705;
| `subPath` | &#x2754; | &#x2754; | &#x2754;
| `storageClassName` | &#x2705; | &#x274c; | &#x274c;
| `accessModes` | &#x2705; | &#x2705; | &#x274c;
| `sizeLimit` | &#x2754; | &#x2754; | &#x274c;
| `volumeMode` | &#x2754; | &#x2754; | &#x274c;

<br>

***Note:*** The `VOLUME NAME` will be extracted from the CLI command and a `claimName` is a always required.

<br>

## Configuration Items Created and Entries Made

The configuration items and entries in the tables below will made in their respective areas.

One `Persistent Volume Claim`, a `Volume`, and a `Volume Mount` will be created for each `volume name` which you specify. Each will be unique to a Pod within the topology.

| Name | Description | Policy |
|---|---|---|
| `VOLUME NAME` | The `name` of the `Volume`. | Entries made in the `Persistent Volume Claim`'s spec, the Pod Spec's `Volumes`, and the `executor` containers `volumeMounts`.
| `claimName` | A Claim name for the Persistent Volume. | If `OnDemand` is provided as the parameter then a unique Volume and Persistent Volume Claim will be created. Any other name will result in a shared Volume between all Pods in the topology with only a Volume and Volume Mount being added.
| `path` | The `mountPath` of the `Volume`. | Entries made in the `executor` containers `volumeMounts`.
| `subPath` | The `subPath` of the `Volume`. | Entries made in the `executor` containers `volumeMounts`.
| `storageClassName` | The identifier name used to reference the dynamic `StorageClass`. | Entries made in the `Persistent Volume Claim` and Pod Spec's `Volume`.
| `accessModes` | A comma separated list of access modes. | Entries made in the `Persistent Volume Claim`.
| `sizeLimit` | A resource request for storage space. | Entries made in the `Persistent Volume Claim`.
| `volumeMode` | Either `FileSystem` (default) or `Block` (raw block). [Read more](https://kubernetes.io/docs/concepts/storage/_print/#volume-mode). | Entries made in the `Persistent Volume Claim`.
| Labels | Two labels for `topology` and `onDemand` provisioning are added. | These labels are only added to dynamically backed `Persistent Volume Claim`'s created by Heron to support the removal of any claims created when a topology is terminated.