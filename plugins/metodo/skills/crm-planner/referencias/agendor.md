# Agendor CRM — Base de conhecimento

Levantamento feito em 12/11/2026 a partir de três fontes:

- **Central de ajuda oficial** (ajuda.agendor.com.br): 256 páginas mapeadas e os artigos centrais lidos na íntegra.
- **Documentação da API v3** (`https://api.agendor.com.br/v3/swagger.json`).
- **Leitura de uma conta real pela API**: conta nova, ainda com a configuração padrão.

Este documento é a base para criar uma skill sobre o Agendor.

---

## 1. O que é o Agendor

CRM brasileiro voltado a equipes de vendas internas e externas, com foco em venda consultiva B2B. É formado por três produtos que se conectam:

| Produto | Papel |
|---|---|
| **Agendor CRM** | Núcleo: clientes, negócios, funis, tarefas, relatórios e automações. |
| **Agendor Chat** | Atendimento por WhatsApp (API oficial da Meta ou QR Code) e chat do site, com distribuição de conversas e automações. Integra ao CRM. |
| **Ava** | Agente de IA que atende o vendedor no próprio WhatsApp. Cadastra e atualiza dados no CRM, agenda tarefas, resume conversas e lembra follow-ups. Não fala com o cliente final. |

Há ainda aplicativos para Android e iOS e um serviço de geração de leads (base de empresas comprada em parceria com a Speedio).

---

## 2. Modelo de dados (como as peças se conectam)

```
Conta
 ├── Usuários ──(pertencem a)── Grupos (permissões)  e  Equipes (privacidade)
 ├── Empresas (PJ) ──< Pessoas (contatos da empresa)
 │      │                 │
 │      └──────┬──────────┘
 │             ▼
 ├── Negócios  (sempre vinculados a uma Empresa ou Pessoa)
 │      ├── Funil → Etapa (sequência)
 │      ├── Status: em andamento | ganho | perdido (+ motivo de perda)
 │      ├── Produtos (com preço, quantidade, impostos)
 │      ├── Responsável + usuários com acesso
 │      └── Propostas (PDF gerado a partir de modelos)
 ├── Tarefas / Atividades (ligadas a Empresa, Pessoa ou Negócio)
 │      └── tipos: Visita, Reunião, Ligação, E-mail, Proposta, WhatsApp
 └── Cadastros auxiliares: Categorias, Origens, Setores, Motivos de perda,
     Produtos, Categorias de produto, Moedas, Métricas, Campos customizados
```

### Entidades principais

**Empresa** (PJ): nome fantasia, razão social, CNPJ (com preenchimento automático), e-mail, site, endereço, contatos, categoria, origem, setor, ranking de 0 a 5 estrelas, responsável, usuários com acesso, produtos de interesse e campos customizados.

**Pessoa** (PF ou contato de uma empresa): nome, CPF, cargo, aniversário, empresa vinculada, contatos (e-mail, celular, WhatsApp, redes sociais), endereço, categoria, origem, ranking, responsável e campos customizados.

**Negócio** (oportunidade):
- Dados básicos: título, valor (multimoeda, com câmbio para BRL), produtos e ranking de 0 a 5.
- Datas: início, conclusão prevista ou real, `wonAt` e `lostAt`.
- Situação: funil, etapa, status e motivo de perda.
- Pessoas: autor, responsável e usuários com acesso.
- Outros: campos customizados, e-mail próprio (para registrar e-mails por cópia oculta) e "negócio pai" (`parentDealId`, usado quando o negócio é duplicado).
- Pode ser **congelado**, com motivo e data para descongelar automaticamente.

**Tarefa / Atividade**: texto, tipo, prazo, data de conclusão, quem concluiu, usuários atribuídos e métricas. Ligações por VoIP podem trazer transcrição e resumo do áudio.

**Produto**: nome, código, categoria, preço, unidade, ativo ou inativo, imagem, descrição, impostos e taxas, e campos customizados.

