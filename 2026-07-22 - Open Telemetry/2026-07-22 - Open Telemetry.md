# Observabilidade com OpenTelemetry

> Documento da disciplina **Desenvolvimento de Sistemas Corporativos (DSC)** — UFPB
> Guia conceitual + tutorial prático para instrumentar o projeto da sua equipe.

Este documento tem três partes:

1. **O que é telemetria** — o problema que ela resolve.
2. **O que é o OpenTelemetry (OTel)** — o padrão que vamos usar.
3. **Tutorial** — como instrumentar o código da sua equipe (Java, Python ou Node/TypeScript) e visualizar os dados num painel.

Ao final há uma lista de **exercícios/entregáveis**.

---

## 1. O que é telemetria?

Quando um sistema está rodando em produção e alguém diz *"o site está lento"* ou *"deu erro ao finalizar o pedido"*, como você descobre o que aconteceu? Olhar o código não ajuda — o código está parado; o problema está no **comportamento em execução**.

**Telemetria** é a prática de o próprio sistema **emitir dados sobre o que está fazendo enquanto executa**, para que possamos observá-lo de fora sem precisar parar, recompilar ou adivinhar. A palavra vem do grego *tele* (à distância) + *metron* (medida): medir à distância.

A capacidade de entender o estado interno de um sistema apenas pelos dados que ele emite chama-se **observabilidade**. Um sistema é "observável" quando você consegue responder perguntas novas sobre ele — inclusive perguntas que você não previu — sem ter que adicionar código novo toda vez.

### Os três sinais da telemetria

A telemetria moderna se organiza em três tipos de dados, chamados de **sinais** (*signals*):

| Sinal | Pergunta que responde | Exemplo no seu projeto |
|-------|-----------------------|------------------------|
| **Traces** (rastros) | *"Por onde este pedido passou e onde gastou tempo?"* | requisição HTTP → controller → consulta no banco → renderização da página |
| **Metrics** (métricas) | *"Quantas requisições por segundo? Qual a latência p95? Quanta memória?"* | throughput, taxa de erro, uso de heap da JVM/processo |
| **Logs** | *"O que exatamente foi registrado neste instante?"* | `"pagamento recusado para o pedido 4213"` |

Os três se complementam. As **métricas** te avisam que *algo* está ruim (a latência subiu). Os **traces** te mostram *onde* (90% do tempo estava numa query SQL). Os **logs** te dizem *o quê* (a query travou esperando um lock).

### O conceito central: o *trace* distribuído

O sinal mais revelador é o **trace**. Um trace representa **a jornada completa de uma requisição** através do sistema. Ele é composto de **spans** (intervalos), onde cada span é uma unidade de trabalho com início, fim e duração:

```
Trace: GET /produtos/42                                    (total: 812 ms)
├─ span: HTTP GET /produtos/42                             [████████████] 812 ms
│  ├─ span: ProdutoController.detalhe()                    [███████████ ] 790 ms
│  │  ├─ span: SELECT * FROM produto WHERE id = ?          [█████       ] 310 ms
│  │  ├─ span: SELECT * FROM avaliacao WHERE produto = ?   [ ████       ] 250 ms   ⚠️ N+1?
│  │  └─ span: render template "produto.html"             [      ███   ] 180 ms
```

Olhando esse "gráfico de cascata" (*waterfall*), você **vê** onde o tempo foi gasto — sem chutar. Descobre gargalos (uma query sem índice), erros (um span marcado em vermelho) e problemas de arquitetura (o clássico N+1 de queries).

---

## 2. O que é o OpenTelemetry?

Historicamente, cada ferramenta de monitoramento tinha seu próprio formato e sua própria biblioteca. Se você instrumentava seu código para a ferramenta X e depois queria trocar pela ferramenta Y, tinha que reinstrumentar tudo. Isso prendia o desenvolvedor (*vendor lock-in*).

O **OpenTelemetry** (abreviado **OTel**) é um **padrão aberto e independente de fornecedor** para gerar, coletar e exportar telemetria. É um projeto da CNCF (a mesma fundação do Kubernetes) e hoje é o padrão de facto da indústria. Ele define:

