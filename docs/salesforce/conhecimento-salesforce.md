# Salesforce Sales Cloud — Base de conhecimento

Levantamento feito em 12/11/2026, só com documentação pública, para servir de base a uma skill. Fontes:

- **Página de edições e preços** do Sales Cloud (salesforce.com).
- **Trailhead:** módulos de leads e oportunidades, segurança de dados, fases e processo de vendas, previsão de vendas, regras de validação, Flow e APIs.
- **Referência de objetos** da documentação para desenvolvedores (objeto Opportunity).

> A central de ajuda (help.salesforce.com) carrega o conteúdo por JavaScript e não abriu na extração. Os temas que vieram de lá ou do conhecimento geral da plataforma estão marcados com **(complementar)** e devem ser conferidos antes de virar regra numa skill.

---

## 1. O que é o Sales Cloud

É o CRM de vendas da Salesforce, construído sobre a **plataforma Salesforce**. Essa plataforma é a mesma do Service Cloud, do Experience Cloud e dos demais produtos, e é totalmente configurável: objetos, campos, telas, regras e automações.

A diferença central em relação a CRMs como o Agendor é que **quase tudo é metadado configurável**. O administrador monta o modelo de dados e as regras do processo. Por isso o Salesforce escala para operações grandes, mas exige um administrador (ou consultoria) para implantar.

Componentes que hoje vêm junto: **Agentforce** (agentes de IA), **Slack**, **Tableau**, **Data 360** e o **app móvel**.

---

## 2. Modelo de dados (objetos padrão e relações)

```
Lead (prospect não qualificado)
  │  conversão
  ▼
Conta (Account) ──< Contato (Contact)
  │
  └──< Oportunidade (Opportunity) ──< Produtos da Oportunidade (OpportunityLineItem)
          │                                 └── vêm de uma Tabela de Preços (Pricebook2 → PricebookEntry → Product2)
          ├──< Cotação (Quote), que pode ser sincronizada com a oportunidade
          ├──< Papéis de contato e Equipe da oportunidade (com divisão de crédito)
          ├── Campanha (Campaign), para atribuir origem e ROI
          └── Contrato, Território, Score da oportunidade
Atividades: Tarefa (Task) e Evento (Event), ligadas a qualquer registro
            (o campo WhoId aponta para Lead/Contato e o WhatId para Conta/Oportunidade etc.)
Caso (Case), do atendimento, também se relaciona com Conta e Contato
```

### Lead
- É um prospect, pessoa ou empresa, que demonstrou interesse mas ainda não foi qualificado.
- Origens típicas: indicações, formulário do site (**Web-to-Lead**), parceiros, anúncios, eventos e importação.
- **Status do lead** é uma lista de opções (ex.: "Open - Not Contacted"). Um ou mais status são marcados como *convertidos*.
- **Intelligence View** é um painel dos leads com indicadores: sem atividade, ocioso (sem atividade concluída há 30 dias), sem próxima atividade, atrasado, para hoje e próximos 30 dias.
- **Conversão:** usa os dados do lead para criar (ou vincular a registros existentes) uma **Conta**, um **Contato** e, se quiser, uma **Oportunidade**.
  - É possível converter sem criar oportunidade.
  - Com *Person Accounts* (contas de pessoa física) e o lead sem empresa, ele vira uma Conta Pessoal mais a Oportunidade.
  - O nome da conta é acrescentado ao nome da oportunidade criada.
  - O responsável e o status convertido são escolhidos na conversão. Pela API, a conversão usa `convertLead()`.
  - Depois de convertido, o lead não pode mais ser editado nem usado (complementar).

### Oportunidade
- Representa uma venda em andamento. Campos-chave: **Conta**, **Fase (StageName)**, **Data de fechamento (CloseDate)**, **Valor (Amount)**, **Probabilidade (%)**, **Categoria de previsão (ForecastCategory)**, **Tipo**, **Origem do lead**, **Próximo passo**, **Campanha**, **Tabela de preços**, **Cotação sincronizada** e **Território**.
- Também tem campos de sistema: `IsClosed`, `IsWon`, histórico de mudança de valor e de data de fechamento (`OpportunityHistory`), indicadores de atividade e score.
- **Regras de comportamento:**
  - Ao definir a Fase, a **categoria de previsão** é preenchida com o padrão daquela fase, a não ser que seja informada explicitamente.
  - Regras de validação e gatilhos da oportunidade também são executados quando a alteração de um produto filho atualiza a oportunidade.
  - A divisão de crédito entre membros da equipe usa o objeto `OpportunitySplit`.
  - Ao **clonar uma oportunidade com produtos**, primeiro é preciso corrigir erros nos produtos e limpar campos únicos.

