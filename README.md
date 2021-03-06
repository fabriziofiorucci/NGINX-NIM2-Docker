# NGINX Instance Manager 2 - Docker image

This repository is archived. To run NGINX Instance Manager 2 as a container please refer to the official installation docs at https://docs.nginx.com/nginx-instance-manager/

## Description

This repo creates a docker image for NGINX Instance Manager >= 2.1 (https://docs.nginx.com/nginx-instance-manager/) to run it on Kubernetes/Openshift.
The image can optionally be built with F5 Telemetry Tracker support (see https://github.com/fabriziofiorucci/F5-Telemetry-Tracker)

## Prerequisites

- Kubernetes/Openshift cluster with dynamic storage provisioner enabled: see the [example](/contrib/pvc-provisioner)
- NGINX Ingress Controller with `VirtualServer` CRD support (see https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/)
- Access to F5/NGINX downloads to fetch NGINX Instance Manager >= 2.1 installation .deb file
- Linux host running Docker to build the image

## How to build

1. Clone this repo
2. Download NGINX Instance Manager >= 2.1 .deb installation file for Ubuntu 20.04 "focal_amd64" (ie. `nms-instance-manager_2.1.0-466409269~focal_amd64.deb`) and copy it into `nim-files/`
3. Build NGINX Instance Manager Docker image using:

```
./scripts/buildNIM.sh [NIM_DEBFILE] [target Docker image name] [F5 Telemetry Tracker enabled (true|false)]

for instance:

./scripts/buildNIM.sh ./nim-files/nms-instance-manager_2.1.0-466409269~focal_amd64.deb your.registry.tld/nginx-nim2:tag true
```

this builds the image and pushes it to a private registry. The "F5 Telemetry Tracker enabled" parameter (to be set to either "true" or "false") specifies if F5 Telemetry Tracker (https://github.com/fabriziofiorucci/F5-Telemetry-Tracker) shall be included in the image being built

4. Edit `manifests/1.nginx-nim.yaml` and specify the correct image by modifying the "image" line and configure NGINX Instance Manager username, password and the base64-encoded license file for automated license activation:

```
image: your.registry.tld/nginx-nim2:tag
[...]
env:
  ### NGINX Instance Manager environment
  - name: NIM_USERNAME
    value: admin
  - name: NIM_PASSWORD
    value: nimadmin
  - name: NIM_LICENSE
    value: "<BASE64_ENCODED_LICENSE_FILE>"
```

To base64-encode the license file the following command can be used:

```
base64 -w0 NIM_LICENSE_FILENAME.lic
```

Additionally, parameters user by NGINX Instance Manager to connect to ClickHouse can be configured:

```
env:
  [...]
  - name: NIM_CLICKHOUSE_ADDRESS
    value: clickhouse
  - name: NIM_CLICKHOUSE_PORT
    value: "9000"
  ### If username is not set to "default", the clickhouse-users ConfigMap in 0.clickhouse.yaml shall be updated accordingly
  - name: NIM_CLICKHOUSE_USERNAME
    value: "default"
  ### If password is not set to "NGINXr0cks", the clickhouse-users ConfigMap in 0.clickhouse.yaml shall be updated accordingly
  - name: NIM_CLICKHOUSE_PASSWORD
    value: "NGINXr0cks"
```

5. If F5 Telemetry Tracker was built in the image, configure the relevant environment variables. See the documentation at https://github.com/fabriziofiorucci/F5-Telemetry-Tracker#for-kubernetesopenshift-1

```
env:
  ### F5 Telemetry Tracker Push mode
  - name: STATS_PUSH_ENABLE
    #value: "true"
    value: "false"
  - name: STATS_PUSH_MODE
    value: CUSTOM
    #value: PUSHGATEWAY
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

8. After starting NGINX Instance Manager it will be accessible from outside the cluster at:

NGINX Instance Manager GUI: `https://nim2.f5.ff.lan`
NGINX Instance Manager gRPC port: `nim2.f5.ff.lan:30443`

and from inside the cluster at:

NGINX Instance Manager GUI: `https://nginx-nim2.nginx-nim2`
NGINX Instance Manager gRPC port: `nginx-nim2.nginx-nim2:443`


F5 Telemetry Tracker REST API (if enabled at build time - see the documentation at `https://github.com/fabriziofiorucci/F5-Telemetry-Tracker`):
- `https://nim2.f5.ff.lan/f5tt/instances`
- `https://nim2.f5.ff.lan/f5tt/metrics`
- Push mode (configured through env variables in `manifests/1.nginx-nim.yaml`)

Grafana dashboard: `https://grafana.nim2.f5.ff.lan` - see [configuration details](/contrib/grafana)

Running pods are:

```
$ kubectl get pods -n nginx-nim2 -o wide
NAME                          READY   STATUS    RESTARTS   AGE    IP            NODE       NOMINATED NODE   READINESS GATES
clickhouse-7bc96d6d56-jthtf   1/1     Running   0          5m8s   10.244.1.65   f5-node1   <none>           <none>
grafana-6f58d455c7-8lk64      1/1     Running   0          5m8s   10.244.2.80   f5-node2   <none>           <none>
nginx-nim2-679987c54d-7rl6b   1/1     Running   0          5m8s   10.244.1.64   f5-node1   <none>           <none>
```

9. For NGINX Instances running on VM/bare metal only: after installing the nginx-agent on NGINX Instances to be managed with NGINX Instance Manager 2, update the file `/etc/nginx-agent/nginx-agent.conf` and modify the line:

```
grpcPort: 443
```

into:

```
grpcPort: 30443
```

and then restart nginx-agent


## Tested NGINX Instance Manager releases

This repo has been tested with NIM 2.1.0


## Additional features

- [Grafana dashboard for telemetry](/contrib/grafana)


# Example

## Docker image build

```
$ ./scripts/buildNIM.sh nim-files/nms-instance-manager_2.1.0-508672347~focal_amd64.deb registry.ff.lan:31005/nim2-docker:2.1.0 true
```

## Starting NGINX Instance Manager

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
NAME                          READY   STATUS    RESTARTS   AGE    IP            NODE       NOMINATED NODE   READINESS GATES
clickhouse-7bc96d6d56-jthtf   1/1     Running   0          5m8s   10.244.1.65   f5-node1   <none>           <none>
grafana-6f58d455c7-8lk64      1/1     Running   0          5m8s   10.244.2.80   f5-node2   <none>           <none>
nginx-nim2-679987c54d-7rl6b   1/1     Running   0          5m8s   10.244.1.64   f5-node1   <none>           <none>
```

NGINX Instance Manager GUI is now reachable from outside the cluster at:
- Web GUI: `https://nim2.f5.ff.lan`
- gRPC: `nim2.f5.ff.lan:30443`
- F5 Telemetry Tracker: see [usage](https://github.com/fabriziofiorucci/F5-Telemetry-Tracker/blob/production/USAGE.md)

## Stopping NGINX Instance Manager

```
$ ./scripts/nimDockerStart.sh stop
namespace "nginx-nim2" deleted
```
