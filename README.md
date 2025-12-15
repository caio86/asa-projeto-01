# ASA - Projeto 01: DevOps com Vagrant e Ansible

## Introdução

Este projeto automatiza a criação e configuração de um ambiente virtual de quatro nós usando Vagrant e Ansible, apresentando servidores com funções específicas e gerenciamento centralizado. Ele demonstra os princípios de Infraestrutura como Código (IaC), provisionando uma rede completa com serviços de DHCP, DNS, armazenamento de arquivos, web e banco de dados a partir de um único Vagrantfile e um playbook Ansible principal e um conjunto de roles.

**Professor:** Leonidas Francisco de Lima Junior
**Disciplina:** Administração de Sistemas Abertos
**Alunos:** Caio Luiz Lacerda Terto Silva, Gabriel Gomes Castanha Maiolo

## Visão Geral da Arquitetura

O ambiente é composto por quatro máquinas virtuais (VMs) conectadas por uma rede privada (192.168.56.0/24). Cada VM possui uma função específica, simulando uma infraestrutura de aplicação em várias camadas.

- **arq** (192.168.56.122): O servidor de infraestrutura central, fornecendo serviços de DHCP, DNS, armazenamento LVM e NFS.
- **db** (DHCP): Um servidor de banco de dados MariaDB dedicado.
- **app** (DHCP): Um servidor de aplicação web Apache.
- **cli** (DHCP): Uma estação de trabalho cliente com interface gráfica para testes.

Todas as VMs são provisionadas a partir de uma imagem base `debian/bookworm64` e configuradas por playbooks Ansible para garantir consistência e automação.

## Pilha de Tecnologia

| Componente          | Tecnologia Utilizada                 | Propósito                                                      |
| ------------------- | ------------------------------------ | -------------------------------------------------------------- |
| Virtualização       | VirtualBox, Vagrant                  | Gerenciamento do ciclo de vida da VM e isolamento do ambiente  |
| Provisionamento     | Ansible                              | Gerenciamento de configuração e serviço setup                  |
| Serviços Principais | ISC DHCP Server, BIND9, LVM, NFS     | Base para rede, armazenamento e compartilhamento de arquivos   |
| Aplicação           | MariaDB, Apache2 (HTTPD)             | Hospedagem de banco de dados e aplicação web                   |
| Ferramentas Cliente | Firefox-ESR, AutoFS, X11             | Acesso do usuário e interface gráfica                          |
| Gerenciamento       | Chrony (NTP), SSH (endurecido), Sudo | Sincronização de tempo, acesso seguro e privilégios de usuário |

## Vagrantfile

O Vagrantfile serve como o principal projeto de configuração para toda a infraestrutura. Ele define quatro máquinas virtuais com funções distintas, configurações de rede e etapas de provisionamento usando Ansible.

**Recursos Principais:**

- **Design Modular:** Cada VM (arq, db, app, cli) é definida em blocos separados com configurações específicas para a função.
- **Provisionamento em Múltiplos Estágios:** Utiliza um playbook principal (main.yml) que orquestra a execução de múltiplas roles, cada uma responsável pela configuração e provisionamento de um serviço específico.
- **Isolamento de Rede:** Cria uma rede privada (192.168.56.0/24) com endereçamento misto (estático e DHCP).
- **Gerenciamento de Recursos:** Configura CPU, memória e recursos de armazenamento conforme os requisitos da VM.
- **Provisionamento Automatizado:** Integra playbooks Ansible diretamente ao fluxo de trabalho do Vagrant.

**Detalhes da Configuração:**

- **Configurações Globais** (aplicadas a todas as VMs):

  - SO Base: `debian/bookworm64`
  - Memória: 512MB (exceto cli com 1024MB)
  - Gerenciamento de chaves SSH: Preserva chaves existentes.
  - Pastas Sincronizadas: Desabilitadas para desempenho.
  - Clones vinculados (linked clones) do VirtualBox para provisionamento mais rápido.

- **Configuração de Rede:**

  - `arq`: IP estático (192.168.56.122) - atua como servidor DHCP/DNS.
  - `db` e `app`: DHCP com endereços MAC fixos para endereçamento consistente.
  - `cli`: Cliente DHCP padrão.
  - Gerenciamento automático do servidor DHCP para evitar conflitos.

- **Configuração de Armazenamento:**
  - `arq` recebe três discos virtuais adicionais de 10GB para a configuração de armazenamento LVM.
  - As outras VMs usam a alocação de armazenamento padrão.

**Fluxo de Provisionamento:**

- **Estágio 1:** O playbook principal (main.yml) aplica configurações comuns a todas as VMs por meio de roles compartilhadas.
- **Estágio 2:** O mesmo playbook (main.yml) executa roles específicas para cada função, configurando os serviços especializados de cada VM.

## Máquinas Virtuais e Playbooks

