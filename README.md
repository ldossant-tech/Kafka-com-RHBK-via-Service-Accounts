# **Kafka com RHBK via Service Accounts**

Este guia documenta o passo a passo usado para configurar o Red Hat Build of Keycloak (RHBK) para autenticar e autorizar clients Kafka usando service accounts.

O objetivo validado foi:

- `orders-producer` consegue escrever no topico `orders`.
- `orders-consumer` consegue ler do topico `orders` usando o group `orders-consumer-group`.
- As permissoes sao gerenciadas no RHBK usando Keycloak Authorization Services.

## **Kafka**

O Kafka foi configurado no arquivo [kafka.yaml](./kafka.yaml).

O ponto essencial para o OAuth funcionar foi configurar a CA do RHBK nos dois lugares:

```yaml
spec:
  kafka:
    authorization:
      type: keycloak
      tlsTrustedCertificates:
        - secretName: oauth-server-cert
          pattern: "*.crt"
    listeners:
      - name: external
        authentication:
          type: oauth
          tlsTrustedCertificates:
            - secretName: oauth-server-cert
              pattern: "*.crt"
```

Sem o `tlsTrustedCertificates` no listener OAuth, o broker falhava ao buscar o JWKS do RHBK com:

```
PKIX path building failed
```

Aplicacao:

```bash
oc apply -f kafka.yaml
```

Validacao:

```bash
oc get kafka kafka-cluster -n authorization
oc get pods -n authorization
```

Resultado esperado:

```
kafka-cluster Ready=True
kafka-cluster-broker-0 1/1 Running
kafka-cluster-broker-1 1/1 Running
kafka-cluster-broker-2 1/1 Running
```

## **RHBK: Client `kafka`**

O client `kafka` e o resource server usado pelo Kafka Authorizer.

No RHBK:

```
Clients -> kafka -> Settings
```

Confirme:

```
Client authentication: ON
Authorization: ON
```

Depois use:

```
Clients -> kafka -> Authorization
```

Todas as resources, scopes, policies e permissions ficam nesse client.

## **RHBK: Authorization scopes**

Crie estes scopes em:

```
Clients -> kafka -> Authorization -> Authorization scopes
```

Scopes usados neste laboratorio:

```
Describe
Read
Write
Create
IdempotentWrite
```

Os nomes sao case-sensitive.

## **RHBK: Clients das aplicacoes**

Crie dois clients OIDC confidential.

### **Producer**

```
Client ID: orders-producer
Client authentication: ON
Service accounts roles: ON
Authorization: OFF
Standard flow: OFF
Direct access grants: OFF
```

### **Consumer**

```
Client ID: orders-consumer
Client authentication: ON
Service accounts roles: ON
Authorization: OFF
Standard flow: OFF
Direct access grants: OFF
```

Copie os secrets em:

```
Clients -> orders-producer -> Credentials
Clients -> orders-consumer -> Credentials
```

## **RHBK: Realm roles**

**Realm roles** são papéis globais dentro de um **Realm**

Crie as roles em:

```
Realm roles -> Create role
```

Roles:

```
kafka-orders-producer
kafka-orders-consumer
```

## **RHBK: Service account roles**

Atribua as roles para as service accounts.

Producer:

```
Clients -> orders-producer -> Service account roles
Assign role -> kafka-orders-producer
```

Consumer:

```
Clients -> orders-consumer -> Service account roles
Assign role -> kafka-orders-consumer
```

As service accounts aparecem tambem como usuarios:

```
service-account-orders-producer
service-account-orders-consumer
```

## **RHBK: Resources**

**Resources** são os objetos que você quer proteger/autorizAR.

Crie as resources em:

```
Clients -> kafka -> Authorization -> Resources
```

### **Topico**

```
Name: Topic:orders
Type: Topic
Authorization scopes: Describe, Read, Write, Create
```

### **Consumer group**

```
Name: Group:orders-consumer-group
Type: Group
Authorization scopes: Describe, Read
```

### **Cluster**

```
Name: Cluster:kafka-cluster
Type: Cluster
Authorization scopes: Describe, IdempotentWrite
```

O formato do `Name` precisa seguir o padrao:

```
TYPE:NAME
```

Exemplos validos:

```
Topic:orders
Group:orders-consumer-group
Cluster:kafka-cluster
```

Remova ou nao use resources padrao como:

```
Default Resource
```

O Strimzi Keycloak Authorizer nao consegue interpretar `Default Resource` porque ele nao segue o formato `TYPE:NAME`.

## **RHBK: Policies**

**Policies** definem **quem** pode receber uma permissão

Crie policies em:

```
Clients -> kafka -> Authorization -> Policies
```

### **Producer policy**

```
Type: Role
Name: orders-producer-policy
Role type: Realm role
Role: kafka-orders-producer
Logic: Positive
```

### **Consumer policy**

```
Type: Role
Name: orders-consumer-policy
Role type: Realm role
Role: kafka-orders-consumer
Logic: Positive
```

Use realm roles, nao client roles, porque os tokens das service accounts contem as roles em:

```json
"realm_access": {
  "roles": [
    "kafka-orders-producer"
  ]
}
```

