# 🚀 Arquitetura Orientada a Eventos com KEDA: Auto-Scaling de Consumidores Kafka

> **Demo para Apresentação CNCF**: Uma demonstração prática completa de como o KEDA possibilita auto-scaling inteligente de consumidores Kafka baseado no lag de mensagens no Kubernetes.

## 📋 O que Você Vai Aprender

Esta demo apresenta um padrão completo de **Arquitetura Orientada a Eventos (EDA)** usando tecnologias CNCF:

- **KEDA**: Auto-scaler orientado a eventos do Kubernetes para scaling inteligente de pods
- **Kafka**: Plataforma de streaming distribuída para pipelines de dados em tempo real
- **Debezium**: Change Data Capture (CDC) para streaming de eventos de banco de dados
- **PostgreSQL**: Banco de dados origem com Write-Ahead Logging (WAL)
- **Kind**: Cluster Kubernetes local para desenvolvimento

### 🎯 Visão Geral da Arquitetura

```text
┌─────────────┐   Eventos CDC   ┌─────────────┐   Mensagens    ┌─────────────┐
│ PostgreSQL  │ ──────────────→ │   Kafka     │ ─────────────→ │ Consumidor  │
│(WAL=logical)│                 │  (Debezium) │                │   (KEDA)    │
└─────────────┘                 └─────────────┘                └─────────────┘
                                                                       ↑
                                                          Auto-scaling baseado
                                                            no lag de mensagens
```

## 🏗️ Passo 1: Configuração do Cluster Kubernetes Local

**Por que Kind?** Kind (Kubernetes in Docker) fornece um ambiente Kubernetes leve e reproduzível, perfeito para desenvolvimento e demonstrações.

```bash
# Criar um cluster Kubernetes local
kind create cluster --name cncf-demo

# Instalar metrics-server (necessário para decisões de scaling do KEDA)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Configurar metrics-server para desenvolvimento local (ignora verificação TLS)
kubectl -n kube-system patch deployment metrics-server --type='json' -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"},
  {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-preferred-address-types=InternalIP,Hostname"},
  {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-use-node-status-port"}
]'

# Aguardar o metrics-server ficar pronto
kubectl -n kube-system rollout status deploy/metrics-server
```

**🔍 O que está acontecendo?** Estamos configurando a fundação - um cluster Kubernetes com capacidades de coleta de métricas que o KEDA usará para decisões de scaling.

## ⚡ Passo 2 e 3: Preparar o Namespace e Instalar KEDA - O Auto-Scaler Orientado a Eventos

**💡 Boa Prática**: Isolar recursos em namespaces dedicados melhora segurança, gerenciamento de recursos e troubleshooting.

**O que é KEDA?** KEDA é um Auto-Scaler Orientado a Eventos baseado em Kubernetes que pode escalar suas aplicações de 0 a n baseado em métricas externas como profundidade de fila de mensagens, resultados de consultas de banco de dados ou métricas customizadas.

```bash
# Adicionar repositório Helm do KEDA
# Criar namespace dedicado para nossa demo EDA
kubectl create ns eda-poc

helm repo add kedacore https://kedacore.github.io/charts
helm repo update

# Instalar KEDA com Helm
helm install keda kedacore/keda -n keda --create-namespace

# Verificar se o operador KEDA está rodando
kubectl -n keda rollout status deploy/keda-operator
```

**🔍 Principais Benefícios do KEDA:**

- **Scale-to-Zero**: Economiza recursos quando não há trabalho a fazer
- **Orientado a Eventos**: Reage a eventos reais de negócio, não apenas CPU/memória
- **Multi-Fonte**: Suporte para mais de 50 fontes de eventos (Kafka, RabbitMQ, Azure, AWS, etc.)
- **Nativo do Kubernetes**: Usa HPA e VPA padrão por baixo dos panos

## �️ Passo 4: Configuração do Banco PostgreSQL

**Por que PostgreSQL primeiro?** Precisamos ter o banco de dados operacional antes de configurar o Kafka Connect com Debezium, pois o conector CDC precisa se conectar ao PostgreSQL para monitorar mudanças.

**Destaques da Configuração**: Nossa configuração do PostgreSQL inclui configurações específicas para CDC:

```bash
# Instalar PostgreSQL com configuração pronta para CDC
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install postgresql bitnami/postgresql -n eda-poc -f src/pg-values.yaml \
--set image.repository=bitnamilegacy/postgresql \
--set global.security.allowInsecureImages=true

# Criar a tabela customer e schema inicial
kubectl apply -f src/pg-bootstrap.yaml
kubectl -n eda-poc wait --for=condition=complete job/pg-bootstrap --timeout=120s
```

**🔍 Configurações Críticas do PostgreSQL** (veja `src/pg-values.yaml`):

- `wal_level = logical`: Habilita replicação lógica para CDC
- `max_wal_senders = 10`: Máximo de processos WAL sender concorrentes
- `max_replication_slots = 10`: Máximo de slots de replicação para consumidores CDC

