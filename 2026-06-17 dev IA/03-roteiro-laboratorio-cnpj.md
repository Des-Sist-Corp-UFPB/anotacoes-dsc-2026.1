# Roteiro de laboratório — Cadastro de empresas a partir do Cartão CNPJ

### Desenvolvimento de Sistemas Corporativos (DSC 2026.1) — atividade em sala

**Duração:** ~2h (parte guiada) • **Formato:** dupla ou trio • **Entrega:** ao final da aula

> A aula tem **3h**. Esta primeira parte é **guiada**. Na última hora você segue por conta própria no **[desafio aberto](04-desafio-aberto.md)** — sem roteiro.

---

## O desafio

Construir um sistema que **cadastra empresas a partir do PDF do Cartão CNPJ**. O usuário envia o documento (um PDF do Comprovante de Inscrição e de Situação Cadastral da Receita Federal de **qualquer empresa**), e o sistema **extrai os dados, valida e registra** a empresa.

> Você **não** vai digitar os dados. O sistema deve *ler* o PDF e *entender* o conteúdo. É aí que entra o LLM.

---

## O que você vai praticar (preste atenção: são DOIS eixos)

Este laboratório treina duas habilidades ao mesmo tempo — não as confunda:

| Eixo | O que é | Onde aparece |
|------|---------|--------------|
| **1. Construir _com_ IA** | Você usa uma **IA assistente** para erguer a solução: arquitetura, esqueleto, dúvidas. | O tempo todo, em cada etapa. |
| **2. Construir um sistema que _usa_ LLM** | O **próprio sistema**, em produção, chama um modelo para extrair dados do PDF. | Etapas 4–5. |

Ou seja: você vai **conversar com uma IA para construir** um sistema que **conversa com um LLM para funcionar**.

---

## Regras do jogo — como conversar com a IA

Este roteiro **não te dá os prompts prontos**. Aprender a *pedir* é parte do exercício. Em cada etapa, você decide o que pedir. Siga estes princípios:

- **Dê contexto antes do pedido.** Diga à IA o que está construindo, o stack (Spring Boot), e onde esta peça se encaixa. Pedido sem contexto gera resposta genérica.
- **Seja específico sobre restrições.** Versão da linguagem, bibliotecas permitidas, padrões a seguir, o que **não** fazer.
- **Peça o formato que você quer.** Explicação passo a passo? Só o esqueleto? Comparação de alternativas? Diga.
- **Exija explicação, não só código.** Peça que a IA justifique cada decisão. Se você não entende o que ela gerou, **você ainda não terminou**.
- **Itere.** A primeira resposta raramente é a final. Refine, questione, peça alternativas.
- **Revise criticamente.** A IA erra com confiança. Nada entra no projeto sem você entender e concordar.
- **Nunca cole o que não entende.** A pergunta-controle é sempre: *"eu saberia explicar isso ao professor?"*

> 💡 Toda vez que o roteiro disser **"peça à IA para…"**, é você quem formula a conversa. Pense no objetivo, monte o pedido, avalie a resposta.

---

## Pré-requisitos

- Ambiente de desenvolvimento com Spring Boot funcionando (já usado na disciplina).
- Acesso à IA assistente e à API de um LLM (chave fornecida pelo professor).
- **1 a 3 PDFs de Cartão CNPJ** de empresas reais (baixe do site da Receita Federal — use empresas públicas, ex.: uma universidade, uma estatal). Tenha pelo menos um à mão antes de começar.

---

## Etapas

> Cada etapa traz: **🎯 Objetivo · 🤔 Decisões que VOCÊ toma · 🤖 O que pedir à IA · ✅ Checkpoint · 📖 Conceito**. As etapas marcadas com ⭐ são o núcleo; as marcadas com ➕ são extras se sobrar tempo.

---

### Etapa 0 — Entender o problema e o documento *(≈ 10 min)*

**🎯 Objetivo:** saber o que você vai extrair antes de programar qualquer coisa.

