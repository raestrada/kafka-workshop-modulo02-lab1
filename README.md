# Workshop Kafka Laboratorio

> Una recomendación es trabajar en Linux y en caso de no tener recursos suficientes, se puede crear una cuenta
gratis por USD$100 en [Digital Ocean's](https://try.digitalocean.com/freetrialoffer) o en [Linode](https://www.linode.com/lp/free-credit-100) y disfrutar de una máquina virtual Ubuntu de 4 cores
y 8GBG RAM. Para trabajo local, se puede montar la carpeta de trabajo de la VM usando [SSHFS](https://www.digitalocean.com/community/tutorials/how-to-use-sshfs-to-mount-remote-file-systems-over-ssh).

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

El comando anterior va a crear el operador de Strimzi escuchando el namespace kafka. Ahora para crear un cluster Kafka, primero, crea un manifiesto de k8s para crear un cluster de kafka básico y guárdalo en ```k8s/kafka.yaml```:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: workshop-lab
  namespace: kafka
spec:
  kafka:
    version: 2.8.0
    replicas: 1
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      log.message.format.version: "2.8"
      inter.broker.protocol.version: "2.8"
    storage:
      type: jbod
      volumes:
      - id: 0
        type: persistent-claim
        size: 1Gi
        deleteClaim: false
  zookeeper:
    replicas: 1
    storage:
      type: persistent-claim
      size: 1Gi
      deleteClaim: false
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

Ahora instalemos Skaffold para hacer despliegue de yaml de k8s:

```bash
$ skaffold init --skip-build --kubernetes-manifest="k8s/kafka.yaml"
```

Finalmente, se debe ejecutar skaffold para desplegar el cluster Kafka:

```bash
$ skaffold run
$ kubectl get pod -n kafka --watch
```

La configuración básica, va a tener un broker y un nodo de zookeeper:

```bash
$ kubectl -n kafka get all
```

```csv
NAME                                                READY   STATUS    RESTARTS   AGE
pod/strimzi-cluster-operator-6c7b6c4c6-25r9l        1/1     Running   0          3h12m
pod/workshop-lab-entity-operator-86998bc5d6-z2hlr   3/3     Running   0          7m52s
pod/workshop-lab-kafka-0                            1/1     Running   0          8m21s
pod/workshop-lab-zookeeper-0                        1/1     Running   0          9m

NAME                                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                               AGE
service/workshop-lab-kafka-bootstrap    ClusterIP   10.96.208.43    <none>        9091/TCP,9092/TCP,9093/TCP            8m21s
service/workshop-lab-kafka-brokers      ClusterIP   None            <none>        9090/TCP,9091/TCP,9092/TCP,9093/TCP   8m21s
service/workshop-lab-zookeeper-client   ClusterIP   10.96.136.185   <none>        2181/TCP                              9m1s
service/workshop-lab-zookeeper-nodes    ClusterIP   None            <none>        2181/TCP,2888/TCP,3888/TCP            9m1s

NAME                                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/strimzi-cluster-operator       1/1     1            1           3h12m
deployment.apps/workshop-lab-entity-operator   1/1     1            1           7m52s

NAME                                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/strimzi-cluster-operator-6c7b6c4c6        1         1         1       3h12m
replicaset.apps/workshop-lab-entity-operator-86998bc5d6   1         1         1       7m52s

NAME                                      READY   AGE
statefulset.apps/workshop-lab-kafka       1/1     8m21s
statefulset.apps/workshop-lab-zookeeper   1/1     9m
```

Para probar el funcionamiento, puedes iniciar un pod productor:

```bash
kubectl -n kafka run kafka-producer -ti --image=quay.io/strimzi/kafka:0.23.0-kafka-2.8.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --broker-list workshop-lab-kafka-bootstrap:9092 --topic my-topic
```

y luego en un termina diferente un pod consumidor:

```bash
kubectl -n kafka run kafka-consumer -ti --image=quay.io/strimzi/kafka:0.23.0-kafka-2.8.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server workshop-lab-kafka-bootstrap:9092 --topic my-topic --from-beginning
```

Ahora puedes escribir mensajes en el productor y ver como el consumidor los recibe.