## 📊 Passo 5: Apache Kafka com Operador Strimzi

**O que é Strimzi?** Strimzi fornece uma maneira de executar Apache Kafka no Kubernetes em várias configurações de deployment, tornando o gerenciamento do Kafka cloud-native.

```bash
# Instalar operador Strimzi Kafka
kubectl apply -f 'https://strimzi.io/install/latest?namespace=eda-poc' -n eda-poc

# Aguardar o operador ficar pronto
kubectl -n eda-poc rollout status deploy/strimzi-cluster-operator

# Fazer deploy de um cluster Kafka single-node (adequado para demos)
kubectl -n eda-poc apply -f https://strimzi.io/examples/latest/kafka/kafka-single-node.yaml

# Aguardar o cluster Kafka ficar pronto (pode levar 2-3 minutos)
kubectl -n eda-poc wait kafka/my-cluster --for=condition=Ready --timeout=300s
```

**🔍 O que está acontecendo?**

- O operador Strimzi gerencia todo o ciclo de vida do Kafka
- O cluster inclui brokers Kafka, ZooKeeper e serviços necessários
- A condição Ready garante que todos os componentes estão operacionais

## 🔄 Passo 6: Change Data Capture com Debezium

**O que é CDC?** Change Data Capture permite rastrear e capturar mudanças em seu banco de dados em tempo real, transformando seu banco de dados em um stream de eventos.

**Por que agora?** Agora que temos PostgreSQL e Kafka operacionais, podemos conectar ambos através do Debezium para capturar mudanças do banco em tempo real.

```bash
# Fazer deploy do Kafka Connect com Debezium e auto-registro do conector PostgreSQL
kubectl apply -f src/kafka-connect.yaml

# Aguardar o Kafka Connect ficar pronto
kubectl -n eda-poc rollout status deploy/debezium-connect
```

**🔍 Conceitos Importantes:**

- **Write-Ahead Log (WAL)**: Log de transações do PostgreSQL que o Debezium lê
- **Replicação Lógica**: Permite que sistemas externos se inscrevam para mudanças de dados
- **Nomenclatura de Tópicos**: `dbserver1.public.customer` = `{servidor}.{schema}.{tabela}`

## 🎯 Passo 7: Deploy do Consumer Auto-Scaling com KEDA

Aqui é onde a mágica acontece! Nosso consumer irá escalar automaticamente baseado no lag de mensagens do Kafka.

```bash
# Fazer deploy do consumer com ScaledObject do KEDA
kubectl apply -f src/consumer-keda.yaml
```

**🔍 Análise da Configuração KEDA:**

```yaml
# Do arquivo consumer-keda.yaml
spec:
  scaleTargetRef:
    name: cdc-consumer          # Deployment alvo para escalar
  minReplicaCount: 0           # Escalar para zero quando não há mensagens
  maxReplicaCount: 3           # Máximo de pods para alta carga
  triggers:
  - type: kafka
    metadata:
      consumerGroup: eda-consumer    # Grupo de consumidor Kafka para monitorar
      topic: dbserver1.public.customer  # Tópico para monitorar
      lagThreshold: "1"          # Escalar quando lag > 1 mensagem
```

**💡 Comportamento de Scaling:**

- **Scale to Zero**: Nenhum pod quando não há mensagens (otimização de custos)
- **Resposta Imediata**: Pods iniciam quando mensagens chegam
- **Baseado em Carga**: Mais pods para maior lag de mensagens
- **Shutdown Gracioso**: Confirmação adequada de mensagens antes do término do pod

## 🧪 Step 8: Live Demonstration

Let's see auto-scaling in action!

## Generate test data to trigger scaling events

### 🧪 Passo 8: Demonstração ao Vivo

```bash
# Gerar dados de teste para disparar eventos de scaling
kubectl -n eda-poc delete job pg-bulk-seed --ignore-not-found
kubectl -n eda-poc apply -f src/pg-bulk-seed.yaml
kubectl -n eda-poc wait --for=condition=complete job/pg-bulk-seed --timeout=180s

# Assista à mágica acontecer - pods escalando em tempo real!
kubectl -n eda-poc get pods -l app=cdc-consumer -w
```

**🎬 O que Observar:**

1. **Estado Inicial**: Zero pods consumer (otimização de custos)
2. **Inserção de Dados**: Job cria registros de clientes no PostgreSQL
3. **Trigger CDC**: Debezium captura mudanças e envia para Kafka
4. **Resposta KEDA**: Detecta lag de mensagens e escala pods consumer
5. **Processamento**: Consumers processam mensagens e as confirmam
6. **Scale Down**: Após processamento, pods voltam para zero

## 📊 Monitoramento e Observabilidade

### Verificar Lag do Consumer (Ponto de Decisão do KEDA)

```bash
# Monitorar lag do grupo de consumidores Kafka (o que o KEDA monitora)
kubectl -n eda-poc exec -it my-cluster-kafka-0 -- \
  /opt/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe --group eda-consumer
```