**🤔 Decisões que você toma:**
- Abra um Cartão CNPJ e liste **quais campos** existem (ex.: número do CNPJ, razão social, nome fantasia, data de abertura, natureza jurídica, CNAE principal e secundários, situação cadastral, endereço, porte, capital social…).
- Quais campos são **obrigatórios** para o seu cadastro? Quais são opcionais?

**🤖 O que pedir à IA:** descreva o documento e peça que ela te ajude a **listar e organizar os campos típicos** de um Cartão CNPJ e a discutir quais fazem sentido como dados de uma empresa. Não peça código ainda.

**✅ Checkpoint:** você tem uma lista clara de campos-alvo.

**📖 Conceito:** modelagem de domínio — antes de envolver LLM, saiba o que é o dado.

---

### Etapa 1 — Modelar o domínio ⭐ *(≈ 15 min)*

**🎯 Objetivo:** representar a "Empresa" e os dados extraídos como tipos do seu sistema.

**🤔 Decisões que você toma:**
- Como modelar a entidade Empresa? Quais campos viram atributos? Algum vira um tipo próprio (ex.: endereço, CNAE)?
- Como representar campos com valores fechados (ex.: situação cadastral: ATIVA, BAIXADA, …)?
- O que precisa ser **único** (pista: CNPJ)?

**🤖 O que pedir à IA:** apresente sua lista de campos e o contexto (Spring Boot, vai persistir em banco) e peça ajuda para **desenhar o modelo de dados**. Peça que ela explique as escolhas e aponte alternativas. **Decida você** o que aceitar.

**✅ Checkpoint:** modelo da Empresa definido, com tipos e o identificador único.

**📖 Conceito:** o LLM vai produzir dados que precisam encaixar num **sistema tipado** (Cap. 4 da apostila).

---

### Etapa 2 — Esqueleto do projeto e recebimento do PDF ⭐ *(≈ 20 min)*

**🎯 Objetivo:** ter uma API que **recebe o upload** do PDF.

**🤔 Decisões que você toma:**
- Que endpoint expõe o cadastro? Como ele recebe um arquivo?
- O que acontece se vier um arquivo que não é PDF, ou vazio? (pense nos erros já agora)
- Onde isolar a regra de negócio (camada de serviço) versus a entrada (controller)?

**🤖 O que pedir à IA:** peça o **esqueleto do projeto** e do **endpoint de upload**, deixando claro o stack, que será um cadastro de empresas e que o arquivo é um PDF. Exija a separação em camadas. Peça que ela explique cada dependência que sugerir — **não aceite dependências que você não entende**.

**✅ Checkpoint:** você consegue enviar um PDF e o sistema responde (mesmo que ainda não faça nada com ele).

**📖 Conceito:** camadas / MVC / REST — a funcionalidade com LLM vive **atrás** de uma fronteira limpa.

---

### Etapa 3 — Obter o conteúdo do PDF ⭐ *(≈ 15 min)*

**🎯 Objetivo:** transformar o PDF num conteúdo que o LLM possa processar.

**🤔 Decisões que você toma — esta é importante:**
- Você vai **extrair o texto** do PDF e mandar o texto ao LLM? Ou vai mandar o **documento/imagem** a um modelo multimodal? Quais as vantagens e limitações de cada caminho? (O Cartão CNPJ é um PDF de texto — isso influencia sua escolha.)
- Justifique a decisão. Não existe resposta única; existe a **decisão fundamentada**.

**🤖 O que pedir à IA:** explique o trade-off que você identificou e peça que ela **compare as abordagens** para o seu caso. Depois, peça ajuda para implementar a que você escolheu. Cobre a explicação.

**✅ Checkpoint:** dado um PDF, você obtém o conteúdo pronto para enviar ao modelo.

**📖 Conceito:** preparação da entrada — o LLM é caro; o que você manda importa (tokens, custo — Cap. 9).

---

### Etapa 4 — Extrair os dados com o LLM (saída estruturada) ⭐ *(≈ 25 min)*

**🎯 Objetivo:** o coração do desafio — fazer o LLM devolver os dados **estruturados** e tipados, não texto solto.