### Produtos e preços
- **Product2** é o catálogo de produtos. **Pricebook2** são as tabelas de preços: a Padrão e as personalizadas, por canal ou região. **PricebookEntry** é o preço de um produto numa tabela.
- Cada oportunidade usa **uma** tabela de preços. As linhas (`OpportunityLineItem`) têm quantidade, preço de venda e desconto, e podem ter *agendamentos* de receita ou quantidade.

---

## 3. Processo de vendas: fases, processos e tipos de registro

- **Fase (Stage)** é uma lista de opções da Oportunidade. Cada valor tem:
  - **Tipo:** Aberta, Fechada/Ganha ou Fechada/Perdida.
  - **Probabilidade** padrão em %.
  - **Categoria de previsão:** Pipeline, Best Case (melhor cenário), Commit (compromisso), Closed (fechado) ou Omitted (omitido).
- Fases padrão de exemplo: Prospecção (10%), Qualificação (10%), Análise de necessidade, Proposta/Revisão (60%), Negociação, Compromisso verbal (80%), Fechada/Ganha (100%) e Fechada/Perdida (0%).
- **Processo de vendas (Sales Process)** é um subconjunto filtrado das fases, criado a partir do "Master".
- **Tipo de registro (Record Type)** liga um processo de vendas e um layout de página a um perfil de usuário. Assim, convivem vários "funis" no mesmo objeto, como venda nova, renovação e lançamento A ou B.
- O **Path** é uma barra de fases no topo do registro com **campos-chave e orientações** por fase, o "guidance for success" (complementar).
- Boas práticas citadas pela própria Salesforce:
  - Mapear com o time o passo a passo do processo.
  - Definir o que é preciso para avançar de cada fase.
  - Definir a probabilidade de cada fase.
  - Minimizar o número de fases e padronizar os nomes.

> Comparando com o Agendor: o "funil" do Agendor equivale, no Salesforce, a **Tipo de registro + Processo de vendas + Path**.

---

## 4. Regras de negócio e qualidade de dados

- **Regras de validação:** uma fórmula que, quando dá *verdadeiro*, **bloqueia o salvamento** e mostra uma mensagem de erro, no topo da tela ou junto a um campo.
  - Servem para formatos (telefone, CPF), faixas (desconto máximo) e dependências.
  - Exemplo: exigir um campo a partir de certa fase, com `AND(ISPICKVAL(StageName,"Proposta"), ISBLANK(Campo__c))`.
  - Valem também para gravações pela API e por importação.
  - É o equivalente, mais poderoso, à **qualificação obrigatória** do Agendor.
- **Campos obrigatórios** podem ser definidos no próprio campo (vale para todo lugar) ou só no **layout de página** (vale só para a tela).
- **Listas de opções dependentes** e **tipos de registro** restringem os valores possíveis conforme o contexto.
- **Regras de duplicidade e de correspondência** avisam ou bloqueiam leads, contatos e contas duplicados (complementar).
- **Histórico de campos:** rastreia mudanças nos valores dos campos. Está disponível em todos os objetos personalizados e em parte dos objetos padrão.

---

## 5. Segurança: quem vê e faz o quê

São quatro níveis: **organização**, **objeto**, **campo** e **registro**. As permissões valem também para a API.

| Nível | Como é controlado |
|---|---|
| Organização | Usuários autorizados, política de senhas, horários e IPs de login. |
| Objeto | Criar, ler, editar e excluir cada objeto, via **perfil**, **conjuntos de permissões** e **grupos de conjuntos de permissões** (recomendado). |
| Campo | **Segurança em nível de campo**, que oculta ou deixa só para leitura, via conjuntos de permissões. |
| Registro | Padrões da organização, hierarquia de papéis, regras de compartilhamento e compartilhamento manual (abaixo). |