- **Uma API e SDKs** para cada linguagem (Java, Python, JavaScript/Node, Go, .NET, Rust…), com a mesma semântica em todas.
- **Convenções semânticas** — nomes padronizados de atributos (`http.request.method`, `db.system`, `server.address`…), para que um trace signifique a mesma coisa em qualquer linguagem.
- **O protocolo OTLP** (*OpenTelemetry Protocol*) — o protocolo pelo qual a telemetria viaja, tipicamente via gRPC (porta **4317**) ou HTTP (porta **4318**).

**A grande ideia:** você instrumenta seu código **uma vez**, seguindo o padrão OTel, e pode enviar os dados para **qualquer** backend compatível — Jaeger, Grafana, Prometheus, Datadog, New Relic, etc. — trocando apenas uma configuração, sem mexer no código.

### Como as peças se encaixam

```
┌────────────────────┐      OTLP        ┌──────────────────────┐        ┌──────────────┐
│  Sua aplicação      │  (:4317 / :4318) │  Backend de           │        │  Dashboard    │
│  + instrumentação   │ ───────────────> │  observabilidade      │ ─────> │  (Grafana)    │
│  OpenTelemetry      │  traces/metrics  │  (coleta + armazena)  │        │  você explora │
│                     │  /logs           │                       │        │  os dados     │
└────────────────────┘                  └──────────────────────┘        └──────────────┘
```

Para esta disciplina vamos usar como backend a imagem **`grafana/otel-lgtm`**: um único container que já traz, pré-configurados, o coletor OTLP + os bancos de telemetria (Prometheus para métricas, Tempo para traces, Loki para logs) + o **Grafana** como interface visual. LGTM = **L**oki, **G**rafana, **T**empo, **M**imir. É a forma mais simples de ter um ambiente completo de observabilidade rodando localmente.

### Instrumentação automática vs. manual

Há dois níveis de instrumentação, e você vai usar os dois:

- **Automática (*zero-code*)** — um agente/biblioteca do OTel intercepta as bibliotecas que você já usa (servidor HTTP, driver do banco, cliente HTTP) e gera spans **sem você escrever nenhuma linha**. É o ponto de partida.
- **Manual** — você adiciona spans e atributos ao seu **código de negócio** (ex.: "calcular frete", "aplicar cupom") para enriquecer os traces com o que só você sabe que é importante.

---

## 3. Tutorial: instrumentando o projeto da sua equipe

> **Pré-requisito:** Docker e Docker Compose instalados.

O tutorial tem 4 passos, iguais para todas as equipes:

1. Subir o backend de observabilidade (LGTM).
2. Ligar a instrumentação **automática** na sua aplicação (a parte que muda por linguagem).
3. Gerar tráfego e **ver os dados no Grafana**.
4. Adicionar instrumentação **manual** no seu código de negócio.

### Passo 1 — Subir o backend (igual para todas as linguagens)

Adicione o serviço abaixo ao `docker-compose.yml` do seu projeto (ou crie um `docker-compose.observability.yml`):

```yaml
services:
  otel-lgtm:
    image: grafana/otel-lgtm
    container_name: otel-lgtm
    ports:
      - "3000:3000"   # Grafana (interface web)
      - "4317:4317"   # OTLP via gRPC
      - "4318:4318"   # OTLP via HTTP
```

Suba com:

```bash
docker compose up -d otel-lgtm
```

Abra **http://localhost:3000** (usuário/senha padrão: `admin` / `admin`). Ainda não haverá dados — falta instrumentar a aplicação.

### Configuração comum a todas as linguagens

Independentemente da linguagem, a instrumentação é controlada pelas **mesmas variáveis de ambiente**. Se sua app roda em container, aponte para o nome do serviço (`otel-lgtm`); se roda na sua máquina, use `localhost`.

```bash
OTEL_SERVICE_NAME=eqNN-nomedoprojeto          # troque pelo nome da SUA equipe/projeto
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-lgtm:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_TRACES_EXPORTER=otlp
OTEL_METRICS_EXPORTER=otlp
OTEL_LOGS_EXPORTER=otlp
```

