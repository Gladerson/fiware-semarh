Obs.: variáveis a serem alteradas:

| Exemplos                 | Para substituir          |
|--------------------------|--------------------------|
| 203.0.113.1              | [IP_PUBLICO_PFSENSE]     |
| 172.16.100.X             | [IP_VM_DEBIAN]           |
| dominio.lan              | [SEU_DOMINIO_INTERNO]    |
| chavesecreta321          | [SUA_API_KEY]            |


# Fase 1: Preparação do Servidor (Debian 13)
Estes comandos devem ser executados na sua nova VM ([IP_VM_DEBIAN]).

## 1. Atualizar o Sistema

`sudo apt update && sudo apt upgrade -y`

## 2. Instalar o Docker e o Docker Compose
Vamos usar o repositório oficial do Docker para ter a versão mais recente.

```bash
# Instalar pacotes de pré-requisito
sudo apt install -y ca-certificates curl gnupg

# Adicionar a chave GPG oficial do Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Adicionar o repositório do Docker
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar o Docker Engine e o Compose
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## 3. Adicionar seu Usuário ao Grupo Docker
Isso permite que você execute comandos docker sem usar sudo.

```bash
sudo usermod -aG docker $USER

# ATENÇÃO: Você precisa sair e logar novamente para que isso tenha efeito!
newgrp docker
```

## 4. Criar a Estrutura de Diretórios
Vamos organizar todos os nossos arquivos em uma única pasta.

```bash
# Crie o diretório principal
mkdir ~/fiware_stack
cd ~/fiware_stack

# Crie os diretórios para o NGINX e os volumes
mkdir -p nginx/certificate
mkdir -p volumes/mongo-db-fiware
mkdir -p volumes/grafana_storage
mkdir -p volumes/crate_data
```

## 5. Gerar o Certificado SSL (Wildcard)
Esta é a parte mais importante. Vamos criar um único certificado "curinga" (wildcard) que funcionará para todos os nossos subdomínios (ex: fiware-keycloak.[SEU_DOMINIO_INTERNO], fiware-grafana.[SEU_DOMINIO_INTERNO], etc.).

```bash
cd ~/fiware_stack
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout nginx/certificate/fiware.key \
-out nginx/certificate/fiware.crt \
-subj "/CN=[SEU_DOMINIO_INTERNO]" \
-addext "subjectAltName = DNS:*.[SEU_DOMINIO_INTERNO], DNS:[SEU_DOMINIO_INTERNO], IP:[IP_VM_DEBIAN]"
```

## 6. Ajustar Permissões dos Volumes
Isso garante que os containers possam escrever em suas pastas de dados.

```bash
sudo chown -R 472:472 volumes/grafana_storage/
sudo chmod -R 777 volumes/crate_data/
sudo chmod -R 777 volumes/mongo-db-fiware/
```

# Fase 2: Arquivos de Configuração (No Servidor)
Dentro da pasta `~/fiware_stack`, crie os dois arquivos a seguir.

## 1. O Arquivo docker-compose.yml
Este arquivo está muito mais limpo. O NGINX usará as portas padrão 80 e 443, e todos os serviços (como o Keycloak) estão configurados para seus próprios subdomínios.

Crie o arquivo `~/fiware_stack/docker-compose.yml`:

```bash
nano docker-compose.yml
# Cole o conteúdo
```

## 2. O Arquivo nginx.conf
Este arquivo agora é muito mais limpo e profissional. Ele usa blocos server separados para cada subdomínio.

Crie o arquivo `~/fiware_stack/nginx/nginx.conf`:

```bash
nano nginx/nginx.conf
# Cole o conteúdo
```

## 3. Iniciar a Pilha FIWARE
Agora, de dentro da pasta ~/fiware_stack, execute:

```bash
docker compose up -d
```

# Fase 3: Configuração do Cliente (Seu PC)
Fazer os subdomínios funcionarem no seu IP local. Você deve fazer isso no seu computador pessoal usado fora da rede local (o que você usa para acessar o servidor).

## 1. Encontre seu arquivo hosts
Windows: `C:\Windows\System32\drivers\etc\hosts` (Você precisa abrir o Bloco de Notas como Administrador para editá-lo).

Linux ou macOS: `/etc/hosts (Use sudo nano /etc/hosts)`.

## 2. Adicione as Linhas
Adicione as seguintes linhas no final do arquivo. Elas dizem ao seu computador para direcionar esses domínios para o IP do seu novo servidor:

```bash
# Acesso Externo ao FIWARE da SEMARH (No computador de casa/fora)
[IP_PUBLICO_PFSENSE] fiware-grafana.[SEU_DOMINIO_INTERNO]
[IP_PUBLICO_PFSENSE] fiware-keycloak.[SEU_DOMINIO_INTERNO]
[IP_PUBLICO_PFSENSE] fiware-orion.[SEU_DOMINIO_INTERNO]
[IP_PUBLICO_PFSENSE] fiware-iot.[SEU_DOMINIO_INTERNO]
[IP_PUBLICO_PFSENSE] fiware-quantumleap.[SEU_DOMINIO_INTERNO]
```

Salve o arquivo. Pode ser necessário limpar o cache DNS do seu sistema (no Windows, abra o cmd e digite `ipconfig /flushdns`).

# Fase 4: Teste os Serviços
Agora você pode acessar tudo pelas portas padrão https (443). Seu navegador dará um aviso de "Não Seguro" por causa do certificado auto-assinado. Apenas "aceite o risco" e continue.

* Grafana: `https://fiware-grafana.[SEU_DOMINIO_INTERNO]`
* Keycloak (Admin Console): `https://fiware-keycloak.[SEU_DOMINIO_INTERNO]/admin/`
* Orion (API): `https://fiware-orion.[SEU_DOMINIO_INTERNO]/v2/entities`
* IoT Agent (API): `https://fiware-iot.[SEU_DOMINIO_INTERNO]/version` (Deve mostrar a versão)
* QuantumLeap (API): `https://fiware-quantumleap.[SEU_DOMINIO_INTERNO]/version` (Deve mostrar a versão)
* Envio de dados do dispositivo IoT: `http://[IP_PUBLICO_PFSENSE]:7896`
* Configurar o IoT Agent (criar grupos de serviços, registrar dispositivos): `http://[IP_PUBLICO_PFSENSE]:4041`
* Envio de dados do dispositivo IoT com criptografia: `https://fiware-iot.[SEU_DOMINIO_INTERNO]` (precisaria que o dispositivo IoT resolvesse o nome ou houvesse um domínio público)

