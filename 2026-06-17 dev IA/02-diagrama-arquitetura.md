# Arquitetura de referência — funcionalidade com LLM

Diagrama das camadas de uma funcionalidade corporativa apoiada em LLM (Cap. 10 da apostila). Abra no modo Excalidraw para editar.

![[arquitetura-referencia-llm.excalidraw]]

## Leitura do diagrama (de cima para baixo)

1. **Controller REST** — expõe a funcionalidade.
2. **Serviço de domínio** — a regra de negócio do sistema.
3. **Facade do LLM** — orquestra prompt + chamada + parse. Apoia-se na **camada de prompts** (templates versionados).
4. **Decorators** — cache → retry → logging/métricas, sem poluir o núcleo.
5. **Strategy de provedor** — troca de Provedor A | B | Mock; cada um com seu **Adapter HTTP**.
6. **Validação de saída** — valida contra o schema; se inválida, cai no **Fallback**.

As caixas pontilhadas à direita são responsabilidades de apoio (prompts, adapter, fallback). As setas sólidas mostram o fluxo da requisição.
