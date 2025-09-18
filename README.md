## Pré-requisitos:

- Hyper-v (neste caso específico, vSphere ([[ESXi]] com [[vCenter]])) com pelo menos, 32GB de RAM, prefencialmente 128GB de RAM
- Ubuntu 24.04 Server ISO
- Acesso à internet
- Paciência, perseverança e vontade de aprender

## 01 - Configurando o Sistema Operacional Template

O primeiro passo é instalar ubuntu 24.04 Server em uma VM. 
As especificações são: 

| Hardware  | Mínimo | Aceitável | Recomendado |
| --------- | ------ | --------- | ----------- |
| vCPU      | 2      | 4         | 8           |
| RAM (GB)  | 4      | 8         | 16          |
| Disk (GB) | 16     | 20        | 20+         |

Não esquecer de, durante a instalação, ativar o servidor de SSH.

#### Após a instalação concluída, atualizar o sistema e instalar os pacotes necessários.

atualização do sistema: 
```shell
sudo apt update && sudo apt upgrade -y
```

Instalação dos pacotes necessários: 
```shell
sudo apt install -y apt-transport-https gnupg ca-certificates curl software-properties-common inetutils-traceroute
```

### Desativar o ipv6 (opcional)

Editar o arquivo sysctl.conf
```shell
sudo vim /etc/sysctl.conf
```

E adicionar as seguintes linhas no final do arquivo:
```shell
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
net.ipv6.conf.lo.disable_ipv6=1
```

Para manter o ipv6 desativado após reboot, criar o arquivo rc.local
```shell
sudo vim /etc/rc.local
```

E adicionar o conteúdo conforme abaixo:
```shell
#!/bin/bash
# /etc/rc.local

/sbin/sysctl -p/etc/sysctl.conf
/etc/init.d/procps restart

exit 0
```

Depois tornar o rc.local executável
```shell
sudo chmod 755 /etc/rc.local
```

### Instalação do Docker

Validação de que não há pacotes conflitantes antes de instalar o docker: 
```shell
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

Configuração do repositório Docker: 
```shell
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

