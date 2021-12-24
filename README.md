# NGINX Instance Manager - Docker image

## Description

This repo creates a docker image for NGINX Instance Manager 2.x (NIM, https://docs.nginx.com/nginx-instance-manager/) to run it on Kubernetes/Openshift.
The image can optionally be built with NGINX Instance Counter support (see https://github.com/fabriziofiorucci/NGINX-InstanceCounter)

## Prerequisites

- Kubernetes/Openshift cluster
- NGINX Ingress Controller with `VirtualServer` CRD support (see https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/)
- Access to F5/NGINX downloads to fetch NIM 2.x installation .deb file

## How to build

1. Clone this repo
2. Download NIM 2.x .deb installation file for Ubuntu 20.04 "focal_amd64" (ie. nms-instance-manager_2.0.0-433676695~focal_amd64.deb) and copy it into nim-files/
3. Build NIM Docker image using:

```
./scripts/buildNIM.sh [NIM_DEBFILE] [target Docker image name] [counter enabled (true|false)] [auth username] [auth password]

for instance:

./scripts/buildNIM.sh ./nim-files/nms-instance-manager_2.0.0-433676695~focal_amd64.deb your.registry.tld/nginx-nim2:tag true admin myNimPassword
```

this builds the image and pushes it to a private registry. The last parameter (to be set to either "true" or "false") specifies if NGINX Instance Counter (https://github.com/fabriziofiorucci/NGINX-InstanceCounter) shall be included in the image being built

4. Edit manifests/0.nginx-nim.yaml and specify the correct image by modifying the "image" line. Additionally modify the "env:" section if you need NGINX Instance Counter to push instances data to a remote collector

```
image: your.registry.tld/nginx-nim2:tag
```

5. If the instance counter was built in the image, configure the relevant environment variables, see the documentation at https://github.com/fabriziofiorucci/NGINX-InstanceCounter#for-kubernetesopenshift-1

```
        env:
          ### Instance counter Push mode
          - name: STATS_PUSH_ENABLE
            #value: "true"
            value: "false"
          - name: STATS_PUSH_MODE
            value: CUSTOM
            #value: NGINX_PUSH
          - name: STATS_PUSH_URL
            value: "http://192.168.1.5/callHome"
            #value: "http://pushgateway.nginx.ff.lan"
          ### Push interval in seconds
          - name: STATS_PUSH_INTERVAL
            value: "10"
```

6. Check / modify files in `/manifests/certs` to customize the TLS certificate and key used for TLS offload

7. Start and stop using

```
./scripts/nimDockerStart.sh start
./scripts/nimDockerStart.sh stop
```

8. After starting NIM it will be accessible at:

```
NIM GUI: https://nim2.f5.ff.lan
NIM gRPC port: nim2.f5.ff.lan:30443

Instance counter REST API (if enabled at build time - see the documentation at https://github.com/fabriziofiorucci/NGINX-InstanceCounter):
- https://nim2.f5.ff.lan/counter/instances
- https://nim2.f5.ff.lan/counter/metrics
- Push mode (configured through env variables in manifests/0.nginx-nim.yaml)
```

9. After installing the nginx-agent on NGINX Instances to be managed with NIM, update the file `/etc/nginx-agent/nginx-agent.conf` and modify the line:

```
grpcPort: 443
```

into:

```
grpcPort: 30443
```

and then restart nginx-agent


## Tested NIM releases

This repo has been tested with NIM 2.0

# Example

## Docker image build

```
$ ./scripts/buildNIM.sh nim-files/nms-instance-manager_2.0.0-433676695~focal_amd64.deb registry.ff.lan:31005/nim2-docker:1.0 true admin nim
==> Building NIM docker image
Sending build context to Docker daemon  54.31MB
Step 1/39 : FROM ubuntu:20.04
 ---> ba6acccedd29
Step 2/39 : ARG NIM_DEBFILE
 ---> Running in db29f245b5a1
Removing intermediate container db29f245b5a1
 ---> e1bf097beff2
[...]
 ---> 22ecf8466795
Successfully built 22ecf8466795
Successfully tagged registry.ff.lan:31005/nim2-docker:1.0
The push refers to repository [registry.ff.lan:31005/nim2-docker]
[...]
1.6: digest: sha256:74d81dfcfd63929d04b1243d345d4e6f3d66b772805e74204f0e03bca885b218 size: 7008
```

## Starting NIM

```
$ ./scripts/nimDockerStart.sh start
namespace/nginx-nim2 created
Generating a RSA private key
...................+++++
...............................+++++
writing new private key to 'nim2.f5.ff.lan.key'
-----
secret/nim2.f5.ff.lan created
deployment.apps/nginx-nim2 created
service/nginx-nim2 created
service/nginx-nim2-grpc created 
virtualserver.k8s.nginx.org/vs-nim2 created

$ kubectl get pods -n nginx-nim2 -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP             NODE       NOMINATED NODE   READINESS GATES
nginx-nim2-67d5b48b7d-f76hf   1/1     Running   0          32s   10.244.1.203   f5-node1   <none>           <none>
```

NIM GUI is now reachable at:
- Web GUI: `https://nim2.f5.ff.lan`
- gRPC: `nim2.f5.ff.lan:30443`
- Instance counter: `https://nim2.f5.ff.lan/counter/instances` and `https://nim2.f5.ff.lan/counter/metrics` and push mode

## Stopping NIM

```
$ ./scripts/nimDockerStart.sh stop
namespace "nginx-nim2" deleted
```