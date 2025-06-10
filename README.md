# API - Artefatos

## 1. Introdução
Este `README.md` descreve a arquitetura proposta para um sistema de geração assíncrona de artefatos (XLSX/PDF), incluindo recursos de:
- **Idempotência**  
- **Cache de resultados e status**  
- **Tolerância a falhas**
- **Escalabilidade**

---

## 2. Objetivos Funcionais
1. **Receber requisições** HTTP de geração de artefatos.  
2. **Garantir idempotência** para evitar jobs duplicados.  
3. **Persistir e rastrear** o status de cada job.  
4. **Processar** de forma assíncrona via filas e workers.  
5. **Armazenar** o resultado em S3 e disponibilizar download.  
6. **Servir** o status e o arquivo final sob demanda com cache.

---

## 3. Visão de Arquitetura de Alto Nível
```text
Client ↔ Load Balancer/Nginx
       ↳ Request Service ──▶ Redis (cache/idempotency)
                         ↳ PostgreSQL (jobs)
                         └─▶ RabbitMQ ──▶ Workers (XLSX/PDF) ──▶ S3
       ↳ Status Service ─▶ Redis/PostgreSQL
       ↳ Download Service ─▶ S3 + CloudFront CDN
```

---

## 4. Componentes Principais

| Componente | Responsabilidade |
|------------|------------------|
| **Load Balancer/Nginx** | Distribuir requisições HTTP entre múltiplas instâncias dos micro-serviços. |
| **Request Service** | Endpoints `POST /generate` e lógica de idempotência + enqueue. |
| **Redis** | - Armazenar chaves de idempotência (`SETNX`)<br>- Cache de status e de URL de download para acelerar respostas |
| **PostgreSQL (Jobs)** | - Persistir cada job: id, payload, timestamps, `idempotency_key`, status (`queued`, `processing`, `done`, `failed`) |
| **RabbitMQ** | Fila durável para enfileirar mensagens de geração<br>- Configurar TTL e retries<br>- Dead-Letter Queue para falhas irreversíveis |
| **Workers (XLSX/PDF)** | - Consumir mensagem<br>- Gerar artefato<br>- Atualizar status no PostgreSQL<br>- Salvar no S3<br>- Publicar resultado no cache |
| **Amazon S3** | Armazenar binários gerados (XLSX/PDF) de forma durável e escalável. |
| **Status Service** | Endpoint `GET /status/{job_id}`:<br>- Consulta primeiro no Redis<br>- Em cache miss, busca no PostgreSQL e repopula cache |
| **Download Service** | Endpoint `GET /download/{job_id}`:<br>- Valida sessão/permissões<br>- Retorna URL pré-assinada do S3 |

---

## 5. Fluxo de Requisição e Processamento

1. **Cliente** faz `POST /generate` com payload + `idempotency_key`.  
2. **Request Service**:
   1. Checa `idempotency_key` no Redis (SETNX).  
   2. Se existe, retorna imediatamente `job_id` em cache.  
   3. Senão:
      - Persiste novo registro de job no PostgreSQL.  
      - Enfileira mensagem em RabbitMQ.  
      - Armazena `idempotency_key` no Redis com TTL.  
      - Retorna `job_id` ao cliente.  
3. **Worker** (XLSX ou PDF) consome mensagem:
   1. Marca status = `processing` no PostgreSQL.  
   2. Gera artefato; em caso de erro repetido, encotra DLQ.  
   3. Salva arquivo no S3.  
   4. Atualiza status = `done` e grava URL (ou sinaliza cache).  
4. **Cliente** consulta `GET /status/{job_id}`:
   1. Primeiro tenta Redis.  
   2. Se cache miss, lê PostgreSQL e repopula cache.  
   3. Retorna status (`queued`/`processing`/`done`/`failed`).  
5. **Cliente** faz `GET /download/{job_id}`:
   1. Valida acesso (autorização/TTL da URL).  
   2. Retorna URL pré-assinada do S3 ou faz proxy com stream.

---

## 6. Justificativas de Design