### 1. arq - Servidor de Infraestrutura

O servidor central que hospeda serviços fundamentais de rede e armazenamento.

- **Servidor DHCP:** Configura o `isc-dhcp-server` para gerenciar a rede privada (`eth1`), atribuindo IPs a `db`, `app` e `cli`.
- **Servidor DNS:** Configura o `bind9` como um servidor autoritativo para o domínio `caio.gabriel.devops`, gerenciando pesquisas de encaminhamento (forward) e reverso (`56.168.192.db`).
- **Armazenamento (LVM):** Combina três discos virtuais de 10GB em um único Grupo de Volumes LVM (`dados`), cria um Volume Lógico de 15GB (`ifpb`), formata-o como `ext4` e o monta em `/dados`.
- **Compartilhamento de Arquivos (NFS):** Exporta o diretório `/dados/nfs` via NFS para a sub-rede privada. Usa `all_squash` para mapear todos os usuários remotos para um usuário local `nfs-ifpb` para permissões unificadas.

### 2. db - Servidor de Banco de Dados

Um servidor dedicado para operações de banco de dados.

- **Banco de Dados:** Instala e inicia o pacote `mariadb-server`.
- **Acesso a Arquivos:** Configura o AutoFS para montar automaticamente o diretório compartilhado `/dados/nfs` do servidor `arq` em `/var/nfs` sob demanda.

### 3. app - Servidor Web

Um servidor para hospedar aplicações web.

- **Servidor Web:** Instala, inicia e habilita o servidor HTTP `apache2`.
- **Conteúdo:** Implanta um arquivo `index.html` personalizado na raiz web padrão (`/var/www/html/`).
- **Acesso a Arquivos:** Configura o AutoFS para montar automaticamente o diretório NFS compartilhado de `arq` em `/var/nfs`.

### 4. cli - Estação de Trabalho Cliente

Uma máquina cliente gráfica para acesso e testes do usuário.

- **Ambiente Gráfico:** Instala `firefox-esr`, `xauth` e `x11-utils` para permitir a execução de aplicações gráficas.
- **Encaminhamento X11 (X11 Forwarding):** Configura o servidor SSH (`X11Forwarding yes`, `X11UseLocalhost no`) para permitir que aplicações GUI da VM sejam exibidas na máquina hospedeira (host).
- **Acesso a Arquivos:** Configura o AutoFS para montar automaticamente o diretório NFS compartilhado de `arq` em `/var/nfs`.

### 5. Configuração Base

Aplica configurações comuns de segurança e gerenciamento a todas as VMs.

- **Sistema:** Realiza uma atualização completa de pacotes e instala utilitários de cliente NFS.
- **Usuários e Grupos:** Cria o grupo `ifpb` (com privilégios sudo) e os usuários `caio` e `gabriel`. Gera chaves SSH para eles e implanta a chave pública do host para acesso Ansible sem senha.
- **SSH Hardening:** Configura o daemon SSH para usar apenas autenticação baseada em chave, desabilitar o login de root, restringir o acesso aos grupos `vagrant` e `ifpb` e exibir um banner legal.

## Inventário

Esta seção documenta os principais arquivos de configuração que habilitam os serviços de rede no ambiente. Estes arquivos são implantados no servidor `arq` pelo playbook Ansible e gerenciam a atribuição de DHCP e a resolução DNS para todos os nós.

### Arquivos de Configuração DHCP

**Arquivo:** `dhcp-server/files/dhcpd.conf`

Arquivo de configuração principal para o servidor ISC DHCP rodando no servidor `arq`.

- Define o servidor DHCP autoritativo para a sub-rede `192.168.56.0/24`.
- Configura durações de concessão (padrão: 180 segundos, máximo: 3600 segundos).
- Define o nome de domínio e as opções de servidor DNS para todos os clientes.
- Define um pool DHCP (`192.168.56.50-100`) para atribuição dinâmica de endereços.
- Mapeia endereços IP fixos para endereços MAC específicos para os servidores `db` (`192.168.56.132`) e `app` (`192.168.56.123`).

**Parâmetros Chave:**

- `authoritative`: O servidor é o servidor DHCP oficial para a rede.
- `range 192.168.56.50 192.168.56.100`: Pool de endereços DHCP para atribuição dinâmica.
- `hardware ethernet`: Mapeamento de MAC para IP para reservas estáticas.

### Arquivos de Configuração DNS

**Arquivo:** `dns-server/files/named.conf.options`

Configuração de opções globais para o servidor BIND9 DNS.

- Cria uma Lista de Controle de Acesso (ACL) restringindo o acesso à sub-rede `192.168.56.0/24`.
- Habilita a escuta em todas as interfaces (IPv4 e IPv6).
- Configura permissões de consulta (somente localhost e rede interna).
- Define encaminhadores (forwarders) para servidores DNS públicos (`1.1.1.1`, `8.8.8.8`) para resolução externa.
- Habilita a recursão DNS para clientes internos.

