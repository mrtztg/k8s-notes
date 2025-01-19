# Core Concepts

### Kubernetes Architecture

![Kubernetes Architecture](assets/images/01_kubernetes_architecture.png)
![Kubenetes Architecture](assets/images/02_kubernetes_architecture_2.png)

- **Nodes** are physical servers, which can be on cloud or on-premise
- **Worker Nodes** host application as container s
- **Master Node** is response to mange, plan, monitor, schedule worker nodes
- `kubelet` in each worker node, is like captain. It’s responsible to introduce the node to master, listen to it, send status, schedule, so on
- Communication between worker nodes (like connection from webserver to db) are enable by `kube-proxy`
- k8s consists lots of components that can be installed and work with each other (like api-server). Each one of these components should be aware of existence of others and where they are, to be able to work with each other. There should be a authentication method between these components.

### Containers

- CRI (Container Runtime Interface) is a standard that any container (like Docker, Containerd, rkt) should’ve implement in their development to be able to used in k8s.
- `containerd` was container part of Docker. But became an independent part a few years ago. Because with k8s we don’t need the components of Docker (like CLI, API, BUILD, AUTH, etc) `containerd`  itself will be used in k8s, not the whole Docker. `containerd` can be installed separately without Docker
- Normally, `containerd` has `ctr` command, which has only very basic commands, which is okay when using k8s. Because k8s will be a middleman. But for debugging purposes, we can use either `nerdctl` which adds commands very similar to Docker commands to `containerd`, or we can use `crictl`, which is compatible with all `CRI` compatible containers systems including `containerd` . It commands also very similar to Docker commands
  ![CRI Clients](assets/images/03_cri_clients.png)

### ETCD

- **ETCD** is a distributed reliable key-value store that is Simple, Secure and Fast
- After installation and running, use `etcdctl` to deal with keyValues, users, so on. Before starting using `etcdctl` check the version of API using `etcdctl version`
- In k8s, `ETCD` stores information like Nodes, PODs, Configs, Secrets, Accounts, Roles, Bindings, so on. So, setting is only permitted when they reflect on etcd
- If we installed `etcd` manually, we can change the port of `etcd` panel using `etcd.service` file:

![etcd.service](assets/images/04_etcd_service.png)

- - But if we we installed k8s using `kubeadm`, it already installed etcd as a pod. We can explore database of etcd using etcdctl utility within this pod. For example for getting list of all keys should use `kubectl exec etcd-master -n kube-system ectdctl get / --prefit -keys-only`
    
    ![etcd pod in kubeadm setup](assets/images/05_etcd_kubeadm_pod.png)

- In HA environment, your etcd in each instance should be aware of each other

![etcd config in manual setup](assets/images/06_ectd_manual_service.png)

### Kube-apiserver

- `kube-apiserver` is primary management service in k8s. It sits in the centre of all tasks and changes made on k8s cluster. Actually, `kubectl` command reaches to `kube-apiserver` . It’s the only service that deal with `etcd`.
  - But also, we can use **HTTP requests** directly to `kube-apiserver`, instead of using `kubectl`
- If we installed k8s using `kubeadm`, `kube-apiserver` is installed as a pod. We can see its options within the pod definitions `cat /etc/kubernetes/manifests/kube-apiserver.yaml`

![Kube Api-Server Pod in Kubeadm setup](assets/images/07_kube_apiserver_pod_kubeadm.png)

![Kube Api-Server yaml in Kubeadm setup](assets/images/08_kub_apiserver_adm_setup_yaml.png)

- But if installed k8s manually, we should also install `kube-apiserver` manually. Then we can find its options in `cat /etc/systemd/system/kube-apiserver.service`

![Kube Api-Server in manual setup](assets/images/09_kube_apiserver_manual_setup.png)