> `OTEL_SERVICE_NAME` é o nome que identifica seu serviço no Grafana. Use o identificador da sua equipe (ex.: `eq07-biblioteca`) para não se confundir.

---

### Passo 2 — Instrumentação automática

Siga **apenas a seção da linguagem do seu projeto**.

#### 🟧 Java (Javalin, Spring Boot, ou qualquer app JVM)

O OTel para Java usa um **agente Java** (`.jar`) que se acopla à JVM e instrumenta automaticamente o servidor HTTP (inclusive o Jetty embutido do Javalin), o **JDBC** (toda query ao PostgreSQL vira um span), chamadas HTTP de saída, e métricas da JVM.

1. Baixe o agente (uma vez):

```bash
curl -L -o opentelemetry-javaagent.jar \
  https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar
```

2. Rode sua aplicação com o agente anexado — **sem alterar o código**:

```bash
java -javaagent:opentelemetry-javaagent.jar -jar target/app.jar
```

Em um **Dockerfile**, isso vira:

```dockerfile
COPY opentelemetry-javaagent.jar /app/otel.jar
ENTRYPOINT ["java", "-javaagent:/app/otel.jar", "-jar", "/app/app.jar"]
```

Pronto. Suba a aplicação e vá para o **Passo 3**.

#### 🟦 Python (Flask, FastAPI, Django…)

O OTel para Python usa um wrapper de linha de comando que injeta a instrumentação nas bibliotecas detectadas (framework web, `psycopg`/driver do banco, `requests`).

1. Instale o distro e detecte automaticamente os instrumentadores das libs que você usa:

```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install
```

2. Rode sua aplicação através do wrapper `opentelemetry-instrument` — **sem alterar o código**:

```bash
opentelemetry-instrument python app.py
```

> Se você usa Gunicorn/Uvicorn, o comando vira, por exemplo:
> `opentelemetry-instrument gunicorn app:app`

Suba a aplicação e vá para o **Passo 3**.

#### 🟩 Node.js / TypeScript (Express, Fastify, Koa…)

O OTel para Node usa um módulo de registro carregado **antes** do seu código, que instrumenta automaticamente o framework web, o driver `pg` do PostgreSQL e o módulo `http`.

1. Instale as dependências:

```bash
npm install @opentelemetry/api @opentelemetry/auto-instrumentations-node @opentelemetry/exporter-trace-otlp-http
```

2. Rode carregando o registro de auto-instrumentação **antes** da sua aplicação — sem alterar o código:

```bash
node --require @opentelemetry/auto-instrumentations-node/register app.js
```

> Para **TypeScript**, aponte para o JavaScript compilado (ex.: `dist/app.js`). Alternativamente, defina a variável `NODE_OPTIONS="--require @opentelemetry/auto-instrumentations-node/register"` e mantenha seu comando de start normal.

Suba a aplicação e vá para o **Passo 3**.

---

### Passo 3 — Gerar tráfego e ver os dados

1. Com a app e o `otel-lgtm` no ar, **use o sistema**: navegue por algumas páginas, faça um cadastro, uma busca — qualquer coisa que gere requisições.
2. Abra o Grafana em **http://localhost:3000**.
3. Menu **Explore** → selecione a fonte de dados **Tempo** (traces).
4. Clique em **Search** e busque pelo seu `service.name` (ex.: `eq07-biblioteca`). Os traces das suas requisições vão aparecer.
5. Clique em um trace para abrir a **cascata (waterfall)**. Observe:
   - Qual span demorou mais?
   - Consegue ver a(s) query(ies) SQL como spans filhos?
   - Alguma requisição está marcada com erro (vermelho)?
6. Vá em **Dashboards** para ver as **métricas** (latência, throughput, uso de memória) já coletadas automaticamente.

> 💡 **Provoque um problema de propósito** para aprender a diagnosticá-lo: adicione um `Thread.sleep`/`time.sleep`/`await sleep` de 500 ms em um handler, ou rode uma query pesada, e observe como isso aparece na cascata.

---

### Passo 4 — Instrumentação manual (enriquecendo o trace)

A auto-instrumentação enxerga bibliotecas (HTTP, SQL), mas **não conhece a sua regra de negócio**. Adicionar spans manuais no seu código é o que torna a telemetria realmente útil — e é parte da entrega.

