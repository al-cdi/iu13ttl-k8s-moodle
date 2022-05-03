# Let's build an LMS... on Kubernetes!

## Environment Setup

1. [Install Docker Engine](https://docs.docker.com/engine/install/ubuntu/)
    - If on windows, follow [these instructions](https://docs.docker.com/desktop/windows/install/).
    - Note that if you opt to installing Docker desktop, licensing restrictions may apply.
    - Run ```sudo service docker  start``` after installing!
        - This starts the docker service which will be required by kind to setup your k8s cluster.
2. [Install Kind](https://kind.sigs.k8s.io/docs/user/quick-start/)
3. [Install Helm](https://helm.sh/docs/intro/install/)
4. [Install Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
5. [Install Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

## 1 - Clone the Git Repo

- Let's clone this git repo to access the extra manifests required to setup Moodle.

```
git clone git@github.com:al-cdi/iu13ttl-k8s-moodle.git
```

## 2 - Create your kind cluster

- Let's create a k8s cluster using kind (don't worry - it's easy!).

```
kind create cluster --name moodle
```

- This will create a kind cluster using the latest available kind k8s image.
- After a short time you should be able to run ```kubectl get nodes``` and see your kind cluster's single node.

## 3 - Install Kubeapps

The below commands are from this kubeapps [installation guide](https://tanzu.vmware.com/developer/guides/kubeapps-gs/) (from VMware).

- Install kubeapps via helm

```
helm repo add bitnami https://charts.bitnami.com/bitnami
kubectl create namespace kubeapps
helm install kubeapps --namespace kubeapps bitnami/kubeapps --set useHelm3=true
```

- Create API token, this is used to login to the kubeapps dashboard.

```
kubectl create serviceaccount kubeapps-operator
kubectl create clusterrolebinding kubeapps-operator --clusterrole=cluster-admin --serviceaccount=default:kubeapps-operator
```

- Let's retrieve the token by running the command below.
    - Save this for use in the next step.

```
kubectl get secret $(kubectl get serviceaccount kubeapps-operator -o jsonpath='{range .secrets[*]}{.name}{"\n"}{end}' | grep kubeapps-operator-token) -o jsonpath='{.data.token}' -o go-template='{{.data.token | base64decode}}' && echo
```

## 4 - Access Kubeapps Dashboard

- To initially access the kubeapps dashboard, we will use port-forward the kubeapps service to our local machine.
    - Sysadmins rejoice! YES - you can portforward directly to your system via kubectl. 
    - Super nice for troubleshooting and quick testing :)!

```
kubectl port-forward -n kubeapps svc/kubeapps 8080:80
```

- Visit http://127.0.0.1:8080/ in your local web browser, you should now see kubeapps!
- Use the token previously retrieved to authenticate to the dashboard.

## 5 - Install Moodle

- From the kubeapps dashboard, go to the top right hand corner of the screen, and create a new namespace called "moodle".
- Navigate to the "catalog" menu option, select it, and search for "moodle".
- Install moodle!

## 6 - Access Moodle

- Give moodle a few minutes to install, the database and webserver pods take some time to spin up.
- Use the below port-forward command to access moodle once the pods are running.
    - You can check pod status using the ```kubectl get pods -n moodle``` command.

```
kubectl port-forward -n moodle svc/moodle 8080:80
```

## 7 - Clean Up

- Delete kind cluster.

```
kind delete cluster --name moodle
```

## 8 - Ingress Setup (Optional)

- Recreate your kind cluster using the below command.
    - This adds an additional networking option so that our ingress controller can work.

```
kind create cluster --name moodle --config kind-cluster.yaml
```

- Run through steps 3-6 again from above.
- Use the below command to install the nginx ingress-controller.

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

- Check to see if the nginx ingress controller pod is running, once ingress-nginx-controller-x is Running 1/1, we know it is operational.

```
k get pods -n ingress-nginx
```

## 9 - Moodle Ingress Setup

- Use the below kubectl command to apply the moodle ingress configuration.

```
kubectl apply -f moodle-ingress.yaml
```

- Review the kubectl output for getting the ingress using the below command.

```
kubectl get ingress -n moodle
```

## 10 - Moodle Scale Web Server Pods

- Use the command below and change the web server deployment replica count to "2" in the text editor.

```
kubectl scale deployment/moodle --replicas=2 -n moodle
kubectl get pods -n moodle
```

- Notice that you now have 2 moodle web server pods running! 
- The service that maps to these pods effectively load balances traffic to them.

## 11 - Clean Up

- Delete kind cluster.

```
kind delete cluster --name moodle
```