**Acesso a registros**, do mais restrito ao mais aberto:
1. **Padrões da organização (OWD):** o acesso básico a registros dos *outros*, definido por objeto e separado para usuários internos e externos. As opções são:
   - **Privado:** só o dono e quem está acima dele na hierarquia.
   - **Público somente leitura.**
   - **Público leitura/gravação.**
   - **Controlado pelo pai:** usado em objetos de detalhe numa relação mestre-detalhe.
   - A regra é configurar pelo usuário **mais restrito** e abrir o acesso depois.
2. **Hierarquia de papéis:** quem está acima vê os registros de quem está abaixo. Em objetos personalizados, isso pode ser desligado.
3. **Regras de compartilhamento:** exceções automáticas, por dono ou por critério, para grupos, papéis ou territórios. **Só ampliam** o acesso, nunca restringem.
4. **Compartilhamento manual:** o dono compartilha um registro específico pelo botão Compartilhar.

Recursos extras: **Equipes de conta** e **Equipes de oportunidade**, **Gestão de territórios**, **Filas** (para leads e casos) e **Grupos públicos**.

Auditoria: campos de criado por e modificado por, **histórico de login** (6 meses), **histórico de campos**, **trilha de auditoria da configuração (Setup Audit Trail)** e **Event Monitoring**.

Mudar os padrões da organização dispara um **recálculo de compartilhamento**, com aviso por e-mail ao terminar.

> Comparando com o Agendor: "privacidade aberta ou fechada" equivale aos padrões da organização. Líder e membro de equipe equivalem à hierarquia de papéis. Acesso adicional equivale ao compartilhamento manual. Grupos de permissão equivalem a perfis e conjuntos de permissões.

---

## 6. Distribuição de leads e casos (complementar)

- **Regras de atribuição de leads:** uma única regra ativa por vez, com entradas avaliadas em ordem. Cada entrada aplica um critério (estado, origem, produto) e define um usuário ou uma **fila**, e pode enviar um modelo de e-mail.
  - São aplicadas no Web-to-Lead, na importação e, se a opção estiver marcada, na criação manual.
  - Pela API, é preciso pedir explicitamente a aplicação da regra.
- **Filas:** caixas compartilhadas. Os membros "puxam" os registros da fila.
- **Regras de resposta automática:** e-mail de retorno ao lead ou ao caso.
- **Web-to-Lead:** formulário HTML gerado pelo Salesforce que cria leads, com limite diário de envios.
- Para rodízio (round-robin) ou distribuição por carga, usa-se **Flow** ou apps do AppExchange.

---

## 7. Automação (Flow)

O **Flow Builder** é a ferramenta oficial de automação sem código. Ele substitui o Workflow Rules e o Process Builder, que estão descontinuados.

**Tipos de flow:**

| Tipo | Disparado por | Uso |
|---|---|---|
| **Screen Flow** | Ação rápida, página Lightning, site | Assistentes com telas, como roteiro de qualificação ou simulador. |
| **Autolaunched** | Outro flow, Apex, API | Lógica em segundo plano, sem interface. |
| **Triggered** | Mudança de registro, agendamento, evento de plataforma | O mais usado: "se acontecer X, faça Y". |

**Flow disparado por registro:**
- Dispara na criação, na atualização, na criação ou atualização, ou na exclusão. Tem critérios de entrada.
- Momentos de execução:
  - **Atualização rápida de campo** (antes de salvar): atualiza o próprio registro ou mostra um erro. É a opção de melhor desempenho.
  - **Registros relacionados e ações** (depois de salvar): cria ou atualiza outros registros, envia e-mail, chama subflows.
  - **Assíncrono:** integrações e processos demorados.
  - **Caminhos agendados:** ações no futuro, relativas a datas do registro, como "3 dias antes da Data de fechamento".
- O **Flow Trigger Explorer** organiza a ordem de execução dos flows de um mesmo objeto.
- A ordem geral de execução no salvamento é: validações, gatilhos *before*, flows *before-save*, regras, gatilhos *after*, flows *after-save* e outras etapas. Isso importa para depurar regras conflitantes.
- Para lógica complexa existe o **Apex**, código em gatilhos e classes. Ele exige desenvolvedor e testes, com cobertura mínima para publicar em produção.

> Comparando com o Agendor: as automações do Agendor (gatilho, ações e intervalos) equivalem a um **flow disparado por registro com caminhos agendados**. No Salesforce não há o limite de 15 etapas, e é possível ramificar a lógica com decisões e laços.

---

## 8. Previsão de vendas e metas

