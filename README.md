# k3s-rpi

```bash
# TODO rootless k3s
sudo apt update
sudo apt upgrade -y
echo "cgroup_memory=1 cgroup_enable=memory" | sudo tee -a /boot/cmdline.txt
cat /boot/cmdline.txt
sudo reboot
curl -sfL https://get.k3s.io | sh -s - --flannel-backend none --disable-network-policy --disable=traefik --snapshotter=stargz
export KUBECONFIG=~/.kube/config
mkdir ~/.kube 2> /dev/null
sudo k3s kubectl config view --raw > "$KUBECONFIG"
chmod 600 "$KUBECONFIG"
echo "export KUBECONFIG=~/.kube/config" >> ~/.bashrc
kubectl get -A pods

https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#k8s-install-quick
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

cilium install --version 1.14.1
cilium status --wait
cilium connectivity test


# install istio
# https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/
# https://istio.io/latest/docs/tasks/traffic-management/ingress/
# https://istio.io/latest/docs/setup/additional-setup/getting-started/
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v0.8.0/experimental-install.yaml
curl -L https://istio.io/downloadIstio | sh -
export PATH="$PATH:/home/moritz/istio-1.19.0/bin"
istioctl x precheck 
cd istio-1.19.0/
istioctl install -f samples/bookinfo/demo-profile-no-gateways.yaml -y
kubectl label namespace default istio-injection=enabled
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl get services
kubectl get pods
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
kubectl apply -f samples/bookinfo/gateway-api/bookinfo-gateway.yaml
kubectl wait --for=condition=programmed gtw bookinfo-gateway
istioctl analyze




sudo nano /var/lib/rancher/k3s/server/manifests/istio.yaml

sudo nano /var/lib/rancher/k3s/server/manifests/forgejo.yaml
sudo apt install -y git
wget https://golang.org/dl/go1.21.1.linux-arm64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.21.1.linux-arm64.tar.gz
sudo nano /etc/profile
export PATH=$PATH:/usr/local/go/bin

# https://github.com/octodns/octodns
# https://github.com/AnalogJ/lexicon
# https://go-acme.github.io/lego/dns/hetzner/

sudo apt install python3-pip python3-venv
python3 -m venv env
source env/bin/activate
pip install dns-lexicon[hetzner]
LEXICON_HETZNER_TOKEN=X lexicon hetzner list selfmade4u.de A
lexicon hetzner create selfmade4u.de A --name="*.pi.selfmade4u.de" --content="192.168.2.73"
https://forgejo.pi.selfmade4u.de/
deactivate

go install github.com/go-acme/lego/v4/cmd/lego@master
HETZNER_API_KEY=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx ~/go/bin/lego --email Moritz.Hedtke@t-online.de --dns hetzner --domains *.pi.selfmade4u.de run

kubectl create secret tls pi.selfmade4u.de --namespace kube-system --cert .lego/certificates/_.pi.selfmade4u.de.crt --key .lego/certificates/_.pi.selfmade4u.de.key
```
 
