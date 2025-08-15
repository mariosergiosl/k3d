# k3s Podman

## Configuração de Cluster K3s no Podman para Ambiente de Laboratório

### Descrição do Ambiente Inicial
O ambiente de laboratório é uma máquina virtual rodando openSUSE 15.6. Para a criação do cluster K3s, foi utilizado o Podman, um gerenciador de contêineres OCI.

---

### 1. Preparação da VM e do Podman
Certifique-se de que a sua máquina virtual está com o Podman instalado. 
``` Bash
zypper install podman
```

Instalar o pacote podman-docker: Este pacote é crucial, pois contém o binário catatonit que o k3d (e o Podman em modo de compatibilidade) precisa. Sem ele, a criação de contêineres falha.
``` Bash
zypper install podman-docker
```

O processo de instalação do k3d requer compatibilidade com a API do Docker, então o primeiro passo é habilitá-la e criar um link simbólico para que o k3d consiga se comunicar com o Podman. Habilitar a API do Podman: Para permitir que ferramentas como o k3d e o Portainer se comuniquem com o Podman usando a API do Docker, habilite o socket e crie um link simbólico:
**Habilite o socket do Podman:**
``` Bash
systemctl enable --now podman.socket
```

**Crie um link simbólico para o socket. Isso faz com que o Podman responda às requisições da API do Docker.**
``` Bash
ln -s /run/podman/podman.sock /var/run/docker.sock
```

**Crie o diretório para armazenar os arquivos de configuração do K3s, evitando conflito com outras instalações.**
``` Bash
mkdir -p /opt/k3d
```

---


### 3. Criação e Configuração do Cluster K3s
Este é o comando que cria o contêiner do K3s no modo de rede host, o que garante a conectividade, e salva os arquivos de configuração em um diretório dedicado.
**Rode o contêiner K3s no modo de rede host, mapeando o diretório de dados para /opt/k3d e incluindo o IP da sua VM no certificado TLS para resolver o problema de handshake.**
``` Bash
podman run --network=host --dns 8.8.8.8 -d -v /opt/k3s:/etc/rancher/k3s --privileged --name=k3s-server rancher/k3s:v1.28.1-k3s1 server --tls-san=192.168.56.104
```

**Copie o arquivo kubeconfig para um local de fácil acesso:**
``` Bash
cp /opt/k3d/k3s.yaml /opt/k3d/kubeconfig
````

**Edite o arquivo kubeconfig e altere o endereço do servidor de 127.0.0.1 para o IP da sua VM:**
``` Bash
vim /opt/k3d/kubeconfig
```
(ou use outro editor de texto como o nano)

Substitua a linha server: https://127.0.0.1:6443 por server: https://[seu-nome-ou-ip]:6443.

---

### 4. Verificação da Conexão
Agora, use o kubectl para confirmar que tudo está funcionando perfeitamente.
**Verifique a conexão e o status do cluster:**
``` Bash
kubectl cluster-info --kubeconfig /opt/k3d/kubeconfig
```

**Liste os nós para confirmar que o cluster está pronto:**
``` Bash
kubectl get nodes --kubeconfig /opt/k3d/kubeconfig
```

### 5. Rancher

``` Bash
podman run -d --restart=unless-stopped --privileged -e CATTLE_INSECURE_TRANSPORT=true -p 8026:80 -p 4426:443 -p 6444:6443 -p 9326:9345 -p 8426:8472 -v /opt/rancher26:/var/lib/rancher --name rancher_v2_6 rancher/rancher:v2.6.12
```

### 6. Portainer


curl --insecure -sfL https://192.168.56.104:4426/v3/import/jnqqk9dtgs2sv5mhnn6hbch5fqt28bm8bghd859gcztdqftkjgtjw8_c-m-bwpp8qmj.yaml | kubectl apply -f - --kubeconfig /opt/k3s/k3s.yaml
