# Workshop Kafka Laboratorio

Para configurar el laboratorio donde se van a desarrollar los ejercicios del workshop,
se debe configurar un ambiente con los siguientes requerimientos:

** Alternativa 1 **

1. Docker Desktop (Docker CE en Linux)
2. [Kubernetes in Docker](https://kind.sigs.k8s.io/)

** Alternativa 2 **

1. [Minikube](https://minikube.sigs.k8s.io/docs/start/)

La alternativa 1, permite tener más de un nodo en k8s y además, en el caso de Windows y Mac, permite usar Docker 
con la misma VM y en el caso de Linux, no necesita VM para k8s. Adicionalmente se necesitan las siguientes
herramientas:

1. kubectl
2. Java JDK 15
3. [krew](https://krew.sigs.k8s.io/)
4. [Skaffold](https://skaffold.dev/)

## Instalación en Ubuntu

1. Actualizar apt-get

```bash 
$ sudo apt-get update
```

2. Instalar dependencias de Docker 

```bash 
$ sudo apt-get install     apt-transport-https     ca-certificates     curl     gnupg     lsb-release
```

3. Registrar repositorio de Docker

```bash 
$ echo   "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
 $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

4. Actualizar apt-get

```bash 
$ sudo apt-get update
```

5. Instalar Docker CE

```bash 
$ sudo apt-get install docker-ce docker-ce-cli containerd.io
```

6. Instalar kind
   
```
$ curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.11.1/kind-linux-amd64
$ chmod +x ./kind
$ sudo mv ./kind /usr/bin/kind
```

7. Crear cluster k8s
   
```bash
kind create cluster --name strimzi-lab
```

8. Instalar kubectl

```bash
$ sudo apt-get install -y apt-transport-https ca-certificates curl
$ sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
$ echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
$ sudo apt-get update
$ sudo apt-get install -y kubectl
```

9. Instalar [krew](https://krew.sigs.k8s.io/) / **NO USAR EN PRODUCCION**/

```bash
$ (
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/krew.tar.gz" &&
  tar zxvf krew.tar.gz &&
  KREW=./krew-"${OS}_${ARCH}" &&
  "$KREW" install krew
)

$ echo 'export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"' > ~/.bashrc
$ export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

9. Instalar kubectx y kubens

```bash
$ kubectl krew install ctx
$ kubectl krew install ns
```

10. Instalar [Skaffold](https://skaffold.dev/)

```bash
$ curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64 && \
$ sudo install skaffold /usr/local/bin/
```

## Kafka

Kafka es relativsmente complejo de instalar y configurar en forma adecuada. Para simplificar la instalación y
la adminsitración de Kafka en k8s, vamos a utiliar una distribución de Kafka basada en un operador de k8s
perteneciente a la [Cloud Native Computing Foundation](https://www.cncf.io/), [Strimzi](https://strimzi.io/).

Para instalar strimzi sólo es necesario ejecutar el siguiente comando:

```bash
$ kubectl create namespace kafka
$ kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
$ kubectl get pod -n kafka --watch
```