- Cada usuário precisa da permissão **Allow Forecasting** e de estar na **hierarquia de previsão**. Essa hierarquia é gerada a partir da hierarquia de papéis (ou de territórios), com um gerente de previsão por nível.
- **Tipos de previsão:** definem o objeto base, se a medida é em **receita ou quantidade**, a data de referência (o padrão é a data de fechamento) e a hierarquia. Aceitam filtros, como separar negócio novo de renovação.
- **Categorias:** Pipeline, Best Case, Most Likely (opcional), Commit e Closed, cada uma ligada às fases.
  - **Soma por categoria única:** cada coluna soma só a própria categoria.
  - **Soma cumulativa:** cada coluna inclui as categorias seguintes. Por exemplo, Best Case soma Best Case, Most Likely, Commit e Closed.
- **Metas (quotas):** por período e tipo, mostram o percentual atingido.
- Opções extras:
  - **Envio formal** semanal da previsão.
  - **Julgamentos e ajustes** do gerente. Os ajustes não alteram as oportunidades.
  - **Colunas personalizadas**, como cobertura de compromisso.
  - **Renomear categorias.**
  - **Layout da página de previsões** no Lightning App Builder.
  - **Remapeamento entre fase e categoria.**

---

## 9. Relatórios e dashboards (complementar)

- **Tipos de relatório** definem os objetos e as relações disponíveis, como "Oportunidades com Produtos".
- Formatos de relatório:
  - **Tabular:** lista simples.
  - **Resumo:** com agrupamento por linhas.
  - **Matriz:** com agrupamento por linhas e colunas.
  - **Unido (joined):** combina blocos de tipos diferentes.
- Os relatórios aceitam filtros, fórmulas de resumo, gráficos e *snapshots* históricos.
- **Dashboards:** componentes alimentados por relatórios. Podem ser **dinâmicos** (os dados seguem quem está vendo) ou exibidos como um usuário fixo. Também podem ser agendados e assinados por e-mail.
- Os relatórios respeitam a segurança. Cada usuário vê só os registros a que tem acesso, como no Agendor.
- Análises mais avançadas ficam no **CRM Analytics** e no **Tableau**.

---

## 10. Edições e preços (página oficial, novembro de 2026)

| Edição | Preço por usuário por mês (USD) | Destaques |
|---|---|---|
| Free Suite | 0 | Gestão básica de contas, contatos, leads e oportunidades. |
| Starter Suite | 25 | Pacote para pequenas empresas, com vendas, atendimento, marketing e comércio. Slack incluso. |
| Pro Suite | 100 (cobrança anual) | Mais automação e personalização. |
| **Core** | 195 | Substitui a antiga **Enterprise**. Inclui tudo do Pro Suite mais pipeline e previsão avançados, score de oportunidade, inteligência de conversas, **APIs** e Agentforce. |
| **Advanced** | 395 | Substitui a antiga **Unlimited**. Inclui segurança completa (Backup & Recover, Security Center, Data Detect, Archive) e 1 milhão de Flex Credits. |
| **Max** | 550 | Substitui a antiga **Agentforce 1**. Agentforce sem limite de uso e 2,75 milhões de Flex Credits. |

- Em todas as edições novas vêm inclusos Agentforce, Slack, Tableau, Data 360 e Premier Success.
- Sandboxes: o **Developer** está incluso. **Developer Pro** e **Partial** variam por edição ou são comprados à parte.
- A página de preços cita como recursos de plataforma: aplicativos personalizados, perfis e layouts personalizáveis, papéis e permissões, e tipos de registro por objeto.
- **Acesso à API**: segundo o Trailhead, existe nas edições Enterprise, Unlimited, Developer e Performance, e na Professional mediante add-on. Na nomenclatura nova, a página de preços cita APIs a partir do **Core**.

---

## 11. APIs e integração

| API | Uso |
|---|---|
| **REST** (JSON ou XML, síncrona) | Criar, ler, atualizar e excluir registros, consultas **SOQL** e **SOSL**, metadados de objetos e `/limits`. A mais usada por apps e integrações leves. |
| **SOAP** (XML, WSDL) | Integrações servidor a servidor, como ERP e financeiro. |
| **Bulk API 2.0** (assíncrona, CSV) | Grandes volumes, a partir de cerca de 50 mil registros. É a base do Data Loader. |
| **Pub/Sub** (gRPC) | Eventos em tempo real: Change Data Capture e eventos de plataforma. |
| **Metadata** e **Tooling** | Ler e publicar **configuração** (objetos, campos, flows, layouts), por exemplo para levar mudanças do sandbox para a produção. |
| **UI API**, **GraphQL**, **Connect**, **Analytics**, **Apex REST/SOAP** | Interfaces personalizadas, consultas sob medida, Chatter, CRM Analytics e endpoints próprios em Apex. |