## **RHBK: Permissions**

No RHBK, **Permissions** são a regra final que junta:

`quem pode + fazer qual ação + em qual recurso`

Ou seja, a permission conecta:

`Policy + Scope + Resource`

No seu cenário Kafka:

`Policy   = quem pode
Scope    = qual ação
Resource = em qual objeto Kafka`

Use **Scope-based permissions**.

Nao use Resource-based permissions para este fluxo.

As permissions finais usadas foram:

### **Describe compartilhado do topico**

```
Type: Scope-based
Name: orders-topic-describe
Resource: Topic:orders
Authorization scopes: Describe
Policies: orders-producer-policy, orders-consumer-policy
Decision strategy: Affirmative
```

### **Producer escreve no topico**

```
Type: Scope-based
Name: orders-producer-topic-write
Resource: Topic:orders
Authorization scopes: Write, Create
Policies: orders-producer-policy
Decision strategy: Affirmative
```

### **Producer IdempotentWrite no cluster**

```
Type: Scope-based
Name: orders-producer-idempotent-write
Resource: Cluster:kafka-cluster
Authorization scopes: IdempotentWrite
Policies: orders-producer-policy
Decision strategy: Affirmative
```

### **Consumer le o topico**

```
Type: Scope-based
Name: orders-consumer-topic-read
Resource: Topic:orders
Authorization scopes: Read
Policies: orders-consumer-policy
Decision strategy: Affirmative
```

### **Consumer usa o group**

```
Type: Scope-based
Name: orders-consumer-group-read
Resource: Group:orders-consumer-group
Authorization scopes: Describe, Read
Policies: orders-consumer-policy
Decision strategy: Affirmative
```

## **RHBK: Evaluate**

**Evaluate** é a tela usada para **simular uma decisão de autorização** antes de testar na aplicação ou no Kafka.

Ela responde à pergunta:

`Este usuário/client/service account pode executar estes scopes neste resource?`

Use:

```
Clients -> kafka -> Authorization -> Evaluate
```

### **Producer**

Selecione:

```
User: service-account-orders-producer
Roles: kafka-orders-producer
Resource: Topic:orders
Scopes: Describe, Write, Create
```

Resultado esperado:

```
Permit
Granted scopes: Describe, Write, Create
```

### **Consumer topic**

Selecione:

```
User: service-account-orders-consumer
Roles: kafka-orders-consumer
Resource: Topic:orders
Scopes: Describe, Read
```

Resultado esperado:

```
Permit
Granted scopes: Describe, Read
```

### **Consumer group**

Selecione:

```
User: service-account-orders-consumer
Roles: kafka-orders-consumer
Resource: Group:orders-consumer-group
Scopes: Describe, Read
```

Resultado esperado:

```
Permit
Granted scopes: Describe, Read
```

## **Pod de teste**

Criamos um pod temporario:

```bash
oc run kafka-client -n authorization \
  --image=registry.redhat.io/amq-streams/kafka-40-rhel9:3.0.1 \
  --restart=Never \
  --command -- sleep infinity
```

Entrar no pod:

```bash
oc exec -n authorization -it kafka-client -- bash
```

## **Truststore do client Kafka**

O client Kafka precisou confiar em duas CAs:

- CA do RHBK, para buscar token no token endpoint.
- CA do Kafka cluster, para conectar no bootstrap externo via TLS.

Extrair CA do RHBK:

```bash
oc extract secret/oauth-server-cert -n authorization --to=/tmp/kafka-certs --confirm
oc cp /tmp/kafka-certs/ca.crt authorization/kafka-client:/tmp/rhbk-ca.crt
```

Criar truststore:

```bash
oc exec -n authorization kafka-client -- \
  keytool -importcert -noprompt \
  -alias rhbk-ca \
  -file /tmp/rhbk-ca.crt \
  -keystore /tmp/rhbk-oauth.truststore.p12 \
  -storetype PKCS12 \
  -storepass changeit
```

Extrair CA do Kafka:

```bash
oc extract secret/kafka-cluster-cluster-ca-cert -n authorization --to=/tmp/kafka-certs --confirm
oc cp /tmp/kafka-certs/ca.crt authorization/kafka-client:/tmp/kafka-cluster-ca.crt
```

Importar no mesmo truststore:

```bash
oc exec -n authorization kafka-client -- \
  keytool -importcert -noprompt \
  -alias kafka-cluster-ca \
  -file /tmp/kafka-cluster-ca.crt \
  -keystore /tmp/rhbk-oauth.truststore.p12 \
  -storetype PKCS12 \
  -storepass changeit
```

## **Producer properties**

Dentro do pod `kafka-client`, crie:

```bash
cat > /tmp/orders-producer.properties <<'EOF'
security.protocol=SASL_SSL
sasl.mechanism=OAUTHBEARER
sasl.login.callback.handler.class=io.strimzi.kafka.oauth.client.JaasClientOauthLoginCallbackHandler
sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
  oauth.client.id="orders-producer" \
  oauth.client.secret="<ORDERS_PRODUCER_SECRET>" \
  oauth.token.endpoint.uri="https://keycloak.apps.ldossant.vmware.tamlab.rdu2.redhat.com/realms/kafka/protocol/openid-connect/token" \
  oauth.ssl.truststore.location="/tmp/rhbk-oauth.truststore.p12" \
  oauth.ssl.truststore.password="changeit" \
  oauth.ssl.truststore.type="PKCS12" ;
ssl.truststore.location=/tmp/rhbk-oauth.truststore.p12
ssl.truststore.password=changeit
ssl.truststore.type=PKCS12
EOF
```

## **Consumer properties**

Dentro do pod `kafka-client`, crie:

```bash
cat > /tmp/orders-consumer.properties <<'EOF'
security.protocol=SASL_SSL
sasl.mechanism=OAUTHBEARER
sasl.login.callback.handler.class=io.strimzi.kafka.oauth.client.JaasClientOauthLoginCallbackHandler
sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
  oauth.client.id="orders-consumer" \
  oauth.client.secret="<ORDERS_CONSUMER_SECRET>" \
  oauth.token.endpoint.uri="https://keycloak.apps.ldossant.vmware.tamlab.rdu2.redhat.com/realms/kafka/protocol/openid-connect/token" \
  oauth.ssl.truststore.location="/tmp/rhbk-oauth.truststore.p12" \
  oauth.ssl.truststore.password="changeit" \
  oauth.ssl.truststore.type="PKCS12" ;
ssl.truststore.location=/tmp/rhbk-oauth.truststore.p12
ssl.truststore.password=changeit
ssl.truststore.type=PKCS12
EOF
```

## **Teste do producer**

```bash
printf "pedido-ok\n" | /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server kafka-cluster-kafka-bootstrap-authorization.apps.ldossant.vmware.tamlab.rdu2.redhat.com:443 \
  --topic orders \
  --producer.config /tmp/orders-producer.properties
```

Resultado esperado:

```
Sem erro TOPIC_AUTHORIZATION_FAILED
```

Um warning como este pode ser ignorado no laboratorio:

```
Expiring credential expires...
```

Ele indica apenas que o access token tem vida curta.

## **Teste do consumer**

```bash
/opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server kafka-cluster-kafka-bootstrap-authorization.apps.ldossant.vmware.tamlab.rdu2.redhat.com:443 \
  --topic orders \
  --group orders-consumer-group \
  --from-beginning \
  --timeout-ms 10000 \
  --consumer.config /tmp/orders-consumer.properties
```

Resultado validado:

```
pedido-ok
```

## Possiveis Erros

### **Broker com `PKIX path building failed` no JWKS**

Causa: broker nao confia no certificado HTTPS do RHBK.

Correcao: configurar `tlsTrustedCertificates` em:

- `spec.kafka.authorization`
- `spec.kafka.listeners[].authentication`

### **Client com `PKIX path building failed` ao buscar token**

Causa: `kafka-console-producer.sh` ou `kafka-console-consumer.sh` nao confia na CA do RHBK.

Correcao: importar `oauth-server-cert/ca.crt` no truststore do client e configurar:

```
oauth.ssl.truststore.location="/tmp/rhbk-oauth.truststore.p12"
oauth.ssl.truststore.password="changeit"
oauth.ssl.truststore.type="PKCS12"
```

### **Client com `SSL handshake failed` no bootstrap Kafka**

Causa: client nao confia na CA do Kafka listener externo.

Correcao: importar `kafka-cluster-cluster-ca-cert/ca.crt` no truststore do client e configurar:

```
ssl.truststore.location=/tmp/rhbk-oauth.truststore.p12
ssl.truststore.password=changeit
ssl.truststore.type=PKCS12
```

### **`Failed to parse Resource: Default Resource`**

Causa: alguma permission do RHBK concede resource chamado `Default Resource`.

Correcao: remover/desabilitar permissions e resources padrao. Use somente resources no formato:

```
TYPE:NAME
```

### **`TOPIC_AUTHORIZATION_FAILED`**

Causa: permissao efetiva no RHBK nao concede `Describe` e/ou `Write` para o producer.

Correcao:

- Validar no Evaluate com `service-account-orders-producer`.
- Garantir `Topic:orders -> Describe, Write, Create`.
- Usar Scope-based permissions.

### **Consumer nao le mensagens**

Causa comum: falta permissao no group.

Correcao:

- `Topic:orders -> Describe, Read`
- `Group:orders-consumer-group -> Describe, Read`

## **Observacoes finais**

- Para service accounts, use Role policies. Service accounts nao recebem groups como usuarios comuns.
- No Evaluate, teste a service account como usuario: `service-account-orders-producer` ou `service-account-orders-consumer`.
- Para este setup, use Scope-based permissions.

## Aviso final

Este repositorio e uma recomendacao tecnica para demonstrar uma arquitetura de observabilidade com OpenTelemetry no OpenShift. Ele nao substitui a documentacao oficial da Red Hat. Para implantacoes em ambiente produtivo, siga sempre as documentacoes oficiais, a matriz de suporte vigente, as politicas internas e o dimensionamento recomendado para a versao do OpenShift e dos Operators instalados.