### Regras de cadastro
- O campo **Nome** é obrigatório e não pode ser ocultado em nenhum formulário.
- Todo **Negócio** precisa de uma Empresa ou Pessoa vinculada.
- Só podem ser marcados como obrigatórios no formulário: Nome, CNPJ/CPF, E-mail e telefones. A obrigatoriedade de outros campos é feita pela *qualificação obrigatória* (seção 4).
- O sistema permite cadastros duplicados. Integrações (como o Chat) verificam duplicidade pelo telefone.

---

## 3. Funis e etapas

- A conta nasce com o **Funil de Vendas padrão**, com as etapas Contato, Envio de proposta, Follow-up e Fechamento. Esse funil **não pode ser excluído** nem deixar de ser o padrão, mas suas etapas podem ser editadas.
- Limites: até **30 funis** por conta e **20 etapas** por funil. Cada funil tem nome e uma sigla de 3 letras (ex.: "POS").
- Criar funis e personalizar etapas exige plano pago. O plano Gratuito opera com um único funil.
- Ao editar etapas que já têm negócios, o sistema pede para fazer a correspondência entre as etapas antigas e as novas.
- **Tempo limite na etapa** (Performance e Corporativo): limite de dias por etapa. Quando é ultrapassado, o card fica vermelho e mostra um contador de dias parado.
- Negócios podem ser **transferidos em massa** entre funis e **duplicados** para outro funil (ex.: venda → pós-venda).
- Painel de **movimentações entre funis** para analisar a passagem de negócios de um processo para outro.
- Na API, a mudança de etapa usa o **número de sequência** da etapa (começa em 1), e não o ID. Se o funil não for o padrão, é preciso informar o ID do funil.

---

## 4. Qualificação obrigatória (regras de avanço)

Recurso do plano Performance. Só administradores configuram.

- **Por etapa:** o negócio só avança se os campos definidos (de Empresa, Pessoa ou Negócio, incluindo customizados) estiverem preenchidos. Cada campo pode ter uma *pergunta de qualificação*, que aparece no alerta.
- **Por status:** é possível exigir campos para marcar o negócio como **ganho** ou **perdido**.
- Um mesmo campo só pode ser obrigatório em **uma** etapa.
- **Pular etapas** obriga a preencher os campos da etapa de destino e também os das etapas puladas.
- Para editar um campo obrigatório, só é possível mudar a pergunta. Para trocar o campo, é preciso excluir e criar de novo.
- O alerta só aparece no funil, na listagem e na ficha da versão nova da interface.
- Na API, o parâmetro `checkRequiredFields: true` aplica a mesma validação em `PUT /deals/{id}/stage` e `PUT /deals/{id}/status`.
- Ocultar um campo em um funil **não dispensa** a obrigatoriedade definida na etapa.

---

## 5. Campos customizados e formulários

- Disponíveis para Empresas, Pessoas, Negócios e Produtos. Só administradores criam, editam e excluem, e apenas em planos pagos.
- Tipos: **Texto** (até 150 caracteres), **Número inteiro** e **Número decimal** (até 12 dígitos, aceitam negativos), **Seleção única**, **Seleção múltipla** e **Data**. O nome do campo tem no máximo 30 caracteres.
- Não há limite de quantidade. O **tipo não pode ser alterado** depois de criado. Excluir uma opção de seleção apaga o histórico dessa opção.
- Existe campo **somente leitura**, útil para dados vindos de integrações.
- **Personalização de formulários** (planos Pro e acima): ocultar, reordenar (só dentro da mesma seção) e tornar obrigatório. Vale para todos os usuários. Ocultar um campo não apaga o dado.
- **Campos por funil** (formulário de Negócio): cada funil define quais campos aparecem e em que ordem.
  - Empresa/Pessoa e Nome do negócio nunca podem ser ocultados nem reordenados.
  - Valor total e Funil não podem ser ocultados, mas podem ser reordenados.
  - Ao duplicar um negócio para outro funil, ele passa a seguir a configuração do funil de destino.
  - Ainda não afeta as colunas da listagem nem os filtros.

---

## 6. Usuários, permissões e privacidade