Instalação da última versão do Docker:
```shell
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Adicionar o usuário local ao grupo Docker:
```shell
sudo usermod -aG docker $USER
```

### Instalação do Kubernetes

#### Preparação do Sistema

Desativar o SWAP: 
```shell
sudo swapoff -a
```

Ativar os módulos no kernel:
```shell
sudo modprobe overlay
sudo modprobe br_netfilter
```

Adicionar os módulos no k8s.conf:
```shell
sudo vim /etc/modules-load.d/k8s.conf
```

Adicionar as seguintes linhas:
```shell
overlay
br_netfilter
```

É necessária a permissão de IP Forwarding no kubernetes.

Editar o sysctl no k8s.conf
```shell
vim /etc/sysctl.d/k8s.conf
```

Adicionar o seguinte parâmetro:
```shell
net.ipv4.ip_forward=1
```

### Criar uma configuração padrão para o *containerd*

É necessário criar um arquivo *toml* de configuração para editá-lo.

Criar o arquivo:
```shell
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null 2>&1
```

Editar o arquivo:
```shell
sudo vim /etc/containerd/config.toml
```

Editar o conteúdo do arquivo e alterar o valor do **SystemdCgroup** de *false* para *true*
``SystemdCgroup = true``

#### Instalação dos pacotes do Kubernetes

Baixar a chave pública do repositório do kubernetes:
```shell
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg # allow unprivileged APT programs to read this keyring
```

Adicionar o repositório:
```shell
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list   # helps tools such as command-not-found to work correctly
```

Atualizar o indexador de pacotes e instalar as ferramentas do kubernetes:
```shell
sudo apt-get update
sudo apt-get install -y kubeadm kubectl kubelet
```

Não permitir a atualização dos pacotes do kubernetes sem explícita solicitação:
```shell
sudo apt-mark hold kubeadm kubectl kubelet
```

---
### Instalação do Helm

[[Kubernetes LAB - Helm]]
#### Passo 1: Instalar o Helm CLI

O Helm consiste em duas partes: o **CLI** (client) e o **Tiller** (servidor, nas versões antigas). A partir do Helm 3, o Tiller foi removido, e o Helm funciona apenas com o CLI, o que simplifica a instalação.

#### No Linux:
1. Baixe o binário do Helm:
   ```bash
   curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
   ```

2. Execute o script de instalação:
   ```bash
   chmod 700 get_helm.sh
   ./get_helm.sh
   ```

3. Verifique a instalação:
   ```bash
   helm version
   ```
   Você verá algo como:
   ```
   version.BuildInfo{Version:"v3.12.0", GitCommit:"..."}
   ```

---

#### Passo 2: Configurar Repositórios do Helm

O Helm usa repositórios para buscar charts. O repositório padrão é o **Bitnami**, mas você pode adicionar outros conforme necessário.

1. Adicione o repositório Bitnami:
   ```bash
   helm repo add bitnami https://charts.bitnami.com/bitnami
   ```

2. Adicione o repositório oficial do Helm (stable):
   ```bash
   helm repo add stable https://charts.helm.sh/stable
   ```

3. Atualize a lista de repositórios:
   ```bash
   helm repo update
   ```

4. Liste os repositórios configurados:
   ```bash
   helm repo list
   ```
   Saída esperada:
   ```
   NAME    URL
   bitnami https://charts.bitnami.com/bitnami
   stable  https://charts.helm.sh/stable
   ```

---

## 02 - VM Template

Após a instalação e configuração do Docker e do Kubernetes, é boa prática, converter a VM em template. Isso facilita a criação das novas VMs, poupando tempo de instalação e configuração. Caso o hyper-v não permita a criação / conversão de template, use o modo de clonagem para clonar a VM.

## 03 - Clonagem / *Deploy* VM

Criar seis novas VMs a partir da matriz (template).
3 VMs atuarão como *Controllers* e as outras três, como *Workers*. 
A nomenclatura e endereçamento IP abaixo foram utilizados:
*Obs: Você pode reduzir os números de controladores e trabalhadores (mínimo 2 de cada) para salvar recursos.*

|   **hostname**    | **IP Address** |
| :---------------: | :------------: |
| k8s-controller-01 |  192.168.0.61  |
| k8s-controller-02 |  192.168.0.62  |
| k8s-controller-03 |  192.168.0.63  |
|   k8s-worker-01   |  192.168.0.71  |
|   k8s-worker-02   |  192.168.0.72  |
|   k8s-worker-03   |  192.168.0.73  |

## 04 - Configurar rede em cada VM

### Endereçamento IP

#### Opção 1 - Configurar reserva de IP no DHCP

Acessar o roteador ([[FritzBox]]) / servidor de DHCP e criar reservas para cada host através do MAC address.

#### Opção 2 - Configurar IP estático

Editar o netplan: 
```shell
sudo vim /etc/netplan/50-cloud-init.yaml
```

E substituir o conteúdo do arquivo pelo código abaixo. <font color=red>Não esquecer de informar o endereço ipv4 correto! </font>
```shell
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      addresses:
        [192.168.0.61/24]
      routes:
        - to: default
          via: 192.168.0.1
          metric: 100
      nameservers:
        search: [localdomain, local]
        addresses:  [192.168.0.1]
      dhcp6: false
      dhcp4: false
```

### Configurar o hostname

Executar o comando hostnamectl conforme abaixo. <font color=red>Não esquecer de informar o hostname correto! </font>
```shell
sudo hostnamectl set-hostname c1-controller-01
```

## 05 - *Snapshot*

Caso o hyper-v permita, faça um snapshot de cada uma das VMs criadas. Isto evita retrabalho em caso da VM for perdida.

--------

## 06 - Inicializar o Cluster Kubernetes

O primeiro passo é inicializar o cluster no **controlador principal** (`c1-controller-01`).

### Passo 1: Inicializar o cluster no controlador principal

No **`c1-controller-01`**, execute:
```bash
sudo kubeadm init --control-plane-endpoint c1-controller-01 --upload-certs --pod-network-cidr=192.168.0.0/16
```

- Se quiser especificar uma rede diferente, adicione `--pod-network-cidr=10.244.0.0/16` (Flannel).
- O `--upload-certs` permite adicionar outros control planes posteriormente.

Ao final, o `kubeadm` mostrará um **token de join** para adicionar workers e nós de controle.

### Passo 2: Configurar `kubectl` no controlador principal

Ainda no **`c1-controller-01`**, configurar o `kubectl`:

```bash
mkdir -p $HOME/.kube \
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config \
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Agora, você pode rodar `kubectl get nodes` para verificar o estado do cluster.
Para isso, execute os comandos como usário comum:
```bash
kubectl get nodes
```
  
### Passo 3: Adicionar os Workers ao Cluster

Nos **três workers** (`c1-worker-01`, `c1-worker-02`, `c1-worker-03`), execute o `kubeadm join` sem `--control-plane`:
```bash
sudo kubeadm join c1-controller-01:6443 --token <TOKEN> \     --discovery-token-ca-cert-hash sha256:<HASH>
```

Confirmar a entrada deles com:

```bash
kubectl get nodes
```

Porém, como a rede foi inicializada, os nós devem aparecer com `NotReady`.

