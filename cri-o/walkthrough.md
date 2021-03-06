### Build & Iinstall runc
```
git clone https://github.com/opencontainers/runc
cd runc
make && make install
```

### Build & Install crio
```
git clone https://github.com/kubernetes-incubator/cri-o.git
cd cri-o
make BUILDTAGS='seccomp'
make install install.config install.completions install.systemd
```

#### Configure crio after install
```
1) update `runc` path to "/usr/local/sbin/runc" in /etc/crio/crio.conf
2) as "signature_policy" in /etc/crio/crio.conf is empty, it will read /etc/containers/policy.json
mkdir /etc/containers
cp -p test/policy.json /etc/containers/policy.json
```

### Configure cni
```
go get -d github.com/containernetworking/plugins
cd $GOPATH/src/github.com/containernetworking/plugins
./build.sh
```

#### install cni
```
sudo mkdir -p /opt/cni/bin
sudo cp bin/* /opt/cni/bin/
```
#### config cni
```
sudo mkdir -p /etc/cni/net.d
sh -c 'cat >/etc/cni/net.d/10-mynet.conf <<-EOF
{
    "cniVersion": "0.2.0",
    "name": "mynet",
    "type": "bridge",
    "bridge": "cni0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.88.0.0/16",
        "routes": [
            { "dst": "0.0.0.0/0"  }
        ]
    }
}
EOF'

sudo sh -c 'cat >/etc/cni/net.d/99-loopback.conf <<-EOF
{
    "cniVersion": "0.2.0",
    "type": "loopback"
}
EOF'
```

### Start k8s
```
CONTAINER_RUNTIME=remote \
CONTAINER_RUNTIME_ENDPOINT='/var/run/crio.sock  --runtime-request-timeout=15m' \
./hack/local-up-cluster.sh
```

### Use [crictl](https://github.com/kubernetes-incubator/cri-tools/tree/master/cmd/crictl) as client command for cri-o
```
export CRI_RUNTIME_ENDPOINT=/var/run/crio.sock
export CRI_IMAGE_ENDPOINT=/var/run/crio.sock
crictl ps
```

### Example create sandbox and container with crictl
```
[root@dma cri-o]# crictl runs test/testdata/sandbox_config.json 
0208745f429459eb7fceddbd24d74b69a17d8499be1a745b9b3088cd6513906c
[root@dma cri-o]# crictl sandboxes
SANDBOX ID          CREATED             STATE               NAME                NAMESPACE           ATTEMPT
0208745f42945       8 seconds ago       SANDBOX_READY       podsandbox1         redhat.test.crio    1
[root@dma cri-o]# crictl create 0208745f42945 test/testdata/container_redis.json test/testdata/sandbox_config.json 
865bb587046336faab843bb22b73674993f7c284e243184a84fbb71b60f4be29
[root@dma cri-o]# crictl ps
CONTAINER ID        IMAGE               CREATED             STATE               NAME                ATTEMPT
865bb58704633       redis:alpine        3 seconds ago       CONTAINER_CREATED   podsandbox1-redis   0
[root@dma cri-o]# crictl start 865bb58704633
865bb58704633
[root@dma cri-o]# crictl ps
CONTAINER ID        IMAGE               CREATED             STATE               NAME                ATTEMPT
865bb58704633       redis:alpine        17 seconds ago      CONTAINER_RUNNING   podsandbox1-redis   0

```


### Issue to debug
1. Clean data, 
If you want clean all the container left by last time, you can just delete "storage_root" `/var/lib/containers/storage`

2. Error `Creating the pod sandbox failed: rpc error: code = 2 desc = container create failed: No help topic for 'create'`  
   solution: `your runc version is too old`
