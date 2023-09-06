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

# install istio
# https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/
# https://istio.io/latest/docs/tasks/traffic-management/ingress/
# https://istio.io/latest/docs/setup/getting-started/
# https://istio.io/latest/docs/setup/additional-setup/getting-started/
sudo nano /var/lib/rancher/k3s/server/manifests/traefik.yaml
```
 
