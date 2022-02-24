# Testing the Compliance Operator for OpenShift

- [Testing the Compliance Operator for OpenShift](#testing-the-compliance-operator-for-openshift)
  - [Introduction](#introduction)
  - [Requirements](#requirements)
  - [Installing the Compliance Operator](#installing-the-compliance-operator)
  - [Understanding Compliance Profiles](#understanding-compliance-profiles)
  - [Understanding Compliance Scan Settings](#understanding-compliance-scan-settings)
  - [Configuring and Running a Compliance Scan](#configuring-and-running-a-compliance-scan)
  - [Reviewing  the Results](#reviewing--the-results)
    - [Applying a remediation to your cluster](#applying-a-remediation-to-your-cluster)
  - [Performing an On Demand Rescan](#performing-an-on-demand-rescan)
  - [Retrieving the RAW ARF Results of the Compliance Scan](#retrieving-the-raw-arf-results-of-the-compliance-scan)
  - [References](#references)

## Introduction

The [Compliance Operator](https://docs.openshift.com/container-platform/4.8/security/compliance_operator/compliance-operator-installation.html) is used to scan an OpenShift cluster and check for compliance against published standards. The Compliance Operator assesses compliance of both the Kubernetes API resources of OpenShift Container Platform, as well as the nodes running the cluster. The Compliance Operator leverages [OpenSCAP](https://www.open-scap.org/), a NIST-certified tool, to conduct the compliance scans.

## Requirements

You will need a running OpenShift Cluster as well as at least 1Gb of ReadWriteOnce persistent storage available in order to use the Compliance Operator.

## Installing the Compliance Operator

Using the OpenShift Console:

1. Log into your cluster
2. Select Operators->Operator Hub
3. Search for Compliance Operator and Select it.
4. Select Install and leave all defaults, and click install.

From the command line:

1. Login to your cluster with the oc command
2. Apply the yaml located in the install directory
   `oc apply -f install/`
3. Verify that the install worked

```shell
$ oc get csv -n openshift-compliance
NAME                          DISPLAY                                    VERSION   REPLACES                 PHASE
compliance-operator.v0.1.35   Compliance Operator                        0.1.35                             Succeeded
rhacs-operator.v3.63.0        Advanced Cluster Security for Kubernetes   3.63.0    rhacs-operator.v3.62.0   Succeeded
$ oc get deploy -n openshift-compliance
NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
compliance-operator              1/1     1            1           9m45s
ocp4-openshift-compliance-pp     1/1     1            1           8m27s
rhcos4-openshift-compliance-pp   1/1     1            1           8m28s
```

## Understanding Compliance Profiles

The Compliance Operator supplies multiple profiles that can be used as a baseline for the scan. To get a list of all the profiles available run the following command:

```shell
$ oc get -n openshift-compliance profiles.compliance
NAME              AGE
ocp4-cis          12m
ocp4-cis-node     12m
ocp4-e8           12m
ocp4-moderate     12m
rhcos4-e8         12m
rhcos4-moderate   12m
```

To understand the details of any given profile you can run the following command:

```shell
$ oc get -n openshift-compliance -o yaml profiles.compliance ocp4-cis
apiVersion: compliance.openshift.io/v1alpha1
description: This profile defines a baseline that aligns to the Center for Internet Security® Red Hat OpenShift Container Platform 4 Benchmark™, V0.3, currently unreleased. This profile includes Center for Internet Security® Red Hat OpenShift Container Platform 4 CIS Benchmarks™ content. Note that this part of the profile is meant to run on the Platform that Red Hat OpenShift Container Platform 4 runs on top of. This profile is applicable to OpenShift versions 4.6 and greater.
id: xccdf_org.ssgproject.content_profile_cis
kind: Profile
metadata:
  annotations:
...
rules:
- ocp4-accounts-restrict-service-account-tokens
- ocp4-accounts-unique-service-account
- ocp4-api-server-admission-control-plugin-alwaysadmit
- ocp4-api-server-admission-control-plugin-alwayspullimages
```

To view the individual rules that make up a compliance profile, you can run:

```shell
$ oc get -n openshift-compliance -o yaml rules.compliance ocp4-accounts-restrict-service-account-tokens
apiVersion: compliance.openshift.io/v1alpha1
description: Service accounts tokens should not be mounted in pods except where the workload running in the pod explicitly needs to communicate with the API server. To ensure pods do not automatically mount tokens, set automountServiceAccountToken to false.
id: xccdf_org.ssgproject.content_rule_accounts_restrict_service_account_tokens
kind: Rule
metadata:
...
rationale: Mounting service account tokens inside pods can provide an avenue for privilege escalation attacks where an attacker is able to compromise a single pod in the cluster.
severity: medium
title: Restrict Automounting of Service Account Tokens
```

## Understanding Compliance Scan Settings

Before configuring the Compliance Scan to run, you will need to decide how you want the operator to handle remediation. There are two scan settings options:

```shell
$ oc get scansettings -n openshift-compliance
NAME                 AGE
default              36m
default-auto-apply   36m
```

The default scansettings will run the selected scans, but only report on conformance or non-conformance of each rule that is run. The alternative is to use the "default-auto-apply" settings which will scan your cluster and auto-remediate any rules that fail.

## Configuring and Running a Compliance Scan

Once you have decided on which [profiles](#understanding-compliance-profiles) you want to compare against, and the [scan settings](#understanding-compliance-scan-settings) you want to use, we will create a ScanSettingBinding which combines the scan settings with the profile or profiles you want to certify your cluster against.

As an example, we will create a compliance scan that leverages the ocp4-cis-node and ocp4-cis profiles, and runs the scan in a report only mode. Review the scanSettingBinding-co.yaml file within this repo to see how this is created. Once you have reviewed the yaml file, apply to your cluster with the following command:

```shell
$ oc create -f scanSettingBinding-co.yml -n openshift-compliance
scansettingbinding.compliance.openshift.io/cis-compliance created
```

Once your ScanSettingBinding has been applied, you can watch the progress of the scan with:

```
$ oc get compliancescan -n openshift-compliance
NAME                   PHASE       RESULT
ocp4-cis               LAUNCHING   NOT-AVAILABLE
ocp4-cis-node-master   LAUNCHING   NOT-AVAILABLE
ocp4-cis-node-worker   LAUNCHING   NOT-AVAILABLE
```

The compliance operator will take some time to run the first scan. Re-run the previous command until you get the following output indicating the scans are **DONE**:

```shell
$ oc get compliancescan -n openshift-compliance
NAME                   PHASE   RESULT
ocp4-cis               DONE    NON-COMPLIANT
ocp4-cis-node-master   DONE    NON-COMPLIANT
ocp4-cis-node-worker   DONE    NON-COMPLIANT
```

## Reviewing  the Results

Once the compliance scan has completed, you can review the results of the scan.

```
$ oc get ComplianceCheckResult -n openshift-compliance
NAME                                                                           STATUS           SEVERITY
ocp4-cis-accounts-restrict-service-account-tokens                              MANUAL           medium
ocp4-cis-accounts-unique-service-account                                       MANUAL           medium
ocp4-cis-api-server-admission-control-plugin-alwaysadmit                       PASS             medium
ocp4-cis-api-server-admission-control-plugin-alwayspullimages                  PASS             high
ocp4-cis-api-server-admission-control-plugin-namespacelifecycle                PASS             medium
ocp4-cis-api-server-api-priority-v1alpha1-flowschema-catch-all                 NOT-APPLICABLE   medium
ocp4-cis-api-server-encryption-provider-cipher                                 FAIL             medium
ocp4-cis-api-server-encryption-provider-config                                 FAIL             medium
```

> **NOTE** The output above is truncated to make the document easier to read. Your output will be longer.

To review the individual results run the command:

```shell
$ oc get ComplianceCheckResult/ocp4-cis-api-server-encryption-provider-cipher -n openshift-compliance -o yaml
apiVersion: compliance.openshift.io/v1alpha1
description: |-
  Configure the Encryption Provider Cipher
  aescbc is currently the strongest encryption provider, it should
  be preferred over other providers.
id: xccdf_org.ssgproject.content_rule_api_server_encryption_provider_cipher
instructions: |-
  Run the following command:
  $ oc get apiserver cluster -ojson | jq -r '.spec.encryption.type'
  The output should return aescdc as the encryption type.
kind: ComplianceCheckResult
```

We can also review the ComplianceRemediation object for this check with the following command:

```shell
$ oc get ComplianceRemediation/ocp4-cis-api-server-encryption-provider-cipher -n openshift-compliance -o yaml
apiVersion: compliance.openshift.io/v1alpha1
kind: ComplianceRemediation
metadata:
  name: ocp4-cis-api-server-encryption-provider-cipher
  namespace: openshift-compliance
spec:
  apply: false
  current:
    object:
      apiVersion: config.openshift.io/v1
      kind: APIServer
      metadata:
        name: cluster
      spec:
        encryption:
          type: aescbc  
```

The remediation payload is stored in the _spec.current_ attribute of the output. Not all ComplianceCheckResult objects create ComplianceRemediation objects. Only ComplianceCheckResult objects that can be remediated automatically do.

### Applying a remediation to your cluster

If you did not configure your ScanSettingBinding to automatically apply the remediations, you can manually select and apply individual remediations. Once you have identified the remediation you wish to apply, update the _spec.apply_ attribute to "true". For example to automatically apply the ocp4-cis-api-server-encryption-provider-cipher remediation run the following command:

> **WARNING** Applying remediations should only be done after careful consideration and understanding of the changes that will be made, and how they may effect the cluster you are running on.

```
# WARNING - This example change will enable encryption and will disrupt the ability to log into your cluster while the change is applied.
# ONLY APPLY if you understand the above
$ oc -n openshift-compliance patch complianceremediations/ocp4-cis-api-server-encryption-provider-cipher --patch '{"spec":{"apply":true}}' --type=merge
```

## Performing an On Demand Rescan

Based on the [scansetting](#understanding-compliance-scan-settings) you choose, the Compliance Scan will run at the next scheduled time. If however you want to run a scan on demand, you can annotate the scan in order to indicate that a rescan is required.

```shell
$ oc annotate compliancescans/ocp4-cis compliance.openshift.io/rescan=
$ oc get compliancescan -n openshift-compliance
NAME                   PHASE       RESULT
ocp4-cis               LAUNCHING   NOT-AVAILABLE
ocp4-cis-node-master   DONE        NON-COMPLIANT
ocp4-cis-node-worker   DONE        NON-COMPLIANT
```

## Retrieving the RAW ARF Results of the Compliance Scan

In some cases you may need to share the raw results with an auditing organization. The results of the compliance operator are stored in a ReadWriteOnce persistent volume. In order to retrieve these results we will need to spawn a pod that will allow us to copy out the data.

> **NOTE**: Because the persistent volume is ReadWriteOnce, you must ensure that after you have retrieved the files you destroy the pod, or else the nightly scans will be unable to run.

```shell
$ oc get pvc -n openshift-compliance
NAME                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
ocp4-cis               Bound    pvc-f7b76e13-e7f8-4d1b-b565-7b65acb822d0   1Gi        RWO            managed-nfs-storage   22m
ocp4-cis-node-master   Bound    pvc-3fa5aff0-d793-4892-96f8-93e3e2714c6c   1Gi        RWO            managed-nfs-storage   22m
ocp4-cis-node-worker   Bound    pvc-ce4af0c3-c9b6-428e-81f4-090dc244a8c6   1Gi        RWO            managed-nfs-storage   23m
```

In the above output you can see that we have three volumes with results that we will need to retrieve. We will use the `retrieveResults-template.yml` file to create a pod that can be used to pull a copy of the results locally. Start by creating a `retrieveResults-cis.yml` file and then update the persistent volume to point to `ocp4-cis` persistent volume claim from the last command:

```shell
$ cp retrieveResults-template.yml retrieveResults-cis.yml
$ vi retrieveResults-cis.yml
# update the "claimName: <scan results pvc>" section to be '
# claimName: ocp4-cis
$ oc create -f retrieveResults-cis.yml -n openshift-compliance
$ oc -n openshift-compliance cp pv-extract:/scan-results .
$ oc delete pod/pv-extract -n openshift-compliance
```

If you take a look in your current directory, you should now see a folder structure that contains subdirectories "0,1,2". These are the three most recent scans that were completed. Inside each directory will be a file such as `ocp4-cis-api-checks-pod.xml.bzip2` which contains the results of the scan in Asset Reporting Format (ARF).

## References

[OpenSCAP Home Page](https://www.open-scap.org/)
[OpenShift Compliance Operator Install](https://docs.openshift.com/container-platform/4.7/security/compliance_operator/compliance-operator-installation.html)