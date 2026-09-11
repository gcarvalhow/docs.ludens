---
status: approved
domain: identity
created_at: 2026-09-11
approved_at: 2026-09-11
---

# Convite de administrador por link

## 1. Visão da feature

Um administrador já autenticado gera um link de convite dentro da própria
plataforma e o entrega a quem vai assumir a administração. A pessoa convidada
abre o link, completa seu cadastro (nome, e-mail, senha) e passa a ter acesso
administrativo — sem que ninguém precise mexer em servidor ou banco de dados
pra isso acontecer.

É o que transforma "adicionar um administrador" de uma tarefa técnica, feita
por quem mantém a infraestrutura, numa ação que o próprio time do teatro
resolve sozinho, dentro do produto.

## 2. Problema que resolve

Hoje a única forma de criar uma conta de administrador é um script rodado
diretamente no servidor — ver
[identity-auth § escopo negativo](../identity-auth/spec.md#6-o-que-não-é-escopo-negativo).
Isso significa que toda vez que a composição da equipe do teatro muda (alguém
novo assume o cadastro de espetáculos, alguém sai), o administrador atual não
consegue resolver isso sozinho: precisa pedir a quem tem acesso técnico à
infraestrutura para rodar o script por ele.

Não é um problema de alta frequência — um teatro comunitário não troca de
equipe toda semana — mas toda vez que acontece, cria uma dependência externa
ao produto para uma operação que deveria ser interna a ele: dar acesso
administrativo a alguém de confiança.

## 3. Para quem é

- **Beneficiário direto:** o administrador já autenticado, que hoje não tem
  como conceder acesso administrativo a mais ninguém sem sair do produto.
- **Beneficiário indireto:** a pessoa convidada (nova administradora), que
  ganha uma forma de entrada direta e sem fricção; e quem mantém a
  infraestrutura, que deixa de ser acionado para uma tarefa operacional do
  teatro.

Acontece fora da jornada do comprador — é uma ação interna, administrativa,
que ocorre quando a equipe do teatro muda.

## 4. Como melhora a experiência atual

**Antes:** criar um administrador depende de alguém com acesso ao servidor;
o administrador do teatro não tem nenhuma ação própria para isso.

**Depois:** o administrador gera um link dentro da plataforma e o entrega à
pessoa (por e-mail, WhatsApp, como preferir); ela mesma completa o cadastro e
passa a ter acesso. Ninguém de fora do teatro precisa ser acionado.

## 5. Como se conecta com o produto existente

**Dependências obrigatórias:**
[`identity-user-management`](../identity-user-management/spec.md) — a criação
da conta administrativa em si (nome, e-mail, senha) usa o mesmo mecanismo de
criação de conta definido ali; o cadastro da pessoa convidada segue as mesmas
regras de senha.
[`identity-auth`](../identity-auth/spec.md) — depois de criada, a conta é
autenticada imediatamente pelo mesmo mecanismo de sessão usado no login.
[`notification-transactional-email`](../notification-transactional-email/spec.md)
— o envio do link de convite por e-mail (decidido em `logic.md`) reaproveita
esse serviço. **Atenção:** essa spec está aprovada mas ainda não implementada
no backend (só existem os módulos `catalog` e `identity` hoje) — esta feature
não pode ir ao ar antes dela.

**O que habilita:** a administração da plataforma deixa de depender de acesso
à infraestrutura para crescer o time de administradores — abre caminho para o
teatro operar essa troca de equipe sem suporte técnico externo.

**Posição no produto:** complementar a RF09, mesmo domínio (`identity`).
Fecha a lacuna que a própria spec de `identity-auth` já registrava como fora
do fluxo público.

**RF/RN cobertos:** nenhum RF do conjunto aprovado em 2026-08-28 cobre esta
feature diretamente — é escopo novo, decidido pelo PO como extensão natural
de RF09 e tratado como N1 (entra junto do MVP, não como evolução N2/N3).

## 6. O que não é (escopo negativo)

- **Não inclui** hierarquia entre administradores — todo administrador tem as
  mesmas permissões e qualquer um pode gerar um convite; não existe um
  "administrador principal" com mais poder que os demais.
- **Não inclui** mais de um convite ativo ao mesmo tempo — gerar um novo link
  invalida automaticamente qualquer convite anterior que ainda não tenha sido
  usado. Existe sempre no máximo um convite pendente.
- **Não inclui** cancelamento manual do link antes do prazo — se um link foi
  gerado por engano ou enviado à pessoa errada, a forma de invalidá-lo é
  deixar o prazo vencer ou gerar um novo (que o substitui). Pode virar
  melhoria futura se o uso real mostrar essa necessidade.
- **Não inclui** compromisso de envio automático por e-mail — o administrador
  recebe o link e decide como entregá-lo; se a plataforma deve enviar o
  e-mail automaticamente fica para a definição de regras em `logic.md`, não é
  um compromisso fechado aqui.
- **Não inclui** gestão do ciclo de vida do administrador depois de criado
  (remover acesso, redefinir permissões) — esta feature cobre só a entrada de
  uma nova conta administrativa.

## 7. Custos adicionais

Nenhum custo externo novo identificado. Reaproveita a base de conta e senha
já paga por `identity-auth`; se a entrega do link por e-mail for adotada em
`logic.md`, usa o mesmo serviço de e-mail transacional já contratado para
`notification-transactional-email`, sem provedor novo.

## 8. Decisões tomadas

| Ponto | Decisão |
| --- | --- |
| Nível de escopo | N1 — entra junto do MVP, substitui o script de servidor como forma de criar administrador. |
| Quem pode convidar | Qualquer administrador autenticado — não há hierarquia entre administradores. |
| Convites simultâneos | No máximo um convite ativo por vez; gerar um novo expira automaticamente o anterior, se ainda não usado. |
| Cancelamento manual | Fora desta versão — o link expira sozinho no prazo; não há ação de cancelar antes disso. |
| Uso do link | Único — depois que a conta é criada, o mesmo link não pode ser usado de novo. |
| Validade do link | 48 horas — decidido em `logic.md`. |
| Envio do link | Automático por e-mail — a plataforma envia sozinha, sem o administrador ver o link bruto. Decidido em `logic.md`. |

## 9. Perguntas abertas

Nenhuma. As duas questões em aberto desta seção foram fechadas em `logic.md`
(validade e forma de envio do link) e estão refletidas na tabela acima.
