# k8s-flux

This repository contains code used to define the [Weaveworks Flux](https://github.com/weaveworks/flux) set up for the one of my sample clusters. Flux is used to sync Kubernetes YAML stored in Git with a remote cluster.

Please note that this repo is only using YAMLs that can be copied across to your projects. I am not a big fan of Helm as it usually takes more time than necessary to deploy resources. Hopefully, you will find it easy and quick to follow my example which I am using with multiple kubernetes clusters. 

## Layout, YAMLs and Flux Configuration

All of the relevant YAML in this repository are in the `system/` directory. In these directories are the YAMLs to spin up the _system_ Flux in the `kube-system` namespace. Namespaces definitions in `system/namespaces/`. And number of services that can be found useful to deploy through flux.

### _System_ Flux

This Flux has a **cluster admin** role so is extremly powerful! The _system_ flux deploys code in the `system/` directory and its child directories for its cluster. Note that this includes the YAML for iteslf! Changes at this level in the directory can break your Flux deployment. The system YAMLs at this level are very similar to the default YAMLs in [Flux](https://github.com/weaveworks/flux).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flux
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: flux
        args:
        - --git-url=git@github.com:marcincuber/k8s-flux.git
        - --git-branch=master
        - --git-path=system
        - --git-poll-interval=120s
        - --registry-poll-interval=120s
        - --k8s-namespace-whitelist=kube-system
```
* `--git-url` is the URL Flux watches for changes. For the _system_ Flux that is this repository
* `--git-branch` is the desired branch of the repository to watch
* `--git-poll` sets the poll interval for Flux to check for changes to git
* `--git-path` is the subdirectory in the target repo that Flux deploys. For the `system` flux that is `system/`
* `--k8s-namespace-whitelist` is the namespace in which Flux watches for deployed containers and poll the remote registry to discover newer versions.

More flags and further docs can be found -> [Flux](https://github.com/weaveworks/flux).

### Resources deployed through Flux

In this example you can find the following Kuberentes resources to be controlled/deployed by flux;

*	Namespaces
*	Cluster Roles
*	Role Bindings and/or Cluster Role Bindings
*	Service Accounts
*	External DNS
*	Metrics Server
*	Sealed-Secrets [home of sealed-secrets](https://github.com/bitnami-labs/sealed-secrets)
*	NewRelic Monitoring
*	Kube State Metrics required for NewRelic Monitoring
*	Flux instance/s for applications

## External DNS

External DNS Kubernetes YAML is stored in `system/external-dns/`. Documentation and additional information about [External DNS](https://github.com/kubernetes-incubator/external-dns).

In my cluster I make use of `kube2iam` which is being used in External DNS deployment template to perform required actions. Please ensure that you replace `ENTER_YOUR_IAM_ROLE_ARN_HERE` in `system/external-dns/external-dns.yaml` with the ARN of the role that exists in your AWS account.

### Required IAM Permissions for IAM ROLE- policy

```json
{
 "Version": "2012-10-17",
 "Statement": [
   {
     "Effect": "Allow",
     "Action": [
       "route53:ChangeResourceRecordSets"
     ],
     "Resource": [
       "arn:aws:route53:::hostedzone/*"
     ]
   },
   {
     "Effect": "Allow",
     "Action": [
       "route53:ListHostedZones",
       "route53:ListResourceRecordSets"
     ],
     "Resource": [
       "*"
     ]
   }
 ]
}
```

In order to configure the external-dns to your taste you can read -> [external-dns configuration options](https://github.com/kubernetes-incubator/external-dns/blob/master/docs/tutorials/aws.md).

## Metrics Server

### User guide

You can find the user guide in
[the official Kubernetes documentation](https://kubernetes.io/docs/tasks/debug-application-cluster/core-metrics-pipeline/).

### Design

The detailed design of the project can be found in the following docs:

- [Metrics API](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/resource-metrics-api.md)
- [Metrics Server](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/metrics-server.md)

For the broader view of monitoring in Kubernetes take a look into
[Monitoring architecture](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/monitoring_architecture.md)

Note that this is just a sample Metrics Server, nothing has been customised. 

##  Sealed Secrets

We use [sealed-secrets](https://github.com/bitnami-labs/sealed-secrets) which allows us to store Encrypted Secret in the Github repository. The SealedSecret can be decrypted only by the controller running in the target cluster and nobody else (not even the original author) is able to obtain the original Secret from the SealedSecret.

### Details

The key certificate (public key portion) is used for sealing secrets, and needs to be available wherever `kubeseal` is going to be used. The certificate is not secret information, although you need to ensure you are using the correct file. You can install kubeseal using `brew install kubeseal` or from source [sealed-secrets](https://github.com/bitnami-labs/sealed-secrets).

`kubeseal` will fetch the certificate from the controller at runtime (requires secure access to the Kubernetes API server), which is convenient for interactive use. The recommended automation workflow is to store the certificate to local disk with `kubeseal --fetch-cert > mycert.crt`, and use it offline with `kubeseal --cert mycert.crt`. Once sealed-secrets has been deployed to the cluster, you should ensure that you the master key has been backed up.

```bash
# Usage example:
# Create a json/yaml-encoded Secret somehow:
# (note use of `--dry-run` - this is just a local file!)
kubectl create secret generic mysecret --dry-run --from-literal=foo=bar -o json > mysecret.json
# or 
kubectl create secret generic mysecret --dry-run --from-literal=foo=bar -o yaml > mysecret.yaml
# encrypt secret (json)
kubeseal < mysecret.json > mysealedsecret.json 
# encrypt secret (yaml)
kubeseal --format yaml < mysecret.json > mysealedsecret.yaml
# or
kubeseal --format yaml < mysecret.yaml > mysealedsecret.yaml
# Output file mysealedsecret.json|yaml is safe to commit
```

Please note for the above commands it is sufficient to use the `.crt` which is the public part of the secret. You need to store safely your secret containing both private and public parts which can be retrived using the following command;

```
kubectl get secret -n kube-system sealed-secrets-key -o yaml > master.pem
```

## NewRelic Monitoring

To activate the Kubernetes integration, we deploy the newrelic-infra agent onto a Kubernetes cluster as a daemon set and it requires kube-state-metrics. Yaml files can be found in `newrelic-monitoring/` folder. They are deployed using Flux but you need to take care of specifying CLUSTER_NAME and NEWRELIC_KEY, see below what is required.

To find out how to install NewRelic Monitoring for you cluster manually or for any other documentation see -> [NewRelic K8s Integration docs](https://docs.newrelic.com/docs/integrations/kubernetes-integration/installation/kubernetes-installation-configuration).

###	Configuration

In case you decide to use NewRelic and the templates provided here, make sure to modify the deployment file `system/newrelic-monitoring/newrelic-k8s/newrelic-k8s.yaml`. The following block should be modified;

```
env:
	- name: "CLUSTER_NAME"
	  value: "PUT_YOUR_OWN_CLUSTER_NAME_TO_ADD"
	- name: "NRIA_LICENSE_KEY"
	  valueFrom:
	    secretKeyRef:
	      name: newrelic-config
	      key: newrelic_key
```

Note that in my example, I create and pass `NRIA_LICENSE_KEY` as a secret. If you prefer to put your newrelic-key directly use;

```
env:
  - name: NRIA_LICENSE_KEY
    value: YOUR_LICENSE_KEY
```

###	Sealed Newrelic Token

To handle `newrelic-key as a secret`, I am utilising the `sealed-secrets` resource. Assuming you have your secret locally, you can encrypt it.

```
newrelic-token.yaml # Locally stored secret yaml file containing NewRelic key which is base64 encoded
# Fetch your sealed-secrets .crt (public part)
kubeseal --fetch-cert > cert.crt
# Encrypt your secret using the .crt
kubeseal --cert cert.crt --format yaml < newrelic-token.yaml > newrelic-token-sealed.yaml
``` 

Once you generated your encrypted file ensure that you only commit the encrypted file `newrelic-token-sealed.yaml`. You shouldn't commit `newrelic-token.yaml`.

The encrypted file `newrelic-token-sealed.yaml` is placed in `encrypted-secrets/`. Make sure you update newrelic-token-sealed.yaml with your own encrypted base64 encoded newrelic token, otherwise your newrelic monitoring won't have a valid token to run with.

## Flux instance for application/s

If you are planning to deploy your application using Flux to custom namespace/s. You can create another set of templates for Flux app which will only be allow to performed actions on application's namespace. Such example can be found in `projects/test-sub-project/sample-dev-flux-project/`.

### _Application_ Fluxes

The _application_ Fluxes reside in their own namespaces in the cluster. These namespaces are simply `<target-namespace>-flux` so for `sample-dev-project` the Flux namespace is `sample-dev-flux`. This namespace just contains Flux, memcached and flux K8s service account.

Each _application_ Flux has namespace admin over its own namespace and the target namespace. This means it can manage all namespaced resources in those namespaces. This permissions boundary is enforced through K8s RBAC.

The custom arguments for the _application_ Fluxes are as follows;

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flux
  namespace: sample-dev-flux
spec:
  template:
    spec:
      serviceAccount: test-sample-dev-flux
      containers:
      - name: flux
        args:
        - --git-url=git@github.com:marcincuber/test-app-flux-repo.git
        - --git-branch=master
        - --git-poll-interval=1m0s
        - --git-path=dev-app
        - --k8s-namespace-whitelist=sample-dev
```

The only signifcant differences between this config and the _system_ Flux config are that the `--git-url` is now pointing at the application flux repository, `--git-path` corresponds do a relevant directory in that repository and the `--k8s-namespace-whitelist` is now the target namespace for the _application_ Flux. Also in your deployment spec you specify service account `test-sample-dev-flux`.

Please make sure you modify application so that it points at your application's repository. I only used fake values in my `test-sub-project` deployment flux yaml.

## Disaster Recovery

In the event that a cluster is completely lost or needs to be spun up from scratch, the applications can be easily be restored with Flux:

1. Deploy the _system_ Flux manually using `kubectl apply -f system/`.
1. Get the new _system_ Flux SSH key from the new FLux with `kubectl logs -l name=flux -n kube-system | head` or `fluxctl --k8s-fwd-ns=kube-system identity`.
1. Add the _system_ Flux SSH key (public part) to the _system_ GitHub service account or deploy keys in your repo settings.
1. Your sealed-secrets application (custom resource) will be redeployed as part of flux. However, you will need to make sure that you put the previously generated key. Above named as `master.pem`.
1. Put the secret back before starting the controller - or if the controller was already started, replace the newly-created secret and restart the controller (sealed-secrets).
1. Simply run `kubectl replace -f master.pem` and then `kubectl delete pod -n kube-system -l name=sealed-secrets-controller`.
1. Wait for the application Fluxes to be deployed.
1. Get the new _application_ Flux public SSH key from the pod logs with `kubectl logs -l name=flux -n ${target_ns-flux} | head`. Do this for each application Flux that you have configured.
1. Add the _application_ Flux SSH keys to the _application_ GitHub bot account (remove the old keys if necessary).
1. Wait for the application pods to come up.
