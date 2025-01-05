# Core Concepts

### Kubernetes Architecture
![Kubernetes Architecture](practices/01/assets/images/01_kubernetes_architecture.png)
![Kubenetes Architecture](practices/01/assets/images/02_kubernetes_architecture_2.png)
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
![CRI Clients](practices/01/assets/images/03_cri_clients.png)

### ETCD

- **ETCD** is a distributed reliable key-value store that is Simple, Secure and Fast
- After installation and running, use `etcdctl` to deal with keyValues, users, so on. Before starting using `etcdctl` check the version of API using `etcdctl version`
- In k8s, `ETCD` stores information like Nodes, PODs, Configs, Secrets, Accounts, Roles, Bindings, so on. So, setting is only permitted when they reflect on etcd
- If we installed `etcd` manually, we can change the port of `etcd` panel using `etcd.service` file:

![etcd.service](practices/01/assets/images/04_etcd_service.png)

- 
    - But if we we installed k8s using `kubeadm`, it already installed etcd as a pod. We can explore database of etcd using etcdctl utility within this pod. For example for getting list of all keys should use `kubectl exec etcd-master -n kube-system ectdctl get / --prefit -keys-only`
    
    ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/6afc06fe-8e9b-4b53-84e2-9148620cb602/f762a0d9-7b1f-448a-94f2-e641471f2fe4/image.png)
    
- In HA environment, your etcd in each instance should be aware of each other

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/6afc06fe-8e9b-4b53-84e2-9148620cb602/3ea00485-088d-4cca-a141-b8dce0e214ed/image.png)

### Kube-apiserver

- `kube-apiserver` is primary management service in k8s. It sits in the centre of all tasks and changes made on k8s cluster. Actually, `kubectl` command reaches to `kube-apiserver` . It’s the only service that deal with `etcd`.
    - But also, we can use **HTTP requests** directly to `kube-apiserver`, instead of using `kubectl`
- If we installed k8s using `kubeadm`, `kube-apiserver` is installed as a pod. We can see its options within the pod definitions `cat /etc/kubernetes/manifests/kube-apiserver.yaml`

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/6afc06fe-8e9b-4b53-84e2-9148620cb602/6a881511-fe30-4a2f-9f07-e7e79840381f/image.png)

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/6afc06fe-8e9b-4b53-84e2-9148620cb602/9dc7eb83-2a0f-45e7-accb-7352a9e35f2c/image.png)

- But if installed k8s manually, we should also install `kube-apiserver` manually. Then we can find its options in `cat /etc/systemd/system/kube-apiserver.service`

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/6afc06fe-8e9b-4b53-84e2-9148620cb602/1997c81b-ee5c-4731-93aa-d1870334eafa/image.png)

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

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/6afc06fe-8e9b-4b53-84e2-9148620cb602/5148710e-6b22-4cf8-904c-13096ba73bba/image.png)

### Kube Controller Manager

- Every intelligence in k8s sits inside `Kube Controller Manager`. It has lots of controllers including the one in picture below. They’re all enabled by default when we install Kube Controller Manager, but we can disable any of them

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/6afc06fe-8e9b-4b53-84e2-9148620cb602/7c8ebb74-3ba4-41d8-b815-f90274e732d8/image.png)

- Controllers are like officers in master ship. Each Controller is responsible for a set of things in workers. One officers is responsible whenever worker ships comes and leave, so on.
So, in kubernetes, Controller is responsible to make sure status of different components are in desired status (**Watch Status, Remediate Situation)**. Eg:
    - Node Controller, checks the status of nodes every 5 sec (is changeable), if is unhealthy, if stays unhealthy for 40 seconds marks is at unhealthy. If it’s unhealthy for 5 minutes, it removes pods assigned to that Node and assign pods to a healthy Node.
    - Replication Controller, makes sure the desired number of pods are available.
