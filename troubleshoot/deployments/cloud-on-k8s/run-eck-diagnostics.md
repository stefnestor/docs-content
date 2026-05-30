---
navigation_title: Diagnostics
mapped_pages:
  - https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-take-eck-dump.html
applies_to:
  deployment:
    eck: all
products:
  - id: cloud-kubernetes
---

# Capture ECK diagnostics [k8s-take-eck-dump]

{{eck}} (ECK) diagnostics is a stand-alone command line tool that you can run on hosts where the ECK operator is installed. It emits ECK and Kubernetes data, and stores everything into an archive file that can be provided to [Elastic support](/troubleshoot/index.md#troubleshoot-work-with-support) for troubleshooting and investigation purposes.

::::{note}
:::{include} /troubleshoot/_snippets/diagnostics-privacy.md
:::
::::

## Prepare [k8s_prepare] 

To install the [eck-diagnostics](https://github.com/elastic/eck-diagnostics/) tool:

1. Locate the [latest release](https://github.com/elastic/eck-diagnostics/releases/latest).
2. Download the `*.tar.gz` asset associated to your hardware profile.
3. Unpack the gzip'ed tar archive to access the diagnostic binary `eck-diagnostics`.

## Run [k8s_run] 

You must know the Kubenetes namespaces of your operator and stack resources to run this diagnostic. The most common usage of this `eck-diagnostics` binary is:

```bash
eck-diagnostics -o <operator-namespaces> -r <resources-namespaces>
```

This tool supports various command line flags. The most common command line flags considered are:

* (Optional) Run it with `-h` or `--help` to print all available options. 

    ```bash
    eck-diagnostics --help
    ```

* (Required) Run it with `-r` or `--resources-namespace` to indicate the namespaces where your Elastic stack resources are deployed. 

* (Optional) Run it with `-o` or `--operator-namespaces` to override the default `elastic-system` namespace for the deployed ECK operator.

* (Optional) Run it with `--stack-diagnostics-timeout` to designate the maximum waiting time to pull the {{es}} and {{kib}} diagnostics. Defaults to `5m0s`.

* (Optional) Run it with `--run-stack-diagnostics=false` to disable automatically pulling [{{es}} diagnostics](/troubleshoot/elasticsearch/diagnostic.md) and [{{kib}} diagnostics](/troubleshoot/kibana/capturing-diagnostics.md) hosted within the resources namespace. This is enabled by default and is recommended. However, this requires the temporary deployment of additional Pods into the Kubernetes cluster. 

* (Optional) Check [ECK Diagnostics in air-gapped environments](/deploy-manage/deploy/cloud-on-k8s/air-gapped-install.md#k8s-eck-diag-air-gapped) for command line flags to run support diagnostics in environments without access to the open internet.

* (Optional) The tool can also filter the Elastic resources that it runs diagnostics against by specifying the `-f` or `--filters` flag. By specifying the type and name of resource, you can filter for any combination of Elastic stack components.

    ```bash
    # Filter only for an elasticsearch cluster named 'mycluster', and a kibana instance named 'mykibana'.
    eck-diagnostics -o <operator-namespaces> -r <resources-namespaces> -f "elasticsearch=mycluster" -f "kibana=mykibana"
    ```

## Example [k8s_example] 

Assuming the ECK operator is deployed in a namespace called `operators` and Elastic stack resources are deployed in the `security` and `monitoring` namespaces, you should run `eck-diagnostics` as follows:

```bash
eck-diagnostics -o=operators -r=security,monitoring
```

Sample output:

```bash
2026/01/01 20:34:20 ECK diagnostics with parameters: {DiagnosticImage:docker.elastic.co/eck-dev/support-diagnostics:9.3.1 ECKVersion: Kubeconfig: OperatorNamespaces:[operators] ResourcesNamespaces:[security monitoring] OutputDir:/tmp RunStackDiagnostics:true Verbose:false}
2026/01/01 20:34:22 Extracting Kubernetes diagnostics from operators
2026/01/01 20:34:23 ECK version is 3.1.0
2026/01/01 20:34:23 Extracting Kubernetes diagnostics from security
2026/01/01 20:34:23 Extracting Kubernetes diagnostics from monitoring
2026/01/01 20:34:24 ECK diagnostics written to /tmp/eck-diagnostic-2026-01-01T20-34-21.zip
```
