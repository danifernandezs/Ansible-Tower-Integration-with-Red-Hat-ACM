# Ansible Tower integration with Red Hat Advanced Cluster Management for Kubernetes

Now you will go through Ansible Tower integration with Red Hat Advanced Cluster Management for Kubernetes. You will run individual AnsibleJob, integrate AnsibleJobs with policy violations, Cluster lifecycle automations and as last scenario with ArgoCD integration you will run AnsibleJobs as a prehook.

Yo will need an Ansible Tower Instance (3.8.4 at time).

You will only configure Red Hat Advanced Cluster Management for Kubernetes.

## Before You Begin

In this section you will create the basic integration between RHACM with Ansible Tower and ArgoCD.

The Ansible Tower integration is based on `Ansible Automation Platform Resource Operator`. Make sure to install the operator before you begin the next exercises.

### Installing the Ansible Automation Platform Resource Operator

#### TL;DR;
````bash
oc apply -f aap-operator/.

oc patch installplan "$(oc get installplan -n ansible-automation-platform -o 'jsonpath={..metadata.name}')" \
    -n ansible-automation-platform \
    --type merge \
    --patch '{"spec":{"approved":true}}'
````

Applied resources by the previous command.
- Ansible Automation Platform namespace
- Operator Subscription
- Operator Group (With manual approval policy)

### Openshift GitOps Operator and ACM integration

#### TL;DR;
````bash
oc apply -f gitops-operator/.
oc patch installplan "$(oc get installplan -n openshift-operators -o 'jsonpath={..metadata.name}')" \
    -n openshift-operators \
    --type merge \
    --patch '{"spec":{"approved":true}}'
oc apply -f argocd-resource/.
for i in $(oc get managedcluster --no-headers | awk '{ print $1 }');
do oc label managedcluster $i cluster.open-cluster-management.io/clusterset=all-clusters;
done;
````

Applied resources and operations:
- OpenShift GitOps Operator Subscription (With manual approval policy)
- ArgoCD Resource
- ManagedClusterSet
- ManagedClusterSetBinding
- GitOps Server
- Placement Rule
- Managed clusters patched

## Default integration

All the integration needs use an OpenShift secret, this secret contains the Ansible Tower API url and the token to be used.

This secret is needed in all namespaces that going to be execute a remote Ansible Job Template.

## Setting up Authentication

In order to allow RHACM to access Ansible Tower you must set up a Namespace scoped secret for RHACM to use. RHACM uses the secret to authenticate against the Ansible Tower instance. The secret contains the Ansible Tower URL and Access Token.

To create the secret, navigate to **Credentials** -> **Add credentials** -> **Red Hat Ansible Automation Platform** in the RHACM UI and fill the next fields

- Credentials name
- Namespace

Press **Next**.

At the next screen, specify the **Ansible Tower host** and **Ansible Tower token**.

Press **Next**. Review the information, and press on **Add**.

## Simple AnsibleJob

Since the Ansible Automation Platform Resource Operator is running and listening to all AnsibleJob resource, you can invoke it to execute an Ansible Tower Template at specified instance.

````bash
oc create ns individual-ansible-job
````
````bash
---
apiVersion: tower.ansible.com/v1alpha1
kind: AnsibleJob
metadata:
  generateName: ansible-notification-
  namespace: individual-ansible-job
spec:
  tower_auth_secret: tower
  job_template_name: Remote Notificator
  extra_vars:
    triggered_by: Aura
    notification: email
    to: danifernandezs@redhat.com
````