Há três camadas que se somam:

1. **Perfis:** existem só dois. O **Administrador** vê tudo e configura a conta. O **Colaborador** vê apenas o próprio conteúdo, ou o que a privacidade permitir.
2. **Grupos** (Menu > Grupos): definem *o que* o usuário pode fazer, como exportar, editar cadastros ou ver relatórios. Cada usuário está em um único grupo, e adicioná-lo a outro grupo o retira do anterior.
3. **Privacidade** define *o que* o usuário enxerga:
   - **Conta aberta:** todos veem tudo.
   - **Conta fechada:** cada registro fica visível só para o responsável, para quem cadastrou, para os administradores e para os usuários com *acesso adicional*.
   - Cada registro pode abrir ou fechar a própria privacidade e receber acessos adicionais, inclusive em massa.
   - **Equipes:** o **Líder** vê a carteira dos membros. Os **Membros** não veem a carteira uns dos outros. Um usuário pode ser líder de uma equipe e membro de outra. Administradores não precisam estar em equipes.
- Usuários não são excluídos, são **desativados**, e o histórico deles continua nos relatórios (aparecem como "inativo"). A conta oferece autenticação em duas etapas.

---

## 7. Automações

Disponíveis nos planos Performance e Corporativo, em Menu > Configurações > Automações (a partir de modelos ou do zero).

- Cada automação tem um **gatilho**, uma ou mais **ações** e, se quiser, **intervalos** em dias, semanas ou meses. O limite é de **15 etapas** por automação, a última etapa não pode ser um intervalo e é preciso ter pelo menos uma ação.
- **Gatilhos:** negócio adicionado, editado, chegou a uma etapa, ganho, perdido ou parado há X dias; empresa adicionada; pessoa adicionada. Os gatilhos de negócio podem valer para um funil específico ou para qualquer funil, e aceitam filtros.
- **Ações:**
  - Agendar uma atividade, com opção de pular fins de semana.
  - Enviar e-mail.
  - Enviar WhatsApp (só para quem usa o Agendor Chat).
  - Criar um novo negócio em outro funil.
  - Duplicar o negócio, escolhendo se copia atividades, valor e produtos, e campos.
  - Atualizar o negócio: etapa, funil, status ou responsável.
  - Dar acesso à empresa ou pessoa vinculada, para contas com privacidade fechada.
- O campo **"Salvar em"** define se as ações seguintes atuam sobre o negócio original ou sobre o negócio recém-criado ou duplicado.
- Ao trocar o responsável, há três opções para a empresa ou pessoa vinculada: passar para o novo responsável, manter o atual e dar acesso ao novo, ou não alterar.
- **Regras de execução:**
  - As etapas rodam em sequência. Se uma falhar, a execução para ali e não tenta de novo.
  - Uma automação já iniciada **não é cancelada** se o negócio mudar de etapa ou for marcado como ganho ou perdido. Para evitar disparos indevidos, inclua um filtro de condição antes de cada envio.
  - Registros criados por **importação** ou **integração** também disparam automações. A exceção é o WhatsApp, que não dispara para listas importadas (proteção contra envio em massa).
  - Itens criados por automação aparecem com um ícone de raio e a marca "Automação".

---

## 8. Propostas comerciais

- São geradas a partir da ficha do negócio e trazem os produtos dele automaticamente (nome, preço unitário, quantidade, total, imagem, unidade e campos customizados de produto).
- Personalização: cabeçalho com dados da empresa, logomarca, imagem de fundo e seções de texto, imagem, tabela ou produtos. A validade padrão é de **30 dias**. O PDF tem pré-visualização.
- Modelos: o sistema traz 3 prontos (Produtos e Serviços, Assinaturas e Indústria). O plano **Pro** permite usar 1 modelo. **Performance** e **Corporativo** permitem até **200**. Só administradores criam e editam modelos.
- O envio é por e-mail direto do CRM e fica salvo no histórico do negócio. Cada edição gera uma nova versão sem apagar a anterior.
- E-mails e propostas aceitam **variáveis**, como nome do cliente e valor.