Confirmar o motivo do status não estar pronto com o comando:

```bash
kubectl describe nodes
```
O comando acima listará os nodes. <font color=orange>Procurar pela linha:</font>
`container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized`

Caso positivo, basta inicializar o Calico, no <font color=cyan>item 07.</font>
## 07 - Instalar o Calico

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/custom-resources.yaml
```

Confirmar se os nodes estão prontos:

```bash
kubectl get nodes
```

Obs: Pode demorar algum tempo para que os nodes estejam prontos (`Ready`)

Após instalar, verificar se os pods do Calico estão rodando corretamente:

```bash
kubectl get pods -n calico-system
```

Todos os pods devem aparecer com status **Running** ou **Completed**.

## 08 - Instalar o MetalLB

Executar o comando para instalar o MetalLB:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/main/config/manifests/metallb-native.yaml
```

<font color=orange>Nota:</font> **deve ser executado apenas uma vez em qualquer um dos controladores**, pois ele cria os recursos no cluster inteiro e o Kubernetes se encarrega de distribuir os pods do MetalLB nos nós necessários.

Como o MetalLB roda como um conjunto de pods dentro do namespace `metallb-system`, ele será gerenciado pelo próprio Kubernetes, independentemente de qual nó executou o comando.

Depois, configurar o MetalLB para fornecer um intervalo de IPs:
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.0.220-192.168.0.230  # Altere para um intervalo válido na sua rede
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advert
  namespace: metallb-system
```

Aplicar as configurações:
```bash
  kubectl apply -f metallb-config.yaml
```

## <font color=blue> Explicação da Ordem </font>

1. **Calico primeiro** (ou outro CNI)
    
    - O **Calico** deve ser instalado primeiro, pois ele configura a rede do cluster Kubernetes.
    - Sem um **CNI (Container Network Interface)**, os nós e pods não conseguem se comunicar corretamente.
      
2. **Ingress NGINX em segundo**
    
    - Depois de garantir que o cluster tem conectividade interna, instalamos o **Ingress NGINX Controller** para gerenciar o tráfego HTTP/HTTPS dentro do cluster.    
    - O Ingress NGINX precisa de um **Service LoadBalancer** para expor as aplicações externamente, então ele será usado junto com o MetalLB.
      
3. **MetalLB por último**
    
    - O **MetalLB** deve ser instalado depois do Ingress NGINX, pois ele fornece IPs para os serviços `LoadBalancer`.
    - Se instalado antes, alguns serviços podem não funcionar corretamente por falta de endpoints válidos.


--------

## 09 - Testar o cluster

### Passo 1: Criar um Service para testar o MetalLB

O MetalLB fornece **endereços IP externos** para os serviços do Kubernetes, então vamos criar um **Service do tipo LoadBalancer** para testar.

Crie um arquivo chamado `nginx-lb.yaml` com o seguinte conteúdo:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-lb
  labels:
    app: nginx-lb
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-lb
  template:
    metadata:
      labels:
        app: nginx-lb
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb-service
spec:
  selector:
    app: nginx-lb
  type: LoadBalancer  # Aqui o MetalLB atribuirá um IP externo
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Agora, aplicar esse arquivo:

```shell
kubectl apply -f nginx-lb.yaml
```

### Passo 2: Verificar se o MetalLB atribuiu um IP

Após alguns segundos, rodar:

```bash
kubectl get svc
```

Você verá algo assim:

```nginx
NAME             TYPE          CLUSTER-IP       EXTERNAL-IP    PORT(S)  AGE 
kubernetes       ClusterIP     10.96.0.1        <none>         443/TCP  20m 
nginx-lb-service LoadBalancer  10.102.187.100   192.168.0.231  80/TCP   5s
```

🔹 O **EXTERNAL-IP** (`192.168.0.231` no exemplo) é o IP que o **MetalLB atribuiu** ao serviço.  
🔹 Esse IP deve estar dentro do intervalo que você definiu no arquivo de configuração do MetalLB.

### Passo 3: Testar o acesso

Agora, teste se o serviço está acessível de outro computador na rede. No navegador ou com `curl`, acesse:
```bash
curl http://192.168.0.231
```

Se tudo estiver certo, você verá a página padrão do **NGINX**!



## Fontes: 
https://youtu.be/vX2n05t0AQg

[Ubuntu | Docker Docs](https://docs.docker.com/engine/install/ubuntu/)

[How to Disable IPv6 on Ubuntu Linux](https://itsfoss.com/disable-ipv6-ubuntu-linux/)

[server - How to put a IP static in Ubuntu 24.04 - Ask Ubuntu](https://askubuntu.com/questions/1532356/how-to-put-a-ip-static-in-ubuntu-24-04)