**🤔 Decisões que você toma:**
- Que **formato** você vai exigir do modelo para conseguir integrar ao seu sistema? (pista: algo que você consegue converter no seu modelo da Etapa 1)
- O que o modelo deve fazer com um campo que **não encontrar** no documento? Inventar? Deixar vazio? **Defina isso** — é o que separa um sistema confiável de um que alucina.
- Onde, na arquitetura, fica a chamada ao LLM? Ela deve ficar **isolada atrás de uma interface**, para você poder trocar de provedor ou usar um substituto nos testes. Por quê isso importa?

**🤖 O que pedir à IA:**
- Peça ajuda para **definir a instrução** que será enviada ao modelo de extração — descrevendo a tarefa, os campos esperados, o formato de saída e o comportamento diante de dados ausentes. (Você está projetando o **prompt do sistema**; trate-o como código — ele será revisado e versionado.)
- Peça ajuda para **isolar o acesso ao LLM atrás de uma interface** (pense: Strategy/Adapter) e para **converter a saída** no seu modelo de domínio.

**✅ Checkpoint:** ao enviar um Cartão CNPJ, você recebe os dados da empresa já como objeto do seu domínio.

**📖 Conceito:** **saída estruturada** + **prompt como artefato** + **isolamento atrás de interface** (Caps. 3, 4 e 10).

---

### Etapa 5 — Validar (e a grande decisão: LLM × determinístico) ⭐ *(≈ 15 min)*

**🎯 Objetivo:** não confiar cegamente no que o modelo devolveu.

**🤔 Decisões que você toma — esta é a pegadinha da aula:**
- O número do CNPJ tem **dígitos verificadores** com uma regra matemática exata. Você vai pedir ao **LLM** para validar isso, ou vai validar com **código determinístico**? **Por quê?**
- Quais campos são obrigatórios? O que fazer se vierem fora do esperado (valor inválido, campo faltando)?
- Se a extração falhar ou vier inconsistente, qual é o **plano B**? (rejeitar? tentar de novo? mandar para conferência humana?)

> 🔑 Princípio: **use o LLM para o que é ambíguo (entender o documento); use código para o que é exato (validar o dígito do CNPJ, garantir unicidade).** Não use um LLM para uma conta que tem resposta certa.

**🤖 O que pedir à IA:** peça ajuda para implementar a **validação do CNPJ de forma determinística** e a **validação do schema/campos obrigatórios**, além de uma estratégia de **retry/fallback** quando a saída do modelo for inválida. Peça que ela explique a regra do dígito verificador.

**✅ Checkpoint:** dados inválidos não passam; uma extração ruim não derruba o sistema nem grava lixo.

**📖 Conceito:** validação de saída, guardrails, fallback (Caps. 4 e 8); **julgamento LLM × determinístico** (Cap. 1).

---

### Etapa 6 — Persistir e evitar duplicidade ⭐ *(≈ 10 min)*

**🎯 Objetivo:** registrar a empresa de fato — uma única vez.

**🤔 Decisões que você toma:**
- O que acontece se o mesmo CNPJ for cadastrado duas vezes? Como impedir? (LLM ou código?)
- Atualizar o registro existente ou rejeitar o duplicado?

**🤖 O que pedir à IA:** peça ajuda para **persistir** a empresa e **garantir unicidade** do CNPJ, tratando o caso de duplicidade conforme você decidiu.

**✅ Checkpoint:** empresa salva; segundo envio do mesmo CNPJ é tratado, não duplicado.

**📖 Conceito:** de novo, **regra exata = código**, não LLM.

---

### Etapa 7 — Observabilidade, custo e segurança ➕ *(≈ 10 min)*

**🎯 Objetivo:** pensar como em produção.

**🤔 Decisões que você toma:**
- O que vale a pena **registrar** a cada extração? (modelo usado, tokens, tempo, sucesso/falha) E o que você **não** deve logar em texto claro? (dados pessoais — LGPD)
- Um PDF malicioso poderia conter texto tentando **subverter** as instruções do seu modelo (prompt injection). Como você reduz esse risco? Como trata a saída do modelo: como verdade ou como **dado não confiável** a ser validado?

