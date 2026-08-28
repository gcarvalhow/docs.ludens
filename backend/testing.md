# Testes e CI

> **Responsável:** QA — Adrian Cesar Gonçalves (testes) e DevOps — Gabriel Carvalho (CI) · **Aprovação:** PO (Gabriel Carvalho) · **Última revisão:** 2026-08-28 · **Status:** vigente
> Versão operacional do [Acordo de Manutenibilidade §4 e §6.3](../team/maintainability.md).

Cobre a estratégia de testes automatizados do backend (`api.ludens`) e o pipeline
de **integração contínua** que roda esses testes. Deploy contínuo (CD) fica de
fora por enquanto — ver [gaps](#gaps).

## Foco primário — camada de domínio do backend

Os testes automatizados focam nas **regras de negócio da camada de domínio do
backend** (DDD):

- reserva e compra de ingressos;
- controle de disponibilidade de assentos ([RN05](../requirements/business-rules.md#rn05--consistência-de-disponibilidade));
- cálculo de valores (inteira/meia-entrada);
- processamento de pedidos.

**Cálculo de valores por tipo de ingresso:** inteira usa preço cheio;
meia-entrada aplica metade. Por decisão do PO
([RN04](../requirements/business-rules.md#rn04--meia-entrada)), a emissão da
meia-entrada **não** exige o número do documento de estudante — os testes devem
refletir isso.

O domínio é testado sem banco nem HTTP — instancia o agregado, chama o método,
verifica estado e eventos acumulados. Casos de uso e repositórios usam banco de
teste (Postgres em contêiner), sem mockar SQLAlchemy. Cada funcionalidade traz
seus próprios casos de teste na sua spec; esta página fixa só a abordagem.

## Roteiro de testes antes de cada entrega

Antes de cada entrega oficial, o QA executa um roteiro cobrindo o **fluxo
principal** e as regras críticas:

```text
busca do espetáculo → seleção da sessão e do ingresso → reserva → pagamento → confirmação
```

Casos obrigatórios no roteiro:

- Compra de inteira (preço cheio) e de meia-entrada (metade do valor).
- Reserva que expira e devolve os ingressos à disponibilidade
  ([RN03](../requirements/business-rules.md#rn03--expiração-da-reserva)).
- Duas compras concorrentes que, juntas, não podem exceder a capacidade
  ([RN05](../requirements/business-rules.md#rn05--consistência-de-disponibilidade)).
- Falha de pagamento libera a reserva.
- Limite de ingressos por CPF é respeitado
  ([RN01](../requirements/business-rules.md#rn01--limite-de-ingressos-por-cpf)).

## Registro de bugs

Todo bug encontrado é documentado com:

| Campo | Conteúdo |
| --- | --- |
| Passos para reproduzir | Sequência numerada |
| Resultado esperado | O que deveria acontecer |
| Resultado obtido | O que aconteceu |
| Severidade | Alta / Média / Baixa |
| Evidência | Print, log ou stack trace |

**Bugs de alta severidade** (bloqueiam compra/pagamento ou expõem dados) impedem
o fechamento das tarefas e do ciclo.

## Ferramentas

- Backend: **Pytest** (`pytest -q`), rodando na pipeline.
- Frontend: lint como portão mínimo hoje; testes de componente a definir (fora do
  escopo deste documento — ver a futura pasta `frontend/`).

## Integração contínua (CI)

Cada repositório de código tem seu próprio workflow de CI, disparado em `push` e
`pull_request` para `master`. O ambiente conteinerizado (Docker) é o padrão de
execução — a aplicação deve construir e subir via Docker sem erros.

### `api.ludens` — Backend

| Etapa | Comando | Ferramenta |
| --- | --- | --- |
| Setup | Python 3.12 | `actions/setup-python` |
| Instalar | `pip install -e ".[dev]"` | — |
| Lint | `ruff check .` | Ruff |
| Testes | `pytest -q` | Pytest |
| Build | `docker build` do backend | Docker |

### `web.ludens` — Frontend

| Etapa | Comando | Ferramenta |
| --- | --- | --- |
| Setup | Node 20 | `actions/setup-node` |
| Instalar | `npm ci` (fallback `npm install`) | — |
| Lint | `npm run lint` | ESLint + Prettier |
| Build | `npm run build` | Vite |

### `docs.ludens` — Documentação

Sem CI obrigatório (repositório só de conteúdo). Recomenda-se rodar localmente
antes do commit:

```bash
npx markdownlint-cli2 "**/*.md"
npx markdown-link-check README.md   # ou lychee
```

## Portões de merge

- Lint verde (Ruff / ESLint).
- Testes verdes (Pytest).
- Build Docker sem erros.
- Pelo menos **1 aprovação** de outro desenvolvedor no Pull Request.

## Gaps

- **Deploy contínuo (CD)**: não definido. Hoje só há CI (lint + testes + build).
- **Testes de frontend**: pendentes de definição, serão tratados na pasta
  `frontend/`.
- **Cobertura mínima**: não há meta numérica de cobertura como portão — apenas a
  exigência qualitativa de "testes relevantes criados/atualizados" do
  [Definition of Done](../team/quality.md).
