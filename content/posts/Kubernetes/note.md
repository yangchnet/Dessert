```bash
sudo kubeadm init \
    --pod-network-cidr=192.168.0.0/16 \
    --apiserver-advertise-address=10.10.120.69 \
    --kubernetes-version=v1.23.3
```

Output:
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.10.120.69:6443 --token 8upx38.6n8hksaxoa575luv \
	--discovery-token-ca-cert-hash sha256:a38d4e2e6b826e311bdf385ba6451087e2fa237e42ddf59f2bbd115e5edc3243
```