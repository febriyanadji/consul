## Kubernetes

values.yaml :
```
consul:
  global:
    name: consul
    enabled: false
    datacenter: dc1
    tls:
      enabled: false
    gossipEncryption:
      secretName: "consul-gossip-encryption-key"
      secretKey: "key"

  connectInject:
    enabled: true
    # transparentProxy:
    #   defaultEnabled: false

  server:
    enabled: false

  client:
    enabled: false
    # join: ["10.104.32.10"]
    # exposeGossipPorts: true
    # hostNetwork: true
    # dnsPolicy: ClusterFirstWithHostNet

  externalServers:
    enabled: true
    hosts: ["10.104.32.10"]
    useSystemRoots: true
  controller:
    enabled: true
  ingressGateways:
    enabled: false
    # defaults:
    #   replicas: 1
    # gateways:
    #   - name: ingress-gateway
    #     service:
    #       type: LoadBalancer
    #       ports:
    #         - port: 8080
```


## consul-server-01

consul.hcl :
```
datacenter = "dc1"

data_dir = "/opt/consul"

client_addr = "0.0.0.0"

ui_config{
  enabled = true
}

server = true

bind_addr = "0.0.0.0" # Listen on all IPv4

advertise_addr = "10.104.32.10"

bootstrap_expect=1

encrypt = "vj9P9tx4CDlKc0pjXD7iieByfR8sV91W9aBLyoxfEZo="

retry_join = ["10.104.32.10"]

ports {
  grpc = 8502
  http = 8501
}
```

## consul-client-01

consul.hcl :
```
datacenter = "dc1"

data_dir = "/opt/consul"

client_addr = "0.0.0.0"

bind_addr = "0.0.0.0" # Listen on all IPv4

advertise_addr = "10.104.32.11"

encrypt = "vj9P9tx4CDlKc0pjXD7iieByfR8sV91W9aBLyoxfEZo="

retry_join = ["10.104.32.10"]

ports {
  grpc = 8502
}
```

nginx.hcl :
```
{
    "service": {
        "name": "app3",
        "port": 80,
        "connect": {
            "sidecar_service": {
                "proxy": {
                    "upstreams": [
                        {
                            "destination_name": "app1",
                            "local_bind_port": 8085
                        },
                        {
                                "destination_name": "app2",
                                "local_bind_port": 8086
                        }
                    ]
                }
            }
        }
    }
}
```

## Applications

app1.yaml :
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
  labels:
    app: app1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
      annotations:
        consul.hashicorp.com/connect-inject: "true"
        consul.hashicorp.com/transparent-proxy: "false"
        consul.hashicorp.com/connect-service-upstreams: "app2.svc:8083, app3.svc:8084"
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: app1
  labels:
    app: app1
spec:
  selector:
    app: app1
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

## Envoy

```
consul connect envoy -sidecar-for=app3
```