# Fase 5: Configuração de rede para o plano de gerenciamento
Este tráfego (Grafana, Keycloak) deve passar pelo NGINX na porta 443, usando subdomínios reais.

## Ação 1: Configurar seu DNS Interno (no [SEU_DOMINIO_INTERNO]) No seu servidor DNS (seja ele o pfSense ou um Windows Server), crie os seguintes registros Tipo A:

### Passo 1: Configurar no Windows Server (DNS Manager)

1. Acesse seu Windows Server.
2. Abra o Gerenciador de Servidores (Server Manager), vá em Ferramentas (Tools) e clique em DNS.
3. Na árvore da esquerda, expanda o nome do servidor e vá em Zonas de Pesquisa Direta (Forward Lookup Zones).
4. Localize e clique na zona [SEU_DOMINIO_INTERNO].
5. Agora, clique com o botão direito em uma área vazia do lado direito e escolha Novo Host (A ou AAAA)....

Você precisará criar 5 registros, um para cada serviço, todos apontando para o mesmo IP da VM ([IP_VM_DEBIAN]).

#### Registro 1 (Grafana):
* Nome: fiware-grafana (O campo FQDN preencherá automaticamente para fiware-grafana.[SEU_DOMINIO_INTERNO])
* Endereço IP: [IP_VM_DEBIAN]
* Clique em Adicionar Host.

#### Registro 2 (Keycloak):
* Nome: fiware-keycloak
* Endereço IP: [IP_VM_DEBIAN]
* Clique em Adicionar Host.

#### Registro 3 (Orion):
* Nome: fiware-orion
* Endereço IP: [IP_VM_DEBIAN]
* Clique em Adicionar Host.

#### Registro 4 (IoT Agent):
* Nome: fiware-iot
* Endereço IP: [IP_VM_DEBIAN]
* Clique em Adicionar Host.

#### Registro 5 (QuantumLeap):
* Nome: fiware-quantumleap
* Endereço IP: [IP_VM_DEBIAN]
* Clique em Adicionar Host.

