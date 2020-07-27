# Solving the DevOps with Kubernetes MOOC

This repo contains my solutions for the exercises of <https://devopswithkubernetes.com/>.

- [Solving the DevOps with Kubernetes MOOC](#solving-the-devops-with-kubernetes-mooc)
  - [Solutions for Part 1](#solutions-for-part-1)
  - [Solutions for Part 2](#solutions-for-part-2)
    - [Deploying Solution for Ex. 2.03](#deploying-solution-for-ex-203)
    - [Exercise 2.05](#exercise-205)
    - [Exercise 2.06](#exercise-206)
    - [Exercise 2.07](#exercise-207)
    - [Exercise 2.08](#exercise-208)
  - [Solutions for Part 3](#solutions-for-part-3)
    - [Exercise 3.01](#exercise-301)

## Solutions for Part 1

| exercise | solution in subfolder        |
| -------- | ---------------------------- |
| 1.01     | `hashgenerator`              |
| 1.02     | `project`                    |
| 1.03     | `hashgenerator`              |
| 1.04     | `project`                    |
| 1.05     | `project`                    |
| 1.06     | `project`                    |
| 1.07     | `hashgenerator`              |
| 1.08     | `project`                    |
| 1.09     | `hashgenerator` & `pingpong` |
| 1.10     | `hashgenerator`              |
| 1.11     | `hashgenerator`              |
| 1.12     | `project`                    |
| 1.13     | `project`                    |

Against [better judgement](https://vsupalov.com/docker-latest-tag/) I used the "latest" tag for some assignments and Kubernetes `yaml` configurations. That's why some solutions can't be played through again, because the automatically built Image at Docker Hub doesn't match the step of the solution anymore. This is especially true for the main application. The project on the other hand is logically tagged.

## Solutions for Part 2

_The solution for exercises are based on each other. For this reason, not all of them are documented here._

### Deploying Solution for Ex. 2.03

```sh
# Create ping/pong app with namespace
$ kubectl create namespace pingpong
namespace/pingpong created
$ kubectl apply -f pingpong/manifests/
deployment.apps/pingpong-dep created
ingress.extensions/pingpong-ingress created
service/pingpong-svc created
# test
$ curl -s http://localhost:8081/pingpong
{"counter":1}
# Create main app in namespace that fetches ping/pongs and prints uuid every 5secs
$ kubectl create namespace hashgenerator
namespace/hashgenerator created
$ kubectl apply -f hashgenerator/manifests/
deployment.apps/hashgenerator-dep created
ingress.extensions/hashgenerator-ingress created
service/hashgenerator-svc created
# test
$ curl -s http://localhost:8081/pingpong
{"counter":2}
$ curl -s http://localhost:8081/
2020-07-17T11:44:52.631Z: 4bdeb719-73cb-482f-b66c-40d04d0d880d<br> 2
# what's in the namespaces?
$ kubens pingpong
Context "k3s-default" modified.
Active namespace is "pingpong".
$ kubectl get all
NAME                                READY   STATUS    RESTARTS   AGE
pod/pingpong-dep-777dbbd675-5kvrr   1/1     Running   0          3m49s

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/pingpong-svc   ClusterIP   10.43.219.90   <none>        6789/TCP   3m49s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/pingpong-dep   1/1     1            1           3m49s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/pingpong-dep-777dbbd675   1         1         1       3m49s
$ kubens hashgenerator
Context "k3s-default" modified.
Active namespace is "hashgenerator".
$ kubectl get all
NAME                                     READY   STATUS    RESTARTS   AGE
pod/hashgenerator-dep-66c9cc599d-wgkzg   2/2     Running   0          2m31s

NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/hashgenerator-svc   ClusterIP   10.43.228.139   <none>        3001/TCP,2345/TCP   2m31s

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hashgenerator-dep   1/1     1            1           2m31s

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/hashgenerator-dep-66c9cc599d   1         1         1       2m31s
```

### Exercise 2.05

ConfigMaps for `hashgenerator-server` aka main application

```sh
$ kubectl config set-context --current --namespace=hashgenerator
# Create ConfigMap from `hashgenerator-env-file.properties`
$ cat hashgenerator/manifests/hashgenerator-env-file.properties
SERVER_PORT=3001
GENERATOR_PORT=3002
HASHGENERATOR_URL=http://hashgenerator-svc:2345/
PINGPONG_URL=http://pingpong-svc.pingpong:6789/pingpong
MESSAGE=Hello%
$ kubectl create configmap hashgenerator-config-env-file --from-env-file hashgenerator/manifests/hashgenerator-env-file.properties
configmap/hashgenerator-config-env-file created
# It's live
$ curl -s http://localhost:8081
Hello<br> 2020-07-17T17:25:59.300Z: 14c4d2ff-25f2-4b45-86b8-4b7e3b80b435<br> 13
```

### Exercise 2.06

This spins up two pods. One Deployment with the ping/pong app and one StatefulSet with a PostgreSQL database container. During the creation of the database pod, the very very complex schema gets seeded. The database password is stored as as SealedSecret.

```sh
$ kubectl apply -f pingpong/manifests/
deployment.apps/pingpong-dep created
ingress.extensions/pingpong-ingress created
sealedsecret.bitnami.com/postgres-pw created
service/pingpong-svc created
service/postgres-pingpong-svc created
configmap/postgres-pingpong-seed created
statefulset.apps/postgres-pingpong-stateful created
$ curl -s http://localhost:8081/pingpong
{"counter":4}%
$ kubectl logs -f pingpong-dep-6bcb74f875-w5r72 pingpong

> pingpong@1.0.0 start /app
> node server.js

Writing to txt is disabled
NEVER log a db password to stdin: 'passw0rd'
Server started on: 3000
added a new ping to db
added a new ping to db
added a new ping to db
```

### Exercise 2.07

The state of the todo application gets stored in a PostgreSQL database. The app hooked up via [Sequelize](https://github.com/sequelize/sequelize). The table get created based on the [Task model](https://github.com/movd/devopswithkubernetes/blob/2229fc1779d08d97d4168b44536a96c599a6882c/project/backend/src/models/task.ts). For deployment the db connection parameter and password are stored and read from `secret.yaml`. This time around I'm not using SealedSecret. Only the backend and the deployment setup where changed.

<table>
<tr>
<td width="60%">
<pre lang="sh">
$ kubectl apply -f project/manifests/
deployment.apps/project-dep created
ingress.extensions/project-ingress-frontend created
ingress.extensions/project-ingress-api-static created
persistentvolume/project-pv created
persistentvolumeclaim/project-claim created
secret/postgres-pw-url created
service/project-svc created
service/postgres-project-svc created
statefulset.apps/postgres-project-stateful created
</pre>
</td>
<td width="30%">
<code>http://localhost:8081/project</code><br><img src="frontend-screenshot.png" alt="Screenshot of frontend">
</td>
</tr> 
</table>

### Exercise 2.08

I followed the [steps in the course notes](https://devopswithkubernetes.com/part2/#monitoring) and installed Grafana and Loki via Helm. I added request logging via [morgan](https://github.com/expressjs/morgan) to my project application.

```sh
$ kubectl apply -f project/manifests/
deployment.apps/project-dep created
ingress.extensions/project-ingress-frontend created
ingress.extensions/project-ingress-api-static created
# ...just like in the exercise before
```

Grafana dashboard:

```sh
$ kubectl port-forward prometheus-operator-1595404775-grafana-7964b4fb7d-4hflq 3000
Forwarding from 127.0.0.1:3000 -> 3000
Forwarding from [::1]:3000 -> 3000
Handling connection for 3000
```

![Screenshot of Grafana with Loki](grafana-loki-screenshot.png)

## Solutions for Part 3

### Exercise 3.01

Create cluster in Frankfurt with two nodes and connect to it:

```sh
$ gcloud container clusters create dwk-cluster --zone=europe-west3 --num-nodes=2
Creating cluster dwk-cluster in europe-west3... Cluster is being configured...Cluster is being deployed...Cluster is being health-checked...done.
Created [https://container.googleapis.com/v1/projects/dwk-gke-123456/zones/europe-west3/clusters/dwk-cluster].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/europe-west3/dwk-cluster?project=dwk-gke-123456
kubeconfig entry generated for dwk-cluster.
NAME         LOCATION      MASTER_VERSION  MASTER_IP     MACHINE_TYPE   NODE_VERSION   NUM_NODES  STATUS
dwk-cluster  europe-west3  1.15.12-gke.2   34.89.245.27  n1-standard-1  1.15.12-gke.2  6          RUNNING

$ gcloud container clusters get-credentials dwk-cluster --region europe-west3 --project dwk-gke-123456
Fetching cluster endpoint and auth data.
kubeconfig entry generated for dwk-cluster
```

Create ping/pong App with it's own public ip adress. So that external users can ping it via:`http://EXTERNAL-IP/pingpong`. The `hashgenerator-server` reaches it internally via `http://pingpong-svc.pingpong:6789/pingpong`

```
$ kubectl create namespace pingpong
namespace/pingpong created
$ kubectl apply -f pingpong/manifests-gke/
deployment.apps/pingpong-dep created
secret/postgres-pw created
service/pingpong-svc-loadbalancer created
service/pingpong-svc created
service/postgres-pingpong-svc created
configmap/postgres-pingpong-seed created
statefulset.apps/postgres-pingpong-stateful created
$ kubens pingpong
Context "gke_dwk-gke-123456_europe-west3_dwk-cluster" modified.
Active namespace is "pingpong".
# Wait for external IP...
$ kubectl get svc --watch
NAME                        TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
pingpong-svc                ClusterIP      10.51.248.145   <none>        6789/TCP       42s
pingpong-svc-loadbalancer   LoadBalancer   10.51.246.118   <pending>     80:31106/TCP   42s
postgres-pingpong-svc       ClusterIP      10.51.255.120   <none>        5432/TCP       42s
pingpong-svc-loadbalancer   LoadBalancer   10.51.246.118   34.89.233.5   80:31106/TCP   50s
```

ping to pong:

```sh
$ curl -s http://34.89.233.5/pingpong
{"counter":0}%
$ curl -s http://34.89.233.5/pingpong
{"counter":1}%
```

Create the hashgenerator app + server:

```
$ kubectl create namespace hashgenerator
namespace/hashgenerator created
$ kubens hashgenerator
Context "gke_dwk-gke-123456_europe-west3_dwk-cluster" modified.
Active namespace is "hashgenerator".
$ kubectl apply -f hashgenerator/manifests-gke/
configmap/hashgenerator-config-env-file created
deployment.apps/hashgenerator-dep created
service/hashgenerator-svc-loadbalancer created
service/hashgenerator-svc created
# Wait for external IP address
$ kubectl get svc --watch
NAME                             TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
hashgenerator-svc                ClusterIP      10.51.249.229   <none>        2345/TCP       35s
hashgenerator-svc-loadbalancer   LoadBalancer   10.51.241.121   <pending>     80:32281/TCP   35s
hashgenerator-svc-loadbalancer   LoadBalancer   10.51.241.121   35.242.205.114   80:32281/TCP   42s
```

Both `LoadBalancer`s should be running now:

```sh
$ kubectl get svc --all-namespaces | grep LoadBalancer
hashgenerator   hashgenerator-svc-loadbalancer   LoadBalancer   10.51.241.121   35.242.205.114   80:32281/TCP    6m17s
pingpong        pingpong-svc-loadbalancer        LoadBalancer   10.51.246.118   34.89.233.5      80:31106/TCP    10m
```

It works:

```sh
$ curl http://35.242.205.114/
Hello<br> 2020-07-27T16:55:59.611Z: 5fa214bc-a6a5-4fc7-ad28-8dd7c8bc0b53<br> 2%
$ kubectl logs hashgenerator-dep-5764f96cc5-jrq4z --all-containers
HASHGENERATOR_URL: http://hashgenerator-svc:2345/
HASHFILE_PATH: undefined
PINGPONG_URL: http://pingpong-svc.pingpong:6789/pingpong
PINGPONGFILE_PATH: undefined
Server started on: 3001
::ffff:10.48.5.1 requested a hashFile + pingPongFile
Writing to txt is disabled
Server started on: 3002
current: 2020-07-27T16:55:19.580Z: 6f2c833d-c268-4c92-a4a2-045710d48615
current: 2020-07-27T16:55:24.581Z: 2b83ff58-b42d-4bb4-9c9a-8937bfa9e8f9
current: 2020-07-27T16:55:29.586Z: 79261074-c50a-433e-8557-1d55269857d4
current: 2020-07-27T16:55:34.591Z: 82d34fad-04b6-4227-97da-61d73baf418a
```

Delete the cluster:

```
$ gcloud container clusters delete dwk-cluster
The following clusters will be deleted.
 - [dwk-cluster] in [europe-west3]

Do you want to continue (Y/n)?  y

Deleting cluster dwk-cluster...⠹
```