---

## 9. Relatórios e análises

- **Painel de vendas** e **Sumário semanal** (relatório enviado por e-mail).
- **Relatório de conversão por etapa.**
- **Relatório de métricas**, baseado nas métricas associadas às tarefas.
- **Relatórios personalizados** (Performance e Corporativo):
  - Agrupam os dados por um campo de Negócio, Empresa ou Pessoa, como responsável, produto, estado ou origem.
  - Métricas: Conversão, Negócios iniciados, Negócios ganhos, Valor ganho (R$), Ticket médio e Ciclo médio.
  - O que entra na análise é a **data de início** do negócio. O ciclo médio exige as datas de início e de conclusão preenchidas.
  - O escopo segue a privacidade: o administrador vê tudo, o líder vê suas equipes e o colaborador vê só os próprios negócios.
  - A configuração fica no link da página, que pode ser compartilhado.
- **Mapa de vendas** (geográfico) e campo de **somatório** nas listagens.
- **Sugestões inteligentes** e o **painel personalizado** (`custom_dashboard`).

---

## 10. Comunicação e integrações

- **E-mail:** envio pelo CRM com remetente verificado e configuração de DNS, automações de e-mail e envio para vários clientes de uma vez. E-mails são gravados no histórico com cópia (Cc/Cco) para o endereço exclusivo de cada registro.
- **Telefonia:** telefone virtual inteligente, VoIP próprio (VIP Solutions) e chamadas pelo celular via Windows. As ligações têm transcrição e resumo.
- **Agendor Chat ↔ CRM**, em dois modos:
  1. **Qualificação de leads:** o Chat detecta intenção comercial na conversa e *sugere* criar o contato e o negócio.
  2. **Cadastro automático:** no primeiro contato, cria a Pessoa ou Empresa (e, se configurado, o negócio no funil escolhido). A deduplicação é feita pelo telefone.
  - O vínculo fica nos dois sistemas: o CRM recebe uma nota com o link da conversa e o Chat recebe o link do cadastro.
  - Se o usuário usar o mesmo e-mail no Chat e no CRM, o responsável é atribuído automaticamente. Caso contrário, o cadastro entra sem responsável e com privacidade aberta.
  - O Chat ainda oferece distribuição de conversas entre atendentes, times, caixas de entrada, macros, etiquetas, agendamento de mensagens, integração com Meta Ads e OpenAI, e janela de 24 horas na API oficial.
- **Outras integrações:** RD Station, Pluga (que conecta a milhares de apps), Clicksign, Meta Ads, Outlook e migração a partir do Pipedrive.
- **Importação e exportação:** planilhas de Empresas, Pessoas, Negócios e Produtos (com campos customizados). É possível excluir uma importação inteira e fazer backup completo.

---

## 11. Planos (o que cada um libera)

| Recurso | Gratuito | Pro | Performance | Corporativo |
|---|---|---|---|---|
| Usuários | até 3 | ilimitado (pago por licença) | idem | idem |
| Volume | 10 mil empresas, 10 mil pessoas, 1.500 negócios | sem limite | sem limite | sem limite |
| Funis e etapas personalizados | — | ✓ | ✓ | ✓ |
| Campos customizados e formulários | — | ✓ | ✓ | ✓ |
| Integrações nativas | — | ✓ | ✓ | ✓ |
| Propostas | — | 1 modelo | até 200 modelos | até 200 modelos |
| Automações | — | — | ✓ | ✓ |
| Qualificação obrigatória | — | — | ✓ | ✓ |
| Tempo limite na etapa | — | — | ✓ | ✓ |
| Relatórios personalizados | — | — | ✓ | ✓ |
| Telefone virtual inteligente | — | — | ✓ | ✓ |
| Atendimento prioritário, gerente de conta e contrato próprio | — | — | — | ✓ |

O Agendor Chat e a Ava são contratados à parte.

---

## 12. API v3 (para a skill operar o Agendor)

