# k8s-jenkins
Automated Jenkins build with sensible plugins and no setup wizard intended to run CI/CD on Kubernetes

Available on [Docker hub](https://hub.docker.com/r/microdc/k8s-jenkins/)

  * [k8s-jenkins](#k8s-jenkins)
    * [Build](#build)
    * [Local run example](#local-run-example)
    * [Testing using Minikube](#testing-using-minikube)
    * [Deploy on Kubernetes](#deploy-on-kubernetes)
    * [Generate plugins.txt](#generate-pluginstxt)

## Build

```
export VERSION='local'
docker build --rm -t "microdc/k8s-jenkins:${VERSION}" .
# OR
./build.sh

```

## Local run example
This is only useful for testing changes to jenkins config. If you want to test kubernetes specific functionality follow the procedure below.  See step 3 below to generate the config files.
```
docker run --rm -p 8080:8080 -p 50000:50000 \
                -v "${PWD}/repos.txt":/usr/share/jenkins/data/repos.txt \
                -v "${PWD}/ssh_config/config":/var/jenkins_home/.ssh/config \
                -v "${HOME}/.ssh/id_rsa":/var/jenkins_home/.ssh/id_rsa \
                microdc/k8s-jenkins:local
```

## Testing using Minikube
1. [Install Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)
2. Start Minikube with a decent amount of memory
```
minikube start --memory 8192
```
3. Point your docker env to the MiniKube docker instance
```
eval $(minikube docker-env)
```
4. Build your image for minikube to use
```
docker build --rm -t "microdc/k8s-jenkins:local" .
```
5. Follow the instruction for 'Deploy on Kubernetes'

## Deploy on Kubernetes
1. Create a namespace for jenkins
```
kubectl create namespace jenkins
```
2. Create a config map for the git repos you will use (example file repos.txt)
```
kubectl create configmap jenkins-git-repos -n jenkins --from-file=repos.txt
```
3. Create Jenkins ssh config and keys secrets in Kubernetes
```
export DATE=$(date '+%Y-%m-%d')
mkdir -vp "${HOME}/.ssh/jenkins"
ssh-keygen \
    -t rsa -b 4096 -C "Jenkins ${DATE}" \
    -f "${HOME}/.ssh/jenkins/id_rsa"
cat > "${HOME}/.ssh/jenkins/config" << EOF
Host *
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
EOF
```
4. Add the ssh configuration to Kubernetes
```
kubectl create secret generic jenkins-ssh-config -n jenkins \
                                                 --from-file="${HOME}/.ssh/jenkins/config" \
                                                 --from-file="${HOME}/.ssh/jenkins/id_rsa" \
                                                 --from-file="${HOME}/.ssh/jenkins/id_rsa.pub"
```
5. Run in the config
```
kubectl apply -f k8s.yaml
```

6. Access using $(minikube ip) and NodePort port.


## Generate plugins.txt
From time to time we may need to generate a complete plugins list. This was generated from a container
following the original jenkins documentation [here](https://github.com/jenkinsci/docker/blob/master/README.md), like so:
```
JENKINS_HOST=admin:admin@localhost:8080
curl -sSL "http://$JENKINS_HOST/pluginManager/api/xml?depth=1&xpath=/*/*/shortName|/*/*/version&wrapper=plugins" | \
          perl -pe 's/.*?<shortName>([\w-]+).*?<version>([^<]+)()(<\/\w+>)+/\1 \2\n/g'|sed 's/ /:/' | \
          sort
```

