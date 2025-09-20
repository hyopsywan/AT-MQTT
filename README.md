# AT-MQTT

# Projeto MQTT Seguro

## Descrição
Este projeto demonstra a configuração de um broker MQTT seguro (Mosquitto) com:
- Autenticação por usuário e senha  
- Controle de acesso (ACL)  
- Criptografia TLS/SSL  
- Clientes (sensor e assinante) configurados para usar credenciais  

A proposta é mostrar as vulnerabilidades comuns em um broker MQTT sem proteção e como corrigi-las.

---

## Estrutura do Repositório

├── mosquito/ # Configurações do Mosquitto
│ ├── autorização/ # Arquivos de ACL (controle de acesso)
│ ├── certificados/ # Certificados TLS/SSL
│ └── mosquito.conf # Configuração do broker
│
├── fonte/
│ └── sensor-de-temperatura-1.py # Cliente sensor (Python + Paho MQTT)
│
├── Dockerfile.sensor-de-temperatura-1
├── docker-compose.yml


---

## Serviços do Laboratório
- mqtt-broker → Eclipse Mosquitto 2.0.20 (com autenticação, ACL e TLS)  
- sensor → Publica temperatura a cada 5 segundos (Python + Paho MQTT)  
- subscriber → Cliente que assina os tópicos e mostra mensagens no log  

---

## Vulnerabilidades Corrigidas
1. **Acesso Anônimo**  
   - Problema: qualquer um se conectava sem credenciais  
   - Correção: `allow_anonymous false` + autenticação por usuário/senha  

2. **Falta de Autorização**  
   - Problema: usuários acessavam qualquer tópico  
   - Correção: uso de ACL para limitar permissões  

3. **Tráfego Sem Criptografia**  
   - Problema: dados enviados em texto puro  
   - Correção: configuração de TLS/SSL na porta 8883  

4. **Clientes Sem Autenticação**  
   - Problema: sensor e assinante conectavam sem credenciais  
   - Correção: clientes atualizados para usar usuário e senha  

---

## Usuários e Permissões
- admin_user / admin123 → Acesso completo  
- sensor_user / sensor123 → Pode publicar em `sensor/+`  
- subscriber_user / subscriber123 → Pode ler de `sensor/+`  

---

## Implementação Técnica do lab

### Passo 1: Criação de Arquivos de Autenticação
```bash
# Criado diretório de autenticação
mkdir -p mosquitto/config/auth

# Gerados hashes de senha usando mosquitto_passwd
docker run --rm eclipse-mosquitto:2.0.20 sh -c "mosquitto_passwd -c -b /tmp/passwd sensor_user sensor123 && mosquitto_passwd -b /tmp/passwd subscriber_user subscriber123 && mosquitto_passwd -b /tmp/passwd admin_user admin123 && cat /tmp/passwd"

### Passo 2: Configuração do Access Control List (ACL)
Arquivo mosquitto/config/auth/acl:
user sensor_user
topic write sensor/+

user subscriber_user
topic read sensor/+

user admin_user
topic readwrite #

### Passo 3: Geração de Certificados SSL

# Criado diretório de certificados
mkdir -p mosquitto/config/certs

# Gerados certificados SSL
openssl req -new -x509 -days 365 -nodes -out mosquitto/config/certs/ca.crt -keyout mosquitto/config/certs/ca.key -subj "/C=BR/ST=SP/L=SaoPaulo/O=MQTT-Security/OU=IT/CN=MQTT-CA"
openssl genrsa -out mosquitto/config/certs/server.key 2048
openssl req -new -out mosquitto/config/certs/server.csr -key mosquitto/config/certs/server.key -subj "/C=BR/ST=SP/L=SaoPaulo/O=MQTT-Security/OU=IT/CN=localhost"
openssl x509 -req -in mosquitto/config/certs/server.csr -CA mosquitto/config/certs/ca.crt -CAkey mosquitto/config/certs/ca.key -CAcreateserial -out mosquitto/config/certs/server.crt -days 365

### Passo 4: Configuração do Broker

Arquivo mosquitto/config/mosquitto.conf:
# Listener padrão (não criptografado - apenas para testes)
listener 1883 0.0.0.0

# Listener TLS/SSL (criptografado - produção)
listener 8883 0.0.0.0
cafile /mosquitto/config/certs/ca.crt
certfile /mosquitto/config/certs/server.crt
keyfile /mosquitto/config/certs/server.key

# Configurações de segurança
allow_anonymous false
password_file /mosquitto/config/auth/passwd
acl_file /mosquitto/config/auth/acl

### Passo 5: Atualização dos Clientes

-Sensor Python: Adicionada autenticação com client.username_pw_set()
-Subscriber: Configurado com parâmetros -u e -P
-Docker Compose: Adicionadas variáveis de ambiente para credenciais

# Uso
### Iniciar o Sistema
docker compose up -d --build
docker compose logs -f mqtt-subscriber

### Testar Conexões
Conexão Não Criptografada (Porta 1883)
docker run --rm eclipse-mosquitto:2.0.20 mosquitto_pub -h localhost -p 1883 -t sensor/test -m "teste não criptografado" -u admin_user -P admin123

Conexão Criptografada TLS (Porta 8883)
docker run --rm -v $(pwd)/mosquitto/config/certs:/tmp/certs eclipse-mosquitto:2.0.20 mosquitto_pub -h localhost -p 8883 -t sensor/test -m "teste criptografado" -u admin_user -P admin123 --cafile /tmp/certs/ca.crt


#  Detalhes de Conexão
### Broker MQTT

-Host: localhost (ou IP da máquina)
-Porta Não Criptografada: 1883
-Porta Criptografada: 8883
-Autenticação: Obrigatória

### Usuários Configurados

-admin_user / admin123 - Acesso completo a todos os tópicos
-sensor_user / sensor123 - Apenas publicar em tópicos sensor/+
-subscriber_user / subscriber123 - Apenas ler de tópicos sensor/+

# Segurança Implementada
-Autenticação: Acesso anônimo desabilitado, username/password obrigatório
-Autorização: ACL implementada com restrições por usuário
-Criptografia: TLS/SSL na porta 8883
-Auditoria: Logs detalhados para monitoramento de segurança
-Isolamento: Usuários com permissões mínimas necessárias