### Monitorar Eventos de Scaling do KEDA

```bash
# Ver decisões de scaling do KEDA
kubectl -n eda-poc describe scaledobject cdc-consumer-so

# Monitorar HPA criado pelo KEDA
kubectl -n eda-poc describe hpa keda-hpa-cdc-consumer-so
```

### Scaling de Pods em Tempo Real

```bash
# Assistir pods escalarem para cima e para baixo
kubectl -n eda-poc get pods -l app=cdc-consumer --watch

# Verificar logs do consumer para ver processamento de mensagens
kubectl -n eda-poc logs -l app=cdc-consumer -f
```

## 🔧 Opções de Configuração Avançada

### Múltiplos Triggers de Scaling

KEDA suporta múltiplos triggers simultaneamente:

```yaml
triggers:
- type: kafka          # Primário: Lag de mensagens
- type: prometheus     # Secundário: Métricas customizadas  
- type: cron          # Agendado: Scaling preditivo
```

## 🎓 Pontos de Aprendizado Chave para Audiência CNCF

### 1. **Arquitetura Orientada a Eventos Cloud-Native**

- **Sistemas Reativos**: Aplicações respondem a eventos, não solicitações
- **Baixo Acoplamento**: Componentes se comunicam através de eventos
- **Escalabilidade**: Scaling automático baseado na demanda de negócio

### 2. **Papel do KEDA no Kubernetes Moderno**

- **Além de CPU/Memória**: Escalar baseado em métricas de negócio
- **Otimização de Custos**: Capacidades de scale-to-zero
- **Multi-Cloud**: Funciona com qualquer distribuição Kubernetes

### 3. **Benefícios Operacionais**

- **Eficiência de Recursos**: Pague apenas pelo que usar
- **Experiência do Desenvolvedor**: Configuração declarativa de scaling
- **Pronto para Produção**: Recursos built-in de estabilidade e confiabilidade

### 4. **Ecossistema de Integração**

- **50+ Fontes de Eventos**: Kafka, RabbitMQ, Azure, AWS, GCP
- **Compatível com Padrões**: Usa HPA/VPA do Kubernetes
- **Extensível**: Scalers customizados para qualquer fonte de evento

## 🔍 Solução de Problemas Comuns

### Pods Consumer Não Escalando

```bash
# Verificar status do operador KEDA
kubectl -n keda get pods

# Verificar configuração do ScaledObject
kubectl -n eda-poc describe scaledobject cdc-consumer-so

# Verificar eventos do KEDA
kubectl -n eda-poc get events --sort-by='.lastTimestamp' | grep -i keda
```

### CDC Não Funcionando

```bash
# Verificar configurações de replicação do PostgreSQL
kubectl -n eda-poc exec -it postgresql-0 -- psql -U postgres -d appdb -c "SHOW wal_level;"

# Verificar status do conector Debezium
kubectl -n eda-poc port-forward svc/debezium-connect 8083:8083 &
curl http://localhost:8083/connectors/pg-customer-cdc/status
```

### Problemas com Kafka

```bash
# Listar tópicos
kubectl -n eda-poc exec -it my-cluster-kafka-0 -- \
  /opt/kafka/bin/kafka-topics.sh --list --bootstrap-server localhost:9092

# Verificar mensagens do tópico
kubectl -n eda-poc exec -it my-cluster-kafka-0 -- \
  /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic dbserver1.public.customer \
  --from-beginning --max-messages 5
```

## 🧹 Limpeza

```bash
# Remover todo o cluster da demo
kind delete cluster --name cncf-demo
```

## 📚 Recursos Adicionais

### Projetos CNCF Utilizados

- **[KEDA](https://keda.sh/)**: Auto-Scaler Orientado a Eventos do Kubernetes
- **[Strimzi](https://strimzi.io/)**: Operador Kafka para Kubernetes
- **[Kubernetes](https://kubernetes.io/)**: Plataforma de orquestração de containers

### Saiba Mais

- [Documentação KEDA](https://keda.sh/docs/)
- [Documentação Strimzi](https://strimzi.io/documentation/)
- [Padrões Debezium CDC](https://debezium.io/documentation/)
- [Padrões de Arquitetura Orientada a Eventos](https://www.enterpriseintegrationpatterns.com/)

### Comunidade

- **KEDA**: [GitHub](https://github.com/kedacore/keda) | [Slack](https://kubernetes.slack.com/messages/keda)
- **Strimzi**: [GitHub](https://github.com/strimzi/strimzi-kafka-operator)
- **CNCF**: [Landscape](https://landscape.cncf.io/) | [Eventos](https://www.cncf.io/events/)

---

**🎉 Parabéns!** Você demonstrou com sucesso como as tecnologias CNCF trabalham juntas para criar aplicações inteligentes, orientadas a eventos e eficientes em custos no Kubernetes.

**Conclusão Principal**: Aplicações modernas devem reagir a eventos de negócio, não apenas métricas de sistema, e o KEDA torna isso possível de forma nativa no Kubernetes.