- Base: `https://api.agendor.com.br/v3`. Autenticação pelo cabeçalho `Authorization: Token <token>`. O token fica em Menu > Integrações.
- Respostas trazem `data`, `meta.totalCount` e `links` (com `next` e `prev`). Erros vêm em `errors[]`. Token inválido retorna 401.
- Paginação: parâmetro `page` (começa em 1) e `per_page` (padrão 10, máximo 100).

| Recurso | Endpoints |
|---|---|
| Pessoas | `GET/POST /people`, `GET/PUT/DELETE /people/{id}`, `POST /people/upsert`, `/people/{id}/deals`, `/people/{id}/tasks` |
| Empresas | `GET/POST /organizations`, `GET/PUT/DELETE /organizations/{id}`, `POST /organizations/upsert`, `/organizations/{id}/people`, `/organizations/{id}/deals`, `/organizations/{id}/tasks` |
| Negócios | `GET /deals`, `GET/PUT/DELETE /deals/{id}`, `PUT /deals/{id}/stage`, `PUT /deals/{id}/status`, `GET /deals/movements_history`, `/deals/{id}/tasks`. Negócios são criados a partir da pessoa ou da empresa (`POST /people/{id}/deals` ou `POST /organizations/{id}/deals`) |
| Tarefas | `GET /tasks` (exige um filtro de data, como `createdDateGt`, com janela de no máximo 31 dias), `GET/PUT/DELETE /tasks/{id}`, `POST /tasks/export` |
| Produtos | `GET/POST /products`, `GET/PUT/DELETE /products/{id}`, `GET /product_categories` |
| Configuração (somente leitura) | `/funnels`, `/deal_stages`, `/deal_statuses`, `/loss_reasons`, `/lead_origins`, `/categories`, `/sectors`, `/users`, `/users/me`, `/account_currencies`, `/metrics`, `/states`, `/states/{uf}/cities`, `/voip_calls` |
| Campos customizados | `/custom_fields/organizations`, `/custom_fields/people`, `/custom_fields/deals`, `/custom_fields/products` |

**Pontos de atenção da API:**
- `PUT /deals/{id}/status` aceita os valores `ongoing`, `won` e `lost`. Para `lost`, envie `lossReason` com `{id, description}`.
- `PUT /deals/{id}/stage` usa o **número de sequência** da etapa; informe `funnel` se não for o funil padrão.
- Os tipos de tarefa são `VISITA`, `REUNIAO`, `LIGACAO`, `EMAIL`, `PROPOSTA` e `WHATSAPP`.
- Os rotas `upsert` evitam duplicidade ao integrar leads.
- A API **não cria nem altera** funis, etapas, campos customizados, categorias, origens ou motivos de perda. Essas configurações só podem ser feitas pela interface.

---

## 13. Configuração padrão de uma conta nova (observada pela API)

- **Funil:** "Funil de Vendas" (sigla VEN), com as etapas Contato, Envio de proposta, Follow-up e Fechamento.
- **Origens:** Indicação, Evento e Site.
- **Motivos de perda:** Preço, Prazo, Produto/Serviço e Desistência.
- **Categorias:** Cliente efetivo, Cliente em potencial, Concorrente, Fornecedor e Parceiro.
- **Setores:** Comércio Atacadista, Comércio Varejista, Exportadora, Importadora, Indústria em geral, Indústria fabricante de máquinas e Prestadora de serviços.
- **Recursos ativos:** funil novo, listas, formulários e perfil novos, modal de empresa, nova lista de tarefas e painel personalizado.

---

## 14. Lacunas (ainda não lidas em detalhe)

Estes artigos estão mapeados, mas não foram lidos na íntegra. Se a skill precisar deles, é preciso aprofundar:

- Congelamento de negócios.
- Datas de início e conclusão.
- Motivo de perda.
- Sugestões inteligentes.
- Filtros e temporizador das automações.
- Automações de WhatsApp.
- Metas e mapa de vendas.
- Integração com Meta Ads e RD Station.
- Webhooks: existem via integrações, mas não aparecem na API pública.