**🤖 O que pedir à IA:** peça sugestões de **o que registrar** para observabilidade sem expor dados sensíveis, e como **mitigar prompt injection** num fluxo de extração de documentos.

**✅ Checkpoint:** você consegue explicar custo, rastreabilidade e riscos da sua solução.

**📖 Conceito:** observabilidade, custo, segurança e LGPD (Cap. 9).

---

### Etapa 8 — Testar com documentos reais (mini-eval) ⭐ *(≈ 15 min)*

**🎯 Objetivo:** medir se funciona — em mais de um documento.

**🤔 Decisões que você toma:**
- Teste com **vários** Cartões CNPJ diferentes. Acertou todos os campos? Onde errou?
- Como você **automatizaria** essa verificação? (pense: casos conhecidos → saída esperada → comparação). Por que um único teste não basta com um componente probabilístico?

**🤖 O que pedir à IA:** peça ajuda para montar um **conjunto de casos de teste** que compare a extração com valores esperados, e para pensar em como medir a **taxa de acerto**.

**✅ Checkpoint:** você tem evidência (mais de um caso) de que o sistema funciona — e sabe onde ele falha.

**📖 Conceito:** **evals** — o "teste" da era LLM (Cap. 8).

---

## Marcos de tempo (referência)

| Tempo acumulado | Você deveria estar em… |
|----------------:|------------------------|
| 0:25 | Etapas 0–1 (domínio modelado) |
| 0:45 | Etapa 2 (recebe o PDF) |
| 1:00 | Etapa 3 (conteúdo do PDF pronto) |
| 1:25 | Etapa 4 (extração estruturada funcionando) ⭐ |
| 1:40 | Etapas 5–6 (validação + persistência) |
| 2:00 | Etapa 8 (testado) + reflexão |

> Se o tempo apertar, garanta o **caminho mínimo**: receber PDF → extrair com LLM → validar CNPJ → salvar. Etapa 7 e refinamentos ficam como extra.

---

## Entregáveis (ao fim da aula)

1. O sistema rodando: dado um PDF de Cartão CNPJ, ele cadastra a empresa.
2. Uma resposta curta (3–5 linhas) para: **o que no seu sistema é LLM e o que é código determinístico — e por quê?**
3. O registro de **um caso que o modelo errou** (ou a confirmação de que acertou todos os testados) e o que você faria a respeito.

---

## Perguntas de reflexão (para a discussão final)

1. Onde, exatamente, estava a fronteira entre "o que o LLM resolve" e "o que o código resolve"? Você moveu essa fronteira durante o exercício?
2. Você isolou o LLM atrás de uma interface? Se o professor pedisse para trocar de provedor amanhã, quanto código mudaria?
3. Como você garantiria que uma mudança no prompt não quebrou o que já funcionava?
4. Qual foi a parte mais difícil: **conversar com a IA** para construir, ou fazer o **LLM dentro do sistema** se comportar de forma confiável?
5. O que aconteceria com um PDF que **não é** um Cartão CNPJ? E com um adulterado? Seu sistema percebe?

---

## Desafios extras ➕ (se você terminar antes)

- **Sócios e quadro societário:** alguns documentos trazem os sócios. Estenda a extração para uma lista estruturada.
- **Streaming:** mostre o progresso da extração ao usuário em tempo real (pense no padrão Observer).
- **Cache:** se o mesmo PDF for enviado de novo, evite uma nova chamada ao LLM.
- **Confirmação humana:** antes de salvar, exiba os dados extraídos para o usuário revisar e confirmar (human-in-the-loop).
- **Troca de provedor:** prove na prática que sua interface permite trocar o modelo sem mexer no resto do sistema.

---

## Lembrete final

> O LLM é só **uma peça** do seu sistema — a peça que entende linguagem. Tudo ao redor (receber, validar, persistir, testar, observar) é a engenharia de sempre. O objetivo da aula não é "usar IA"; é **construir software confiável que usa IA como componente**.