### Passo 2: Configurar no PfSense (Opcional, mas Recomendado para a VLAN)
Se os dispositivos na VLAN 172.16.100.X usam o PfSense como DNS (e não o Windows Server), você também deve adicionar essas entradas no PfSense para que os dispositivos dessa rede consigam resolver os nomes (caso precisem se comunicar internamente via nome).

1. No PfSense, vá em Services > DNS Resolver (ou DNS Forwarder, dependendo do que você usa).
2. Role até o final, na seção Host Overrides.
3. Clique em Add.

* Host: fiware-grafana
* Domain: [SEU_DOMINIO_INTERNO]
* IP Address: [IP_VM_DEBIAN]
* Description: Fiware Grafana
* Clique em Save.

4. Repita o processo para os outros 4 nomes (fiware-keycloak, fiware-orion, fiware-iot, fiware-quantumleap).
5. Clique em Apply Changes no topo da página.

### Passo 3: Limpar Cache e Testar
No seu computador na rede 192.168.0.X:

1. Abra o Prompt de Comando (CMD) ou PowerShell.
2. Limpe o cache de DNS local:

```console
ipconfig /flushdns
```

3. Teste se o Windows Server está respondendo corretamente com o comando `ping` ou `nslookup`:

```console
ping fiware-keycloak.[SEU_DOMINIO_INTERNO]
```

Se responder Disparando contra [IP_VM_DEBIAN]..., está PERFEITO.

Agora você pode acessar `https://fiware-grafana.[SEU_DOMINIO_INTERNO]` e `https://fiware-keycloak.[SEU_DOMINIO_INTERNO]` de qualquer computador dentroda rede interna, sem precisar configurar nada neles!

## Ação 2: Configurar o pfSense (Acesso Externo de Admin) No pfSense, crie uma regra de NAT Port Forward para o tráfego de gerenciamento:

* Interface: WAN
* Protocolo: TCP
* Porta de Destino (WAN): 443 (ou outra de sua escolha, ex: 8443)
* IP de Redirecionamento (LAN): [IP_VM_DEBIAN]
* Porta de Redirecionamento (LAN): 443

Nota: Para administradores externos acessarem usando o IP público ([IP_PUBLICO_PFSENSE]), eles ainda precisarão editar seus arquivos hosts. Isso é necessário porque, ao acessarem por IP, o NGINX não saberá qual subdomínio entregar e mostrará apenas o serviço padrão (Grafana).

# Fase 6: Configuração de rede para o plano de dados (para os dispositivos IoT)
Aqui está a resposta para sua pergunta: seus dispositivos IoT não devem passar pelo NGINX. Eles devem ter um ponto de entrada dedicado, baseado em IP e porta, que o pfSense redireciona diretamente para o container do IoT Agent.

## Ação 1: Configurar o pfSense (Acesso de Dispositivos):

Configuração porta HTTP do IoT-Agent:

* Interface: WAN
* Protocolo: TCP (ou UDP, dependendo do seu dispositivo)
* Porta de Destino (WAN): 7896 (porta HTTP do IoT-Agent)
* IP de Redirecionamento (LAN): [IP_VM_DEBIAN]
* Porta de Redirecionamento (LAN): 7896

Configuração porta API:

* Interface: WAN
* Protocolo: TCP (ou UDP, dependendo do seu dispositivo)
* Porta de Destino (WAN): 4041 (porta da API)
* IP de Redirecionamento (LAN): [IP_VM_DEBIAN]
* Porta de Redirecionamento (LAN): 4041

---

# TESTES

## Dados do Cenário (Padrão)
* Onde executar os comandos curl: No seu computador pessoal (Windows/WSL).
* Onde executar os comandos de banco: Na VM Debian ([IP_VM_DEBIAN]).
* Service (Tenant): semarh
* Service Path (Caminho): /barragem01
* Chave de Segurança (API Key): [SUA_API_KEY] (pode criar na hora)
* ID do Dispositivo: sensor_teste_final

1. Provisionamento (Configurar o Terreno)
Antes de ligar o sensor, precisamos ensinar o sistema a recebê-lo.

1.1. Criar Grupo de Serviços (IoT Agent)
Define que qualquer sensor que envie a chave [SUA_API_KEY] pertence ao serviço semarh.