| Requisito | Solução | Benefício |
|-----------|---------|-----------|
| **Idempotência** | Redis SETNX + chave | Evita jobs duplicados em retries de clientes ou timeouts |
| **Escalabilidade** | Micro-serviços + fila | Separação clara de responsabilidades; workers independentes |
| **Tolerância a falhas** | Dead-Letter Queue | Mensagens problemáticas não bloqueiam a fila principal |
| **Performance** | Redis como cache | Respostas rápidas para status e download; menor carga no DB |
| **Persistência** | PostgreSQL (ACID) | Trilha auditável e confiável dos jobs |
| **Durabilidade** | S3 para artefatos | Armazenamento elástico, custo otimizado e integração nativa |

---

## 7. Conclusão
A proposta atende integralmente ao desafio, oferecendo:
- Garantia de **idempotência** e **cache** para reduzir retrabalho.  
- Arquitetura **assíncrona** e **escalável** baseada em filas.  
- **Tolerância a falhas** via DLQ.
- Experiência consistente de **status** e **download** para o cliente.  

## 8. Justificativas

### RabbitMQ vs. Kafka
- RabbitMQ escolhido por simplicidade operacional 
  e garantias ACID vs. Kafka (overkill para este volume)
  vs. Redis Streams (menos maduro para produção)

### PostgreSQL vs. Alternativas
- PostgreSQL por ACID, JSON support
  e maturidade vs. MySQL (menor flexibilidade JSON)

### 📈 Estratégia de Escalabilidade

**Horizontal Scaling (Adicionar Pods):**
- APIs: Stateless, fácil replicar (HPA: 2-10 pods)
- Workers: Jobs independentes, escala por queue depth (1-10 pods)

**Vertical Scaling (Maior CPU/RAM):**
- PostgreSQL Primary: ACID consistency needs

**Cluster Scaling (Múltiplos Nodes):**
- RabbitMQ: HA + message throughput (3 nodes)
- Redis: Memory distribution + HA (cluster)

## 9. Diagrama final
[Link do whimsical](https://whimsical.com/challenge-tech-lead-E1eQyVbgqvB1oAXjL8tLCj)

## 10. Vantagens

### **ACID**
Propriedades fundamentais do PostgreSQL que garantem confiabilidade:
- **Atomicity**: Transação completa ou nada (job criado E enviado pra fila, ou nenhum dos dois)
- **Consistency**: Estado sempre válido (sem `job_id` duplicado ou status inválido)  
- **Isolation**: Transações paralelas não se interferem (workers não pegam mesmo job)
- **Durability**: Dados commitados nunca se perdem (jobs persistem após crash)

### **HA (High Availability)**
Sistema disponível 99.9%+ do tempo, mesmo com falhas de componentes:
- **RabbitMQ**: Cluster de 3 nodes - falha de 1 não para o sistema
- **Redis**: Masters + replicas com fail-over automático

### **HPA (Horizontal Pod Autoscaler)**
Controller do `Kubernetes` que adiciona/remove pods automaticamente:
- **APIs**: Escala baseado em CPU > 70% (2-10 pods)
- **Workers**: Escala baseado em queue depth > 10 jobs/worker (1-10 pods)
- **Benefício**: Scaling automático sem intervenção manual

### **Pods**
Menor unidade de deployment no Kubernetes:
- **Definição**: Container + recursos (CPU, memória, rede)
- **Isolamento**: Falha de um pod não afeta outros
- **Stateless**: Podem ser criados/destruídos sem perda de dados
- **Exemplo**: Cada worker XLSX roda em pod separado

### **Queue Depth Scaling**
Métrica para scaling baseada no número de mensagens na fila:
- **Lógica**: Mais jobs aguardando = mais workers necessários
- **Target**: 10 jobs por worker ativo
- **Vantagem**: Métrica mais precisa que CPU (worker pode estar idle aguardando job)
- **Exemplo**: 100 jobs na fila ÷ 10 = 10 workers necessários

### **Dead Letter Queue (DLQ)**
Fila especial para mensagens que falharam múltiplas vezes:
- **Função**: Evita que jobs problemáticos travem a fila principal
- **Trigger**: Após 3 tentativas de retry sem sucesso
- **Processo**: Job vai para `DLQ` para análise manual ou reprocessamento

### **TTL (Time To Live)**
Tempo de vida de dados no cache:
- **Idempotency keys**: 1 hora
- **Status cache**: 5 minutos (balance entre performance e fresh-ness)
- **Função**: Limpeza automática de dados antigos
