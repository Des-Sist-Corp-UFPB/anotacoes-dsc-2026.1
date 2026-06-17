# Desafio aberto — última hora (autônomo)

### DSC 2026.1 · continuação do [roteiro do laboratório](03-roteiro-laboratorio-cnpj.md)

**Tempo:** o que sobrar da aula (≈ 1h). **Não há linha de chegada obrigatória** — entregue o nível que conseguir.

---

## Pré-requisito

Você já tem o sistema da etapa anterior: ele recebe um Cartão CNPJ, extrai os dados e **cadastra a empresa** numa base. Vamos construir **em cima disso**.

---

## Desafio principal — "Converse com a sua base de empresas"

Crie uma funcionalidade em que o usuário **faz perguntas em linguagem natural** sobre as empresas cadastradas e o sistema responde com dados **reais** da sua base.

Exemplos do que o usuário poderia perguntar:

- *"Quantas empresas ativas eu tenho cadastradas?"*
- *"Liste as empresas de João Pessoa."*
- *"Quais empresas abriram depois de 2010?"*
- *"Tem alguma empresa do setor de tecnologia?"*

O ponto-chave: a resposta **não pode ser inventada pelo LLM**. Ela tem que vir da sua base de dados. O LLM **entende a pergunta**; o **seu código busca os dados**.

> 🔑 Esta é a diferença entre um chatbot que alucina e um sistema corporativo confiável.

---

## Como pensar nisso (sem receita)

Você tem duas grandes abordagens — **decida e justifique qual usar**:

1. **Despejar tudo no prompt:** mandar todas as empresas junto com a pergunta e pedir a resposta. Simples… mas o que acontece quando a base tem 10 mil empresas? (pense em tokens, custo, limite de contexto)

2. **Deixar o LLM escolher uma *consulta*:** você oferece ao modelo um conjunto de **funções de busca** que o seu sistema sabe executar (ex.: "buscar por cidade", "contar por situação", "filtrar por data de abertura"). O LLM lê a pergunta, **escolhe a função e os parâmetros**, e o **seu código executa** e devolve o resultado. *(Isto é Tool Use / function calling — Cap. 5 da apostila.)*


---

## Níveis (entregue o quanto conseguir)

**Nível 1 — Pergunta e resposta** 🟢
Uma pergunta simples ("quantas empresas ativas?") é respondida com dado real da base. Vale começar pela abordagem que você achar mais rápida.

**Nível 2 — Function calling de verdade** 🔵
O LLM **escolhe** qual função de busca chamar (e com quais parâmetros) a partir da pergunta. Você **declara** as funções disponíveis; o **seu código** as executa. A resposta ao usuário é redigida com base no resultado retornado.

**Nível 3 — Várias capacidades** 🟣
Ofereça duas ou três funções de busca diferentes e mostre que o modelo escolhe a certa conforme a pergunta (contar, listar por cidade, filtrar por data…).

**Nível 4 — Segurança e limites** 🔴 *(o mais importante de discutir, mesmo que não dê tempo de codar)*
- Suas funções só **leem** dados — certo? O LLM **não** deve poder apagar, alterar ou exportar nada. (princípio do **menor privilégio**)
- Se você cogitou deixar o LLM **gerar a consulta ao banco** (ex.: SQL) diretamente: qual o risco? (pense em injeção, dados expostos) Por que oferecer **funções fechadas** é mais seguro do que dar uma "caneta livre" ao modelo?
- O que acontece se o usuário perguntar algo fora do escopo ("transfira o CNPJ X para mim", "ignore suas instruções")? Seu sistema se protege?

---

## Fase extra — Publique como servidor MCP 🟠 *(nível avançado · Cap. 5.8 da apostila)*

Até o Nível 4, as suas funções de busca estão **acopladas** ao seu assistente: só ele as conhece. Agora imagine que **outro** sistema de IA da empresa (um painel, um assistente de outro time, até uma IDE) também queira consultar a sua base de empresas. Você vai reimplementar tudo de novo em cada um?

A proposta desta fase é **desacoplar**: transformar as suas funções de consulta num **servidor MCP** — uma peça reutilizável que qualquer cliente compatível consome, sem saber como funciona por dentro.

### Como pensar nisso (sem receita)