- Also we can use `ps -aux | grep kube-apiserver` to see running processes of ApiServer
- When we send a request to kube-apiserver (either via HTTP or `kubectl` ) it follows the following flow by `kube-apiserver`  (this example is pod create command)
  - Authenticate User
  - Validate Request
  - Retrieve data
  - Update ETCD ( → then tells the user pod has been created, but actually it’s not yet)
  - The kube-scheduler will periodically watched `etcd`.
    - In this case, it’ll see there is a pod, with no Node assigned. So, it’ll define which Node should be used to place this pod.
    - Then it asks `kube-apiserver`  to create that pod in the proper node.
    - `kube-apiserver`  will update ETCD again and then send this call to `kubectl` in the desired Node
  - `kubelet` will create the proper containers and the back the status to `kube-apiserver`
    - `kube-apiserver`  updates etcd

![k8s madness](assets/images/10_k8s_madness.png)

### Kube Controller Manager

- Every intelligence in k8s sits inside `Kube Controller Manager`. It has lots of controllers including the one in picture below. They’re all enabled by default when we install Kube Controller Manager, but we can disable any of them

![Kube Controller Manager](assets/images/11_controller_manager.png)

- Controllers are like officers in master ship. Each Controller is responsible for a set of things in workers. One officers is responsible whenever worker ships comes and leave, so on.
  So, in kubernetes, Controller is responsible to make sure status of different components are in desired status (**Watch Status, Remediate Situation)**. Eg:
  
  - Node Controller, checks the status of nodes every 5 sec (is changeable), if is unhealthy, if stays unhealthy for 40 seconds marks is at unhealthy. If it’s unhealthy for 5 minutes, it removes pods assigned to that Node and assign pods to a healthy Node.
  - Replication Controller, makes sure the desired number of pods are available.

- `Kube Controller manager` will be installed automatically if we use `kubeadm` . If we installed K8S manually, we should install Controller Manager as well.

- Its option is located in `controller-manager-master`  pod if we used `kubeadm` or in service file if we installed manually.
  
    ![Controller Manager service in manual setup](assets/images/12_controller_manager_manual_service.png)
  
    When we install k8s and Controller manager manually

![Controller manager pod in Kubeadm setup](assets/images/13_controller_manager_pod.png)

1. When install using kubeadm

![Controller manager pod yaml in Kubeadm setup](assets/images/14_controller_manager_pod_yaml.png)

2. To see options in k8s pod
- To see running `controller-manager` processes, run the following command on master Node: `ps -aux | grep kube-controller-manager`
- Navigate to sector **Controller Manager** below to to learn about each controllers

### Kube Scheduler

- It’s only responsible for deciding which pod goes to which Node. It won’t place pod. Placing pod is responsibility of `kubelet`
- It decides the Node based on requirements and criteria like:
  - Resource Requirements and Limits
  - Taints and Tolerations
  - Node Selector/Affinity
- To we can see options similar to how we did in KubeControllerManager

### Kubelet

- It’s the captain of worker Node. Responsible to Register Nodes in Kubernetes cluster.
- When it receives construction to load a container (from scheduler, but via apiserver), it requests container runtime engine (which maybe Docker) to pull the image and run an instance.
- Monitor the pods and periodically reports status of Pods to apiserver
- unlike previous components, **kubeadm does not automatically deploy Kubectl. We always should install it manually on worker node**.
- We can see option in kubelet.service file. Also for listing running options, run the `ps -aux | grep kubelet` on worker node.

### kube-proxy

- There are different solutions for bring networking between k8s pods. This communication better to not to be with IP due to its inconsistency. Better to use service to connect each other.
  But the service itself cannot join pod network by itself, because its just a visual network which lives in k8s memory, not actual component.
  So, how services can be joined to pods network? Using **Kube-proxy**. Kube-proxies run  on each Node, and whenever a new services gets created, it creates proper rules (like using `iptables`) on each Node to forward traffic to those services to the backend pods.

![Kube-proxy](assets/images/15_kube_proxy.png)