#### 🟧 Java

Adicione a dependência da API (com o agente, o SDK já está presente; você só precisa da API e, opcionalmente, das anotações):

```xml
<dependency>
  <groupId>io.opentelemetry</groupId>
  <artifactId>opentelemetry-api</artifactId>
</dependency>
<dependency>
  <groupId>io.opentelemetry.instrumentation</groupId>
  <artifactId>opentelemetry-instrumentation-annotations</artifactId>
</dependency>
```

Forma mais simples — anotação em um método:

```java
import io.opentelemetry.instrumentation.annotations.WithSpan;
import io.opentelemetry.instrumentation.annotations.SpanAttribute;

@WithSpan("calcula-frete")
public BigDecimal calcularFrete(@SpanAttribute("cep") String cep) {
    // ... o agente cria e finaliza o span automaticamente
}
```

Forma programática — controle fino (ex.: medir a renderização do template):

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.context.Scope;

Span span = GlobalOpenTelemetry.getTracer("app")
        .spanBuilder("render-template")
        .setAttribute("template", "produto.html")
        .startSpan();
try (Scope s = span.makeCurrent()) {
    templateEngine.process("produto", contexto, writer);
} catch (Exception e) {
    span.recordException(e);   // registra o erro no trace
    throw e;
} finally {
    span.end();
}
```

#### 🟦 Python

```python
from opentelemetry import trace

tracer = trace.get_tracer("app")

def calcular_frete(cep: str):
    with tracer.start_as_current_span("calcula-frete") as span:
        span.set_attribute("cep", cep)
        # ... sua lógica
        return valor
```

Para registrar um erro no span:

```python
try:
    processar_pagamento(pedido)
except Exception as e:
    span.record_exception(e)
    raise
```

#### 🟩 Node.js / TypeScript

```typescript
import { trace } from '@opentelemetry/api';

const tracer = trace.getTracer('app');

function calcularFrete(cep: string) {
  return tracer.startActiveSpan('calcula-frete', (span) => {
    span.setAttribute('cep', cep);
    try {
      // ... sua lógica
      return valor;
    } catch (e) {
      span.recordException(e as Error);
      throw e;
    } finally {
      span.end();
    }
  });
}
```

Depois de adicionar spans manuais, gere tráfego de novo e confira no Grafana: seus spans de negócio (`calcula-frete`, `render-template`…) agora aparecem **aninhados** dentro do trace da requisição HTTP, com os atributos que você definiu.

---

## Exercícios / Entregáveis

Registre no relatório da equipe (com **prints** do Grafana):

1. **Backend no ar** — print do Grafana rodando, com o `service.name` da sua equipe aparecendo no Tempo.
2. **Trace de uma operação real** — escolha uma funcionalidade relevante do seu sistema (ex.: finalizar pedido, cadastrar item) e capture a cascata completa do trace. Identifique qual etapa consome mais tempo.
3. **Query SQL visível** — mostre um span de consulta ao PostgreSQL dentro de um trace e diga qual tabela/operação ele representa.
4. **Instrumentação manual** — adicione **pelo menos 2 spans manuais** em regras de negócio do seu código e mostre-os aninhados no trace.
5. **Diagnóstico** — provoque (ou encontre) uma operação lenta, mostre-a no trace e explique, com base na telemetria, **onde** está o gargalo e **o que** você faria para resolvê-lo.
6. **Atributo customizado** — adicione ao menos um atributo de negócio útil a um span (ex.: `pedido.valor`, `usuario.tipo`) e explique por que ele ajuda na investigação.

---

## Para aprofundar

- Documentação oficial: <https://opentelemetry.io/docs/>
- Convenções semânticas: <https://opentelemetry.io/docs/specs/semconv/>
- Instrumentação por linguagem: <https://opentelemetry.io/docs/languages/>
- Imagem LGTM usada aqui: <https://github.com/grafana/docker-otel-lgtm>

---

*Documento gerado para a disciplina DSC — UFPB. Dúvidas, tragam para os encontros ou para o canal da disciplina no Discord.*