**Recursos de Segurança:**

- `allow-query`: Restringe consultas DNS apenas a clientes autorizados.
- `allow-transfer`: Limita as transferências de zona apenas para localhost.
- `dnssec-validation`: Habilita a validação DNSSEC para segurança aprimorada.

**Arquivo:** `dns-server/files/named.conf.internal-zones`

Declarações de zona para a infraestrutura de domínio interno.

- Define a zona de pesquisa de encaminhamento (forward lookup) para o domínio `caio.gabriel.devops`.
- Define a zona de pesquisa reversa (reverse lookup) para a rede `192.168.56.0/24` (`56.168.192.in-addr.arpa`).
- Configura ambas as zonas como master (autoritativas) sem atualizações dinâmicas.

**Configuração de Zona:**

- `type master`: O servidor é autoritativo para estas zonas.
- `file`: Especifica o local do arquivo de banco de dados da zona.
- `allow-update { none; }`: Desabilita atualizações dinâmicas de DNS.

**Arquivo:** `dns-server/files/caio.gabriel.devops.db`

Arquivo de banco de dados da zona de encaminhamento contendo mapeamentos de nome de host para IP.

- Define o registro Start of Authority (SOA) com número de série e parâmetros de tempo.
- Define o registro Name Server (NS) apontando para `arq.caio.gabriel.devops`.
- Cria registros A mapeando nomes de host para endereços IP:
  - `arq` → `192.168.56.122`
  - `db` → `192.168.56.132`
  - `app` → `192.168.56.123`

**Detalhes do Registro SOA:**

- `Serial`: `2025082501` (formato AAAA/MM/DDNN para rastreamento de versão)
- `Refresh`: 3600 segundos (1 hora) - intervalo de atualização do slave
- `Retry`: 1800 segundos (30 minutos) - intervalo de repetição após falha
- `Expire`: 604800 segundos (1 semana) - tempo de expiração da zona
- `Minimum TTL`: 86400 segundos (1 dia) - tempo mínimo de cache

**Arquivo:** `dns-server/files/56.168.192.db`

Arquivo de banco de dados da zona reversa contendo mapeamentos de IP para nome de host.

- Define registros SOA e NS idênticos à zona de encaminhamento.
- Cria registros PTR (Pointer) para pesquisas reversas de DNS:
  - `122` → `arq.caio.gabriel.devops`
  - `132` → `db.caio.gabriel.devops`
  - `123` → `app.caio.gabriel.devops`

**Nota sobre Registros PTR:** Os registros usam o último octeto do endereço IP (por exemplo, 122 para 192.168.56.122) no arquivo da zona reversa.

### Arquivos de Configuração do Apache

**Arquivo:** `web-server/files/index.html`

Substitui a página padrão do Apache pela descrição do projeto e pelos nomes dos membros da equipe e IDs de estudante.

## Primeiros Passos

### Pré-requisitos

Certifique-se de ter o seguinte instalado em sua máquina hospedeira (host):

- Vagrant (versão mais recente)
- VirtualBox (versão mais recente)
- Um par de chaves SSH geradas em sua máquina. A chave pública deve ser nomeada `id_ed25519.pub`.

### Implantação

1. Clone o repositório e navegue até o diretório do projeto:

```bash
git clone https://github.com/caio86/asa-projeto-01.git
cd asa-projeto-01/
```

2. Inicie o ambiente executando o seguinte comando na raiz do projeto. Isso criará e provisionará todas as quatro VMs sequencialmente:

```bash
vagrant up
```

## Acessando as VMs

### Acesso SSH

Use comandos SSH para acessar qualquer VM:

```bash
ssh caio|gabriel@192.168.56.122      # arq
ssh caio|gabriel@192.168.56.132      # db
ssh caio|gabriel@192.168.56.123      # app
ssh -X caio|gabriel@192.168.56.50    # cli - o endereço pode variar devido ao DHCP
```

### Acesso ao Servidor Web

SSH com encaminhamento X11 para a máquina `cli`:

```bash
ssh -X caio|gabriel@192.168.56.50
```

Inicie o Firefox e navegue para:

- **URL:** `http://app.caio.gabriel.devops`

### Verificação do Compartilhamento NFS

De qualquer VM cliente (`db`, `app` ou `cli`), verifique a montagem NFS:

```bash
touch /var/nfs/test
ls -l /var/nfs/          # Mostra arquivos compartilhados do servidor arq
```

### Teste de Resolução DNS

Teste a resolução DNS interna enviando ping para outras VMs pelo nome do host:

```bash
ping arq.caio.gabriel.devops
ping db.caio.gabriel.devops
ping app.caio.gabriel.devops
```
