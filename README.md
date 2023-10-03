# k3s-rpi

https://radar.cncf.io/
https://www.cncf.io/projects/
https://gateway-api.sigs.k8s.io/implementations/
https://jsonnet.org/articles/kubernetes.html

https://docs.gitlab.com/ee/user/clusters/agent/gitops/flux_tutorial.html

## Uninstall

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

#helm repo add cilium https://helm.cilium.io/
#helm install cilium cilium/cilium --version 1.15.0-pre.1 \
#  --namespace kube-system \
#  --set agent.sleepAfterInit=true --set global.cleanState=true --set global.cleanBpfState=true

sudo ip link delete cilium_host
sudo ip link delete cilium_net
sudo ip link delete cilium_vxlan
sudo iptables-save | grep -iv cilium | sudo iptables-restore
sudo ip6tables-save | grep -iv cilium | sudo ip6tables-restore

cilium uninstall --wait
/usr/local/bin/k3s-uninstall.sh

sudo rm /etc/cni/net.d/05-cilium.conflist
```

## Install

nix shell nixpkgs#kubectl nixpkgs#kubernetes-helm

```bash
ssh moritz@raspberrypi.local

# TODO rootless k3s, swap
sudo apt update
sudo apt upgrade -y

#sudo apt install ufw
#sudo ufw disable

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

echo "cgroup_memory=1 cgroup_enable=memory" | sudo tee -a /boot/cmdline.txt
cat /boot/cmdline.txt
sudo reboot
# TODO explicitly set node ips, ipv6 and ipv6 and use them later down in the document for e.g. cilium
curl -sfL https://get.k3s.io | sh -s - server --cluster-init --cluster-cidr=10.42.0.0/16,2001:cafe:42:0::/56 --service-cidr=10.43.0.0/16,2001:cafe:42:1::/112 --prefer-bundled-bin --disable-helm-controller --flannel-backend none --disable-network-policy --disable-kube-proxy --disable=traefik --snapshotter=stargz  --node-taint=node.cilium.io/agent-not-ready:NoSchedule
# this will no start with a node taint as there is no network cni plugin

export KUBECONFIG=~/.kube/config
mkdir ~/.kube 2> /dev/null
sudo k3s kubectl config view --raw > "$KUBECONFIG"
chmod 600 "$KUBECONFIG"
echo "export KUBECONFIG=~/.kube/config" >> ~/.bashrc
kubectl get -A all
# also copy this file locally to access cluster remotely

# https://docs.cilium.io/en/v1.12/gettingstarted/kind/#unable-to-contact-k8s-api-server
kubectl get nodes -o wide # should show InternalIP

# https://github.com/kubernetes-sigs/gateway-api/pull/2426/files
# https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/
# https://docs.cilium.io/en/latest/network/servicemesh/gateway-api/gateway-api/#gs-gateway-api
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v0.8.1/experimental-install.yaml

helm repo add cilium https://helm.cilium.io/

# https://docs.cilium.io/en/latest/network/kubernetes/kubeproxy-free/#kubeproxy-free
# https://medium.com/@charled.breteche/kind-cluster-with-cilium-and-no-kube-proxy-c6f4d84b5a9d

# CHECK IP
# gateway api doesnt work with a node taint so first install without it
helm upgrade --install cilium cilium/cilium --version 1.15.0-pre.1 --namespace kube-system --set k8sServiceHost=192.168.2.73 -f cilium.yaml

# helm -n kube-system get values cilium

cilium status --wait
cilium connectivity test



curl -s https://fluxcd.io/install.sh | sudo bash
. <(flux completion bash)

# fine-grained personal access token with repository access and write access to code
flux bootstrap github \
  --token-auth \
  --owner=mohe2015 \
  --repository=k3s-rpi \
  --branch=main \
  --path=clusters/rpi \
  --personal \
  --private=false




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
 