```bash
curl -iX POST \
  'http://[IP_VM_DEBIAN]:4041/iot/services' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: semarh' \
  -H 'fiware-servicepath: /barragem01' \
  -d '{
 "services": [
   {
     "apikey":      "[SUA_API_KEY]",
     "cbroker":     "http://fiware-orion:1026",
     "entity_type": "Thing",
     "resource":    "/iot/json"
   }
 ]
}'
```

1.2. Registrar o Dispositivo (Sensor)
Cadastra o sensor e define o que ele mede (t = temperature, h = humidity).

```bash
curl -iX POST \
  'http://[IP_VM_DEBIAN]:4041/iot/devices' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: semarh' \
  -H 'fiware-servicepath: /barragem01' \
  -d '{
 "devices": [
   {
     "device_id":   "sensor_teste_final",
     "entity_name": "urn:ngsi-ld:Sensor:TesteFinal",
     "entity_type": "Sensor",
     "protocol":    "PDI-IoTA-UltraLight",
     "transport":   "HTTP",
     "attributes": [
       { "object_id": "t", "name": "temperature", "type": "Float" },
       { "object_id": "h", "name": "humidity", "type": "Float" }
     ]
   }
 ]
}'
```

2. Persistência (Criar a Ponte)
Este passo é vital. Sem ele, os dados chegam mas não são salvos no histórico. Você só precisa fazer isso uma vez por Serviço/Path.

2.1. Criar Subscrição no Orion
Avisa o Orion: "Toda vez que chegar dados de t e h em uma entidade do tipo Thing, envie para o QuantumLeap".

```bash
curl -k -iX POST \
  'https://fiware-orion.[SEU_DOMINIO_INTERNO]/v2/subscriptions' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: semarh' \
  -H 'fiware-servicepath: /barragem01' \
  -d '{
  "description": "Ponte Thing para QuantumLeap",
  "subject": {
    "entities": [
      {
        "idPattern": ".*",
        "type": "Thing"
      }
    ],
    "condition": {
      "attrs": [
        "t",
        "h"
      ]
    }
  },
  "notification": {
    "http": {
      "url": "http://quantumleap-fiware:8668/v2/notify"
    },
    "attrs": [
      "t",
      "h"
    ],
    "metadata": ["dateCreated", "dateModified"]
  },
  "throttling": 0
}'
```

3. Simulação (O Teste Real)
Aqui fingimos ser o dispositivo enviando dados da rua.

3.1. Enviar Dados
Use seu IP Público (ou interno, se estiver testando localmente) e a porta 7896.

```bash
curl -iX POST \
  'http://[IP_PUBLICO_PFSENSE]:7896/iot/json?k=[SUA_API_KEY]&i=sensor_teste_final' \
  -H 'Content-Type: application/json' \
  -d '{"t": 25.5, "h": 60}'
```

Esperado: HTTP/1.1 200 OK.

4. Verificação (Tirando a Prova)

4.1. Verificar Dados em Tempo Real (Orion)
Confere se o sistema recebeu e processou o último valor.

```bash
curl -k -X GET 'https://fiware-orion.[SEU_DOMINIO_INTERNO]/v2/entities/Thing:sensor_teste_final?options=keyValues' \
  -H 'fiware-service: semarh' \
  -H 'fiware-servicepath: /barragem01'
```

Esperado: Um JSON mostrando t: 25.5 e h: 60.

4.2. Verificar Histórico (CrateDB)
Confere se o dado foi salvo no banco para o Grafana ler depois. (Execute na VM Debian)

```bash
sudo docker exec -it crate-fiware crash -c "SELECT time_index, t, h FROM mtsemarh.etthing ORDER BY time_index DESC LIMIT 5;"
```

Esperado: Uma tabela contendo a data e os valores 25.5 | 60.

5. Visualização (Grafana)
Acesse `https://fiware-grafana.[SEU_DOMINIO_INTERNO]`.

* Faça login (admin/admin).
* Adicione uma fonte de dados (Data Source) do tipo PostgreSQL:
* Host: crate-fiware:5432
* Database: doc
* User: crate
* TLS/SSL Mode: disable
* Crie um Dashboard e use a query SQL:

```shell
SELECT time_index AS "time", t AS "Temperatura" FROM mtsemarh.etthing ORDER BY 1
```
