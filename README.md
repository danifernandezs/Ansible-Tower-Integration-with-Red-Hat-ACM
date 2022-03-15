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

## Ansible Tower Governance Integration

In this example you will create a policy that monitors whether a _forbidden namespace_ exists at managed clusters.

If the namespace exists a violation will be initiated. Once the violation is initiated an Ansible Job Template will be triggered.

````bash
oc create ns rhacm-policies
````

### Configuring the Policy

The next Policy will initiate an alert if a namespace with the name `forbidden-namespace` is present in the cluster. Apply the policy to the hub cluster -

```
oc apply -f rhacm-policies/policy.yaml
```

After creating the policy, make sure that the policy works as expected. Create a namespace with the name `forbidden-namespace`, at a desired managed cluster.

````bash
oc create namespace forbidden-namespace
````

### Configuring PolicyAutomation

Now that a policy is configured, create a PolicyAutomation object that will initiate an Ansible Job that will fired.

````bash
oc apply -f rhacm-policies/policyautomation.yaml
````
![](rhacm-policies/img/PolicyAutomation01.png)
![](rhacm-policies/img/PolicyAutomation02.png)


## RHACM Automation Template

You can create prehook and posthook AnsibleJob instances that occur before or after creating or upgrading your clusters. 

Configuring prehook and posthook jobs for cluster destroy, and cluster scale actions are not supported.

You must specify the Ansible job template that you want to use for a cluster when you create the cluster. To specify the template when creating a cluster, select the Ansible template that you want to apply to the cluster in the Automation step. If there are no Ansible templates, click Add automation template to create one.

**Infrastructure** -> **Automation** 

![](rhacm-automation/img/AutomationTemplate01.png)
![](rhacm-automation/img/AutomationTemplate02.png)
![](rhacm-automation/img/AutomationTemplate03.png)

A ClusterCurator resource as example:

````bash
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ClusterCurator
metadata:
  name: cloud-automation-template
  namespace: hive
spec:
  destroy:
    posthook: []
    prehook: []
    towerAuthSecret: hive-tower
  install:
    posthook:
      - extra_vars: {}
        name: Clusters Interconnections
      - extra_vars:
          via: Slack
          hook: https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
        name: Remote Notificator
    prehook:
      - extra_vars: {}
        name: Open Network Firewalls
      - extra_vars:
          type: NLB
        name: LoadBalancers provisioning
      - extra_vars: {}
        name: Remote Notificator
    towerAuthSecret: hive-tower
  scale:
    posthook: []
    prehook: []
    towerAuthSecret: hive-tower
  upgrade:
    posthook:
      - extra_vars:
          via: Slack
          hook: https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
        name: Remote Notificator
    prehook:
      - extra_vars: {}
        name: Open Network Firewalls
      - extra_vars: {}
        name: Remote Notificator
    towerAuthSecret: hive-tower
````