- You can install `kube-proxy` manually if you installed k8s manually. Or kubeadm will deploy it as Pod

![Kube-proxy Pod](assets/images/16_kube_proxy_pod.png)

## Kubenetes Pods

- Pods are the smallest object that you can create on Kubernetes
- Container should be in Pod, it can’t be standalone
- For **scaling purpose, we shouldn’t deploy another container instance in the same Pod**. Instead, it should be a Pod with the new instance.
  If our Node doesn’t have enough required capacity for adding more instances, we can add Pod to the next Nodes.

![k8s Pods](assets/images/17_k8s_pods.png)

- Sometimes (rarely) we may need to have more than 1 instance in a Pod, like when our main app needs a helper Sqlite DB for its quick operations and that DB isn’t needed to be accessible for other instances or other services. These instances are in the same Pod, and their network is local and isolated. Also, they have access to each other’s storage.

![Multiple containers in a Pod](assets/images/18_k8s_containers_in_pod.png)

- K8s considers all instances inside a Pod as one object. It shares volumes and volumes of Pod’s instances to each other automatically, it maps them to each other, so on. So, it removes, create the whole Pod completely. It means, for example when the helper DB or the worker app (which sits in the same Pod) gets unhealthy, k8s will kill not only the worker app in that Pod, but even helper DB instance
- For creating Pods, we run the following cmd: `kubectl run nginx --image=nginx` .
  - k8s will create Pod for us, and gets nginx image from Docker hub, and create instance inside that Pod.
  - In this example, `--image=nginx` means from public Docker hub, but we also can define to get image from private repo
- For listing running Pods: `kubectl get pods` or `kubectl get pods -o wide` for detailed info
  - By default, the created Pod won’t be accessible for end users, but we can access to the instance using k8s itself.
- On `kubectl get pods` command, READY column is: `running containers in pod/total containers in pod`
- To see spec of running Pod: `kubectl describe pod myapp-pod`
- For deleting pods: `kubectl delete pods mywebapp`

### K8s YAML files

- Kubernetes uses yaml files as inputs for creation of objects like Pods, replicas, services, deployments, etc. It always have the 4 top level fields: `apiVersion, kind, metadata, spec`

```yaml
# Take care of indents. Siblings should be in the same
#   level of indents, and child of parent should have more
#   indent compared to parent
apiVersion: v1 //refer to table below
kind: Pod
metadata:
# all key-values inside metadata should be in k8s defined
#    list, like name, labales, so on
    name: myapp-pod 
    labels:
#      but keys inside labels can be anything
        app: myapp
#        We can use such fields to different purpose, like filterring:
        type: backend 
spec:
    containers:
#      for each item of list, we use '-':
        - name: nginx-container
          image: nginx
```

| Kind                                | Version |
| ----------------------------------- | ------- |
| Pod                                 | v1      |
| Service                             | v1      |
| ReplicaSet                          | apps/v1 |
| Deployment                          | apps/v1 |
| ReplicationController (Deperacated) | v1      |

- We can also create definition YAML file by adding `dry-run` argument, like:
  
  ```bash
  kubectl run redis --image=redis --dry-run=client -o yaml > redis-pod-def.yaml
  ```

- Once you created the config file, use `kubectl create -f pod-definition.yml` for creating or use `kubectl apply -f pod-definition.yml` for upsert (insert or update)

# Controller Managers

## ReplicationController / ReplicaSet

- Why we need ReplicationController?
  
  1. Maintain desired number of replicas to make sure our app is always available. Replica number can be 1 (means only one instance of that app is running at the time). But 2 is better if we have HA (high availability) in mind.
     
     ![ha using replica](assets/images/19_highavailability_using_replica.png)
  
  2. Load balancing & Scalability. ReplicationController will make sure request load will splitted in all replicas. It can even deploy replicas in other Nodes if required
     
     ![scalability using replica](assets/images/20_scalability_using_replica.png)

