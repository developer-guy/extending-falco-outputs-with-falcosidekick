# Description
In this demo, we are going to demonstrate how you can send Falco alerts to the NATS subjects with make use of falcosidekick.

# What is NATS ? 
NATS.io is a simple, secure and high performance open source messaging system for cloud native applications, IoT messaging, and microservices architectures.

# What is Falco ?
Falco, the open-source cloud-native runtime security project, is the de facto Kubernetes threat detection engine. Falco was created by Sysdig in 2016 and is the first runtime security project to join CNCF as an incubation-level project. Falco detects unexpected application behavior and alerts on threats at runtime.

# What is Falcosidekick ?
By default, Falco has 5 outputs for its events: stdout, file, gRPC, shell and http. Falcosidekick aims to enrich this output types, it is a simple daemon for enhancing available outputs for Falco. It takes a falco's event and forwards it to different outputs.

![falco_extend_arch](./falco_extend_archtitechture.png)

# Prerequisites
> quick note, we are going to do this demo on macOS Cataline 10.15.7, you can use brew "a package manager for macOS" to install all of the following tools.

* [Minikube](https://minikube.sigs.k8s.io) v1.17.1
* [natscli](https://github.com/nats-io/natscli) 0.0.1
* [Helm](https://helm.sh) v3.5.1

# Hands On
Lets start with creating our local cluster using Minikube.
```bash
# here is my minikube vm configurations
$ minikube config view
- cpus: 3
- memory: 8192
- vm-driver: hyperkit
$ minikube start
😄  [minikube] minikube v1.17.1 on Darwin 10.15.7
✨  Using the hyperkit driver based on user configuration
👍  Starting control plane node test in cluster test
🔥  Creating hyperkit VM (CPUs=3, Memory=8192MB, Disk=20000MB) ...
🐳  Preparing Kubernetes v1.20.2 on Docker 20.10.2 ...
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔎  Verifying Kubernetes components...
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

We are going to use Helm "a package manager for Kubernetes" tool to install falco, falcosidekick and nats.

Lets start with the Falco first.
```yaml
falco:
  jsonOutput: true
  jsonIncludeOutputProperty: true
  httpOutput:
    enabled: true
    url: "http://falcosidekick:2801/"
```
This is our configuration to set up Falco with the Falcosidekick integration properly. Once this is applied, Falco will start to send output in json format to the Falcosidekick via http.

Lets install this.
```bash
$ helm repo add falcosecurity https://falcosecurity.github.io/charts
"falcosecurity" has been added to your repositories

$ helm install falco --namespace=falco falcosecurity/falco -f falco-values.yaml
NAME: falco
LAST DEPLOYED: Mon Feb  8 20:54:19 2021
NAMESPACE: falco
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Falco agents are spinning up on each node in your cluster. After a few
seconds, they are going to start monitoring your containers looking for
security issues.


No further action should be required.


Tip:
You can easily forward Falco events to Slack, Kafka, AWS Lambda and more with falcosidekick.
Full list of outputs: https://github.com/falcosecurity/charts/falcosidekick.
You can enable its deployment with `--set falcosidekick.enabled=true` or in your values.yaml.
See: https://github.com/falcosecurity/charts/blob/master/falcosidekick/values.yaml for configuration values.
```

Verify everything is working.
```bash
$ kubectl --namespace=falco get pods
Found existing alias for "kubectl". You should use: "k"
NAME                           READY   STATUS    RESTARTS   AGE
falco-8s9bs                    1/1     Running   0          3m53s
```

Lets move on with falcosideck installation.
```yaml
config:
  nats:
    hostport: nats://nats.nats:4222
```

Lets install this too.
```bash
$ helm install falcosidekick --namespace=falco falcosecurity/falcosidekick -f falcosidekick-values.yaml
NAME: falcosidekick
LAST DEPLOYED: Mon Feb  8 21:07:40 2021
NAMESPACE: falco
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace falco -l "app.kubernetes.io/name=falcosidekick,app.kubernetes.io/instance=falcosidekick" -o jsonpath="{.items[0].metadata.name}")
  kubectl port-forward $POD_NAME 2801:2801
  echo "Visit http://127.0.0.1:2801 to use your application"
```

Verify everything is working.
```bash
$ kubectl --namespace=falco get pods
Found existing alias for "kubectl". You should use: "k"
NAME                           READY   STATUS    RESTARTS   AGE
falco-8s9bs                    1/1     Running   0          14m
falcosidekick-55b76d54-99255   1/1     Running   0          83s
```

Lets look logs too to see nats output configured properly.
```bash
$ kubectl --namespace=falco logs -f falcosidekick-55b76d54-99255
Found existing alias for "kubectl". You should use: "k"
2021/02/08 18:07:42 [INFO]  : Enabled Outputs : NATS
2021/02/08 18:07:42 [INFO]  : Falco Sidekick is up and listening on port 2801
```

The last thing that we need to do is set up NATS server to our cluster, lets do this too.
```bash
$ helm install nats --namespace=nats nats/nats --create-namespace
NAME: nats
LAST DEPLOYED: Mon Feb  8 21:16:01 2021
NAMESPACE: nats
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
You can find more information about running NATS on Kubernetes
in the NATS documentation website:

  https://docs.nats.io/nats-on-kubernetes/nats-kubernetes

NATS Box has been deployed into your cluster, you can
now use the NATS tools within the container as follows:

  kubectl exec -n nats -it nats-box -- /bin/sh -l

  nats-box:~# nats-sub test &
  nats-box:~# nats-pub test hi
  nats-box:~# nc nats 4222

Thanks for using NATS!
```

Lets verify this.
```bash
$ kubectl --namespace=nats get pods
Found existing alias for "kubectl". You should use: "k"
NAME       READY   STATUS    RESTARTS   AGE
nats-0     3/3     Running   0          109m
nats-box   1/1     Running   0          110m

```

Now, we can inspect nats-server using natscli tool. To do so, we need to enable port forwarding, lets do this.
```bash
$ kubectl --namespace=nats port-forward svc/nats 8222 &
Forwarding from 127.0.0.1:8222 -> 8222
Forwarding from [::1]:8222 -> 8222

$ kubectl --namespace=nats port-forward svc/nats 4222 &
Forwarding from 127.0.0.1:4222 -> 4222
Forwarding from [::1]:4222 -> 4222

# 4222 is for clients.
# 8222 is an HTTP management port for information reporting.
```

Now connect to it using natscli.
```bash
$ nats sub falco.notice.terminal_shell_in_container # this is the format for the subject that falcosidekick publishes alerts.
# Add new output NATS: https://github.com/falcosecurity/falcosidekick/pull/37/files
```

We are ready to go, lets create our pod and exec to it to produce alert.
```bash
$ kubectl run --generator=run-pod/v1 nginx --image=nginx
Found existing alias for "kubectl". You should use: "k"
Flag --generator has been deprecated, has no effect and will be removed in the future.
pod/nginx created
```

Once we exec to it we should see the alert as the output of "nats sub" command.
``bash
$ kubectl exec -ti nginx -- /bin/bash
root@nginx:/#
```

Look at your terminal that is running "nats sub" command, you should see similar output.
```bash
23:19:09 Subscribing on falco.notice.terminal_shell_in_container
[#1] Received on "falco.notice.terminal_shell_in_container"
{"output":"20:19:15.325119960: Notice A shell was spawned in a container with an attached terminal (user=root user_loginuid=-1 k8s.ns=default k8s.pod=nginx container=5c85dc0ee531 shell=bash parent=runc cmdline=bash terminal=34816 container_id=5c85dc0ee531 image=nginx) k8s.ns=default k8s.pod=nginx container=5c85dc0ee531","priority":"Notice","rule":"Terminal shell in container","time":"2021-02-08T20:19:15.32511996Z","output_fields":{"container.id":"5c85dc0ee531","container.image.repository":"nginx","evt.time":1612815555325120000,"k8s.ns.name":"default","k8s.pod.name":"nginx","proc.cmdline":"bash","proc.name":"bash","proc.pname":"runc","proc.tty":34816,"user.loginuid":-1,"user.name":"root"}}
```

![demo](./demo.png)

> BONUS: you can use [nats-top](https://github.com/nats-io/nats-top) tool to inspect incoming messages to the NATS server.

# References
* https://github.com/falcosecurity/falco/blob/master/rules/falco_rules.yaml
* https://sysdig.com/blog/getting-started-writing-falco-rules/
* https://docs.nats.io/nats-tools/nats_top/nats-top-tutorial
* https://github.com/nats-io/nats-box
* https://nats-io.github.io/k8s/
* https://falco.org/blog/intro-k8s-security-monitoring/
* https://falco.org/blog/extend-falco-outputs-with-falcosidekick/