- `Kube Controller manager` will be installed automatically if we use `kubeadm` . If we installed K8S manually, we should install Controller Manager as well.
- Its option is located in `controller-manager-master`  pod if we used `kubeadm` or in service file if we installed manually.
    
    ![When we install k8s and Controller manager manually](https://prod-files-secure.s3.us-west-2.amazonaws.com/6afc06fe-8e9b-4b53-84e2-9148620cb602/a03df4a4-81e4-4ba3-8cfc-680c2e1e0e6f/image.png)
    
    When we install k8s and Controller manager manually
    

![1. When install using kubeadm](https://prod-files-secure.s3.us-west-2.amazonaws.com/6afc06fe-8e9b-4b53-84e2-9148620cb602/e367bd24-43fa-4267-a331-b0424d6c8558/image.png)

1. When install using kubeadm

![2. To see options in k8s pod](https://prod-files-secure.s3.us-west-2.amazonaws.com/6afc06fe-8e9b-4b53-84e2-9148620cb602/a81abbf1-5725-40a9-be10-64f26a16dc1a/image.png)

2. To see options in k8s pod

- To see running `controller-manager` processes, run the following command on master Node: `ps -aux | grep kube-controller-manager`

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

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/6afc06fe-8e9b-4b53-84e2-9148620cb602/29cc1025-f694-4fce-ac0c-dedd1b683a80/image.png)

- You can install `kube-proxy` manually if you installed k8s manually. Or kubeadm will deploy it as Pod

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/6afc06fe-8e9b-4b53-84e2-9148620cb602/e48b82db-258d-46a2-8b4c-0801dc8a7285/image.png)

## Kubenetes Pods

- Pods are the smallest object that you can create on Kubernetes
- Container should be in Pod, it can’t be standalone
- For **scaling purpose, we shouldn’t deploy another container instance in the same Pod**. Instead, it should be a Pod with the new instance.
If our Node doesn’t have enough required capacity for adding more instances, we can add Pod to the next Nodes.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/6afc06fe-8e9b-4b53-84e2-9148620cb602/cc55f9af-e7c2-48b5-b5de-9e6e1b4d5d4c/image.png)

- Sometimes (rarely) we may need to have more than 1 instance in a Pod, like when our main app needs a helper Sqlite DB for its quick operations and that DB isn’t needed to be accessible for other instances or other services. These instances are in the same Pod, and their network is local and isolated. Also, they have access to each other’s storage.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/6afc06fe-8e9b-4b53-84e2-9148620cb602/985e7c30-d7d4-4731-8647-c7600e438521/image.png)

- K8s considers all instances inside a Pod as one object. It shares volumes and volumes of Pod’s instances to each other automatically, it maps them to each other, so on. So, it removes, create the whole Pod completely. It means, for example when the helper DB or the worker app (which sits in the same Pod) gets unhealthy, k8s will kill not only the worker app in that Pod, but even helper DB instance
- For creating Pods, we run the following cmd: `kubectl run nginx --image nginx` .
    - k8s will create Pod for us, and gets nginx image from Docker hub, and create instance inside that Pod.
    - In this example, `--image nginx` means from public Docker hub, but we also can define to get image from private repo
- For listing running Pods: `kubectl get pods` or `kubectl get pods -o wide` for detailed info
    - By default, the created Pod won’t be accessible for end users, but we can access to the instance using k8s itself.
- On `kubectl get pods` command, READY column is: `running containers in pod/total containers in pod`
- To see spec of running Pod: `kubectl describe pod myapp-pod`
- For deleting pods: `kubectl delete pods mywebapp`

### K8s YAML files

- Kubernetes uses yaml files as inputs for creation of objects like Pods, replicas, services, deployments, etc. It always have the 4 top level fields: `apiVersion, kind, metadata, spec`

```yaml
// Take care of indents. Siblings should be in the same
   level of indents, and child of parent should have more
   indent compared to parent
apiVersion: v1 //refer to table
kind: Pod
metadata:
// all key-values inside metadata should be in k8s defined
	list, like name, labales, so on
	name: myapp-pod 
	labels:
	// but keys inside labels can be anything
		app: myapp
		// We can use such fields to different purpose, like filterring:
		type: frontend 
spec:
	containers:
	// for each item of list, we use '-':
		- name: nginx-container
			image: nginx
```

- Once you created the config file, you can create Pod using `kubectl create -f pod-definition.yml` or `kubectl apply -f pod-definition.yml`
- 

| Kind | Version |
| --- | --- |
| Pod | v1 |
| Service | v1 |
| ReplicaSet | apps/v1 |
| Deployment | apps/v1 |

### Exam notes

- k8s in exam has been installed using `kubeadm`  which
    - already deployed etcd, Kube-Apiserver, Kube-Scheduler, Kube-Controller-Manager as Pods