- **ReplicationController** and **ReplicaSet** does the same thing, but they're not the same. **ReplicaSet** is a newer version and have more capability, like `selector` in definition file.

- In `spec > template` section of definition YAML file, we should have the exact code in Pod definition file except `apiVersion` and `kind` fields.

- In **ReplicaSet** manager, we can define `selector` inside `spec` section. We define it to let ReplicaSet controller know that even if we deployed the similar Pod/app manually, count them as your replicas. **ReplicationController** doesn't have selector section. Sample definition file:
  
  ```yaml
  # apiVersion if kind:ReplicationController
  apiVersion: apps/v1
  kind: ReplicaSet
  metadata:
    name: myapp-replicaset
    labels:
      app: myapp
      type: backend
  spec:
    replicas: 3 
  # define 'selector' only if kind is ReplicaSet
    selector:
  # matchLabels will be used to consider all Pods with this labels (even manually created) part of this replica
  # Also the values inside matchLabels should match labels in template section below
      matchLabels:
        type: backend
  # Inside 'template' should be exactly desired Pod definition
    template:
      metadata:
        labels:
          type: backend
      spec:
        containers:
          - name: myapp
            image: redis
  ```

```
- For creating using replicaset or replicationController:

```bash
kubectl apply -f my_replica.yaml
```

- To see Repli status, use one of the following
  
  ```bash
  # For replicationcontroller
  kubectl get replicationcontroller
  # For replicaset
  kubectl get replicaset
  # or
  kubectl get rs
  ```
  
  - But you still can use `kubectl get pods` to list pods, you'll see replicationController added a suffix to each replica

- To delete replicaset all all underlying PODs (ReplicationController is similar):
  
  ```bash
  kubectl delete replicaset my-replica
  ```

- To edit replica set, use the following command:
  
  ```bash
  kubectl edit replicaset my-replica
  ```
  
  - The changes related to POD (like POD image) won't be reflected on existing PODs of replica. You should delete them, and new ones will be created using the edited definition.
    But changings related to Replica itself (replica number) will reflected immediately 

**Scaling Replicas**

- We can either
  
  - Update YAML file, change replica value, and then run `kubectl replace -f my-replica.yaml` command
  
  - Or change replicas temporarily using `kubectl scale --replicas=5 -f my-replica.yaml`
  
  - Or by defining the name of the created replica, instead of yaml file name `kubectl scale --replicas=5 [TYPE] [NAME]` like `kubectl scale --replicas=5 replicaset my-replicaset`

**Deployments**

- Kubernetes can handle different scenarios and strategies related to deployment to production env. Like:
  
  - Deploying app to multiple instances
  
  - Upgrading the existing docker instances gradually, not all at once
  
  - Rollback the changes recently been made
  
  - Pause, upgrade and then resume instances whenever we want to update underlying configurations of pods

- Deployments in a higher hierarchy to ReplicaSet
  
  ![Deployments](assets/images/21_deployments.png)

- Definition file of Deployments is very similar to ReplicaSet, except the `kind` field which should be `Deployment`

- You can created Deployments either using command, or definition file:
  
  1. Use `kubectl create -f my-deployment.yaml` to create deployment from definition file.
  
  2. Use command like `kubectl create deployment httpd-frontend --image=httpd:2.4-alpine --replicas=3` 

- Use `kubectl get deployment`  or `kubectl get deploy` to list created deployments. Or use `kuectl get all` to list all 

## Services

- Kubernetes services enables communication between the components inside and outside of application. It helps to connect applications to other applications or users, like communication between front-end and backend, or between user and frontend
  
  ![Services](assets/images/22_services.png)

- Normally, the application inside k8s is not accessable from outside, excpet if we connect to k8s Node via SSH. 
  
  ![Connect to Pod via SSH](assets/images/23_connect_to_pod.png)
  
  - But using service, we can give access:
  
  ![Access Pod via Service](assets/images/24_access_pod_via_service.png)

- **Services Type**:
  1.  **NodePort**: To give incoming access to the Pod from ouside of the Node

    ![NodePort Service](assets/images/25_node_port.png)

    - NodePort range is `30_000` to `32_767`

  2. **CluesterIP**: Create virtual IP inside cluster to enable communication between Pods like set of front-end servers with back-end servers
      ![ClusterIP Service](assets/images/27_clusterip.png)
  3. **LoadBalancer**: Enables loadBalancer in the supported cloud providers (like AWS, GCP, Azure). For example we pub AWS Application Load Balancer in front and Kubernetes service (with type=LoadBalancer) will do the rest of configuration in a way that the traffic will be received through the domain that is set to AWS ALB, and load will be balanced.
    ![Load Balancer Serivce](assets/images/28_loadbalancer_service.png)
    - Note that we still can access to underlying Pods using IP:PORT of any of the Pods as example below. But that't not our desired way of accessing. We want a single URL.
      ![Service without endpoint](assets/images/29_service_without_endpoint.png)
- Sample of definition file:

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: my-service
  spec:
    # Possible options for type are: ClusterIP, NodePort, LoadBalancer
    #   Default value is ClusterIP if we don't define. 
    type: NodePort 
    ports:
      - targetPort: 80 # if we don't define targetPort, will be equal to port
        port: 80
        # We don't define nodePort in ClusterIP
        nodePort: 30008 # If we don't define, it'll be first available nodePort
    selector: # K8s uses selector, to know which Pods should be targetted.
      app: myapp
      type: front-end
  ```