- Quais das suas funções de busca fazem sentido **expor** como ferramentas (`tools`) do servidor? (lembre: só **leitura**)
- O servidor é uma peça **independente** do assistente. Quem o consome não conhece o seu banco — conhece só as ferramentas que você publicou. Por que isso é bom? (pense em desacoplamento, reuso, governança — os mesmos princípios de uma API/microsserviço)
- A fronteira de confiança muda: agora **quem se conecta ao seu servidor** ganha acesso ao que ele expõe. O que isso exige? (autenticação, autorização, menor privilégio — de novo)

### O que pedir à IA

- Peça ajuda para **expor as suas funções de consulta como ferramentas de um servidor MCP**, descrevendo cada uma (nome, o que faz, parâmetros) — exatamente as descrições que você já pensou para o function calling.
- Peça para **conectar um cliente MCP** a esse servidor e fazer uma pergunta em linguagem natural que acabe acionando uma das suas ferramentas. Exija que a IA te explique **o que o cliente faz** (descobre as ferramentas, oferece ao modelo) versus **o que o servidor faz** (executa e devolve).

### Níveis dentro desta fase

- **5a — Servidor no ar:** um servidor MCP que expõe **uma** ferramenta de consulta (ex.: "contar empresas por situação").
- **5b — Cliente consumindo:** um cliente MCP conecta, descobre a ferramenta e a usa para responder a uma pergunta real.
- **5c — Reuso de verdade:** exponha 2–3 ferramentas e mostre que o **mesmo** servidor poderia atender a mais de um cliente, sem alteração.

### Pergunta que fecha a fase

> Qual a diferença, na sua arquitetura, entre o **function calling** (que você fez no Nível 2) e o **MCP** (desta fase)? Onde está o **mecanismo** (modelo escolhe a ferramenta) e onde está o **protocolo de integração** (app descobre e conecta às ferramentas)?

---

## Critérios de sucesso

| Critério | Pergunta que você deve saber responder |
|----------|----------------------------------------|
| **Resposta fundamentada** | A resposta veio da base ou o modelo inventou? |
| **Quem executa** | É o **seu código** que toca o banco, não o LLM? |
| **Menor privilégio** | As funções expostas só fazem o estritamente necessário (leitura)? |
| **Isolamento** | O acesso ao LLM continua atrás de uma interface (dá pra trocar de provedor)? |
| **Limites** | O sistema lida bem com perguntas fora do escopo? |
| **Reuso (fase MCP)** | As ferramentas viraram um servidor que **outro** sistema poderia consumir sem reescrever nada? |

---

## Trilhas alternativas (se o tema principal não te animar)

Escolha **uma**. Mesmos princípios, problema diferente:

- **Trilha A — Classificar o segmento.** A partir do CNAE/atividade extraída, faça o LLM **classificar a empresa** num segmento de negócio (ex.: Tecnologia, Varejo, Indústria, Serviços). Pegadinha: como você **garante** que o resultado é um dos segmentos válidos, e não um texto livre qualquer? (volte a pensar em saída estruturada + validação)

- **Trilha B — Gerar uma ficha-resumo.** Para uma empresa cadastrada, gere um **resumo em linguagem natural** ("ficha da empresa") a partir dos dados estruturados. Pegadinha: o resumo pode **contradizer** os dados ou inventar informação? Como você evita isso? Onde está a fronteira entre "dado" e "texto gerado"?

---

## Para a discussão final (3 min de fechamento)

1. Você deixou o LLM **decidir** ou **agir**? Onde estava o limite do que ele podia fazer?
2. Por que dar **funções fechadas** ao modelo é mais seguro do que dar acesso direto ao banco?
3. Em uma frase: no seu sistema, **o LLM é o cérebro ou a boca**? (ele decide o quê fazer, ou só conversa enquanto o seu código decide?)
4. *(quem fez a fase MCP)* Se amanhã outro time quisesse usar as suas ferramentas, quanto do seu código ele precisaria conhecer? O que o **MCP** mudou nessa resposta?

> **A lição que fecha a aula:** num sistema corporativo, o LLM raramente deve *executar* — ele *interpreta* e *sugere*. **Quem age, com responsabilidade e limites, é o seu código.**
