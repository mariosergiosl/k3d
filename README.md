# k3d

## Configuração de Cluster K3s no Podman com K3d para Ambiente de Laboratório

### Descrição do Ambiente Inicial
O ambiente de laboratório é uma máquina virtual rodando openSUSE 15.6. Para a criação do cluster K3s, foi utilizado o Podman, um gerenciador de contêineres OCI.

---

### 1. Preparação da VM e do Podman
Certifique-se de que a sua máquina virtual está com o Podman instalado. 

sudo zypper install podman

Instalar o pacote podman-docker: Este pacote é crucial, pois contém o binário catatonit que o k3d (e o Podman em modo de compatibilidade) precisa. Sem ele, a criação de contêineres falha.

zypper install podman-docker

O processo de instalação do k3d requer compatibilidade com a API do Docker, então o primeiro passo é habilitá-la e criar um link simbólico para que o k3d consiga se comunicar com o Podman. Habilitar a API do Podman: Para permitir que ferramentas como o k3d e o Portainer se comuniquem com o Podman usando a API do Docker, habilite o socket e crie um link simbólico:
**Habilite o socket do Podman:**
sudo systemctl enable --now podman.socket

**Crie um link simbólico para o socket. Isso faz com que o Podman responda às requisições da API do Docker.**
sudo ln -s /run/podman/podman.sock /var/run/docker.sock

**Crie o diretório para armazenar os arquivos de configuração do K3s, evitando conflito com outras instalações.**
mkdir -p /opt/k3d

---

### 2. Remoção de Instalações Anteriores
Para garantir que não haja conflitos, remova qualquer cluster k3d ou contêiner K3s que possa ter ficado da última sessão.

**Remova o cluster k3d anterior:**
k3d cluster delete meu-cluster-k3s

**Pare e remova qualquer contêiner K3s que não tenha sido removido, e o volume associado:**
podman stop k3s-server
podman rm k3s-server
podman volume rm k3d-meu-cluster-k3s-images

---

### 3. Criação e Configuração do Cluster K3s
Este é o comando que cria o contêiner do K3s no modo de rede host, o que garante a conectividade, e salva os arquivos de configuração em um diretório dedicado.
**Rode o contêiner K3s no modo de rede host, mapeando o diretório de dados para /opt/k3d e incluindo o IP da sua VM no certificado TLS para resolver o problema de handshake.**

podman run --network=host -d -v /opt/k3d:/etc/rancher/k3s --privileged --name=k3s-server rancher/k3s:v1.28.1-k3s1 server --tls-san=192.168.56.104

**Copie o arquivo kubeconfig para um local de fácil acesso:**
cp /opt/k3d/k3s.yaml /opt/k3d/kubeconfig

**Edite o arquivo kubeconfig e altere o endereço do servidor de 127.0.0.1 para o IP da sua VM:**
vim /opt/k3d/kubeconfig
(ou use outro editor de texto como o nano)

Substitua a linha server: https://127.0.0.1:6443 por server: https://192.168.56.104:6443.

---

### 4. Verificação da Conexão

Agora, use o kubectl para confirmar que tudo está funcionando perfeitamente.
**Verifique a conexão e o status do cluster:**
kubectl cluster-info --kubeconfig /opt/k3d/kubeconfig

**Liste os nós para confirmar que o cluster está pronto:**
kubectl get nodes --kubeconfig /opt/k3d/kubeconfig