- If there are more than one Pod for the specifiec 'Selector', the service will act as the load balancer and will split traffic to all targets using `Random` Algorithm. These multiple pods can be in one Node, or in multiple Nodes. Then we can use IP of any of the target Nodes to access underlying Pods of service. In the following picture `curl http://192.168.1.2:300008` or `curl http://192.168.1.3:300008` or `curl http://192.168.1.4:300008`

  ![Multi Node, Multi Pod as Service Target](assets/images/26_service_target_multi_node.png)
  
- To create service using definition file, use `kubectl create -f my-service.yaml` and for listing the running services: `kubectl get services` or `kubectl get svc`
- Kubernetes creates a ClusterIP in the beginning for us (I still don't know the reason behind)
### Additional Commands

- Get all running components in groups

```bash
kubectl get all
```

# Cheat Sheets

[Kubernetes Quick Reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

# References

[Kubernetes Awesome](https://awesome-architecture.com/devops/kubernetes/kubernetes/)

# Exam Tips

- k8s in exam has been installed using `kubeadm`  which
  - already deployed etcd, Kube-Apiserver, Kube-Scheduler, Kube-Controller-Manager as Pods
- Createing YAML files are time consuming during the exam. Instead, try to use dry-run commands to create YAML, or in some cases you can directly create from command instead of YAML
  - Create an NGINX Pod 
    - `kubectl run nginx --image=nginx`
  - Generate POD Manifest YAML file (-o yaml). Don't create it(--dry-run) 
    - `kubectl run nginx --image=nginx --dry-run=client -o yaml`
  - Create a deployment
    - `kubectl create deployment --image=nginx nginx`
  - Generate Deployment YAML file (-o yaml). Don't create it(--dry-run)
    - `kubectl create deployment --image=nginx nginx --dry-run=client -o yaml`
  - Generate Deployment YAML file (-o yaml). Don’t create it(–dry-run) and save it to a file.
    - `kubectl create deployment --image=nginx nginx --dry-run=client -o yaml > nginx-deployment.yaml`
  - Make necessary changes to the file (for example, adding more replicas) and then create the deployment.
    - `kubectl create -f nginx-deployment.yaml`

  - In k8s version 1.19+, we can specify the --replicas option to create a deployment with 4 replicas.
    - `kubectl create deployment --image=nginx nginx --replicas=4 --dry-run=client -o yaml > nginx-deployment.yaml`