- **Autenticação:** OAuth 2.0 por meio de um *Connected App* ou *External Client App*. O usuário precisa da permissão **API Enabled**.
- **Limites:** tempo de execução por chamada, chamadas longas simultâneas e **total de chamadas em 24 horas** (varia por edição e licença). O consumo pode ser acompanhado em System Overview, no cabeçalho `Sforce-Limit-Info`, no endpoint `/limits` e com alertas de uso.
- **Diferença em relação ao Agendor:** no Salesforce, **a configuração também é acessível por API**, via Metadata e Tooling. Por isso uma skill pode, em tese, criar campos, fases, regras e flows, e não apenas dados.
- Ecossistema: o **AppExchange** é o marketplace de apps. Ferramentas de linha de comando: Salesforce CLI (`sf`) e extensões para o VS Code.

---

## 12. Salesforce × Agendor (equivalências)

| Conceito | Agendor | Salesforce Sales Cloud |
|---|---|---|
| Pré-qualificação | não existe objeto Lead (entra direto como Pessoa ou Empresa e Negócio) | **Lead**, convertido em Conta, Contato e Oportunidade |
| Empresa / Pessoa | Empresa / Pessoa | Conta / Contato (ou Conta Pessoal para pessoa física) |
| Negócio | Negócio | Oportunidade |
| Funil | Funil (até 30, com até 20 etapas cada) | Tipo de registro + Processo de vendas + Path |
| Etapa | Etapa | Fase (com probabilidade e categoria de previsão) |
| Motivo de perda | Lista fixa da conta | Campo personalizado de lista de opções, exigido por regra de validação |
| Campos obrigatórios por etapa | Qualificação obrigatória (plano Performance) | Regras de validação por fórmula (qualquer edição com personalização) |
| Campos customizados | 6 tipos | Mais de 20 tipos (fórmula, resumo de totais, pesquisa, mestre-detalhe, etc.) |
| Privacidade aberta ou fechada | por conta e por registro | Padrões da organização por objeto |
| Equipes (líder e membros) | Equipes | Hierarquia de papéis, equipes de conta e de oportunidade, territórios |
| Grupos de permissão | Grupos | Perfis e conjuntos de permissões |
| Automações | Gatilho + até 15 ações e intervalos | Flow (sem limite de etapas, com decisões e laços) + Apex |
| Distribuição de leads | Via Agendor Chat e automações | Regras de atribuição, filas e Flow |
| Propostas | Gerador de PDF com modelos | Cotações (Quote) com modelos de PDF; o CPQ e o Revenue Cloud são produtos à parte |
| Previsão de vendas e metas | Não tem módulo de previsão de vendas | Previsão colaborativa com categorias, metas, ajustes e envio formal |
| Relatórios | Painéis prontos + relatório personalizado | Construtor de relatórios, dashboards, CRM Analytics e Tableau |
| IA | Ava (no WhatsApp do vendedor) | Agentforce (agentes de vendas e de nutrição de leads) |
| WhatsApp | Agendor Chat nativo | Mensageria via Service Cloud ou apps do AppExchange |
| API de configuração | Não existe | Metadata e Tooling API |
| Implantação | O próprio gestor configura | Normalmente exige administrador ou consultoria |

---

## 13. Lacunas (próximos aprofundamentos)

- Ler os artigos da central de ajuda sobre conversão de leads, regras de atribuição e Web-to-Lead. Ela não abriu pela extração simples, então será preciso abrir as páginas uma a uma com renderização.
- Ler sobre relatórios e dashboards, Path, Kanban, cotações, Person Accounts e gestão de territórios.
- Detalhar o Agentforce para vendas: Lead Nurture, Pipeline Management e SDR.
- Conferir os limites exatos da API por edição na nomenclatura nova (Core, Advanced e Max).
- Pensar na aplicação a lançamentos imobiliários: lead por empreendimento, tipos de registro por lançamento e objetos personalizados de Empreendimento, Unidade e Reserva.
