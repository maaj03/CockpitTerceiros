# Cockpit de Atendimento — Documento de Design

**Data:** 26/10/2026
**Situação:** rascunho para validação
**Organização:** São Salvador Alimentos — Central de Serviços Compartilhados

---

## 1. Contexto e objetivo

Hoje os terceiros (consultorias e parceiros) trabalham dentro do **Freshservice**, o ITSM da empresa. Cada **frente de trabalho** tem uma **conta de agente licenciada, compartilhada** por todos os terceiros daquela frente. Essas licenças custam caro.

O **Cockpit de Atendimento** é uma aplicação web própria onde os terceiros passam a fazer todo o trabalho do dia a dia. Por trás, o Cockpit lê e grava no Freshservice usando **um único usuário de integração**. Com isso, as contas compartilhadas das frentes podem ser canceladas.

O Freshservice continua sendo a **fonte da verdade**. O Cockpit não tem banco de chamados, problemas, mudanças ou artigos. Ele guarda apenas o que é dele: usuários, permissões, configurações, registro de ações e marcações de leitura.

### 1.1 Critério de sucesso

**Cancelar as licenças de agente compartilhadas das frentes**, sem piorar o atendimento.

### 1.2 Anti-objetivo (o que seria fracasso mesmo funcionando)

- Sobrar qualquer tarefa do dia a dia que obrigue um terceiro a entrar no Freshservice. Nesse caso, a licença daquela frente não pode ser cortada.
- Perder rastreabilidade de quem fez o quê.
- Estourar o limite de API do Freshservice.

### 1.3 Ganho secundário

Hoje, no Freshservice, só se sabe que "a conta da frente X" fez algo. No Cockpit, cada terceiro entra com login individual (SSO), e cada ação fica registrada com o nome da pessoa. **A rastreabilidade melhora em relação a hoje.**

---

## 2. Escopo

### 2.1 Dentro da V1

A V1 precisa sair **completa**, permitindo uma virada total de uma vez, se a empresa decidir. A virada pode acontecer frente por frente.

1. **Chamados** (incidentes e requisições): fila, atendimento, status, campos, conversa, anexos, resolução, transferência, cancelamento, reabertura.
2. **Problemas:** abertura, atendimento, análise de causa raiz, erro conhecido, notas, tarefas, vínculos.
3. **Mudanças:** abertura, planejamento, anotações com evidências, tarefas, associações, solicitação de aprovação (Change Padrão), acompanhamento e encerramento.
4. **Base de conhecimento:** consulta, criação em rascunho com template, numeração e pasta automáticas, imagens coladas, validação por par e publicação.
5. **Administração:** usuários, mapeamento de frentes para filas e pastas, tipos de documento, configuração de campos por status.
6. **Rastreabilidade:** registro de todas as ações feitas pelo Cockpit.
7. **Pesquisa de histórico:** encontrar chamados, problemas e mudanças antigos ou já encerrados das frentes do usuário, fora da fila atual (§7.7).
8. **Relatórios das filas:** exportar a lista filtrada da tela, escolhendo os campos (§11A).

**Fora de propósito:** abrir chamados novos. Para criar um filho para outra equipe, os terceiros usam o **Transferir** (§7.3). Para abrir uma requisição do catálogo, eles usam o portal do solicitante, que não exige licença.

### 2.2 Fora da V1 (V2 ou nunca)

| Item | Destino |
|---|---|
| Notificações por e-mail ou Microsoft Teams | V2 |
| Assistente de IA para gerar artigos por conversa | V2. A V1 já deixa os "tipos de documento" configuráveis, que o assistente vai usar. |
| Webhooks do Freshservice para atualização instantânea | V2, se o cache se mostrar insuficiente |
| Abas de chamado **Tarefas, Ativos e Associações** | Fora. São pouco usadas, e os casos raros ficam com a equipe interna. |
| Aba **Aprovações** do chamado | Fora. Só chegam à fila chamados já aprovados. |
| Aprovar ou rejeitar mudanças (papel de aprovador/CAB) | Fora. Continua com a equipe interna no Freshservice. |
| Chats do Teams e e-mails pós-CAB | Fora. Continuam manuais. |
| Dashboards e relatórios além da exportação de filas (§11A) | Fora da V1. Ver §13 (conferência antes de cortar cada licença). |
| Abrir chamados novos | Fora (ver acima). |

---

## 3. Restrições

- **Tecnologia:** Node.js com TypeScript. Back-end em **Fastify**, front-end em **React + Vite** servido pelo próprio Fastify, banco **SQL Server**. O time interno constrói e mantém.
- **Infraestrutura:** Docker em uma VM Linux interna, publicada no **proxy reverso interno atrás do firewall**. A VM precisa de saída liberada para o Freshservice e o Microsoft Entra ID.
- **SQL Server:** usar preferencialmente a instância corporativa já mantida pelo time de banco, com backup e monitoramento. Nesse caso, o `docker compose` sobe só a aplicação. *(Decisão do assistente; confirmar com o time de banco.)*
- **Login:** SSO pelo **Microsoft Entra ID**, com um registro de app próprio do Cockpit, separado do app do Freshservice.
- **Limite de API do Freshservice:** a produção respondeu `x-ratelimit-total: 500` por minuto, o patamar Enterprise. Pela tabela da documentação, os limites por operação nesse patamar são: listar chamados 140/min; ver, criar e atualizar chamado 160/min cada; listar agentes e solicitantes 140/min. O desenho precisa ficar muito abaixo disso.
- **Volume:** cerca de 800 chamados por mês dos terceiros, de um total de 2.500 a 3.000. Uns 40 por dia útil.
- **Usuários:** cerca de **40 terceiros**, em muitas frentes. A ideia é migrar todos. A virada pode ser total ou por frente, e não há uma frente-piloto definida.
- **Ambientes do Freshservice:** QAS (`ssabrhelpdesk-qa.freshservice.com`) e PRD (`ssabrhelpdesk.freshservice.com`). Os IDs de categorias, pastas, grupos e campos **são diferentes entre os dois**. Nada pode ficar fixo no código; tudo vem de configuração ou é descoberto pela API.
- **Segredos:** a chave da API do Freshservice fica só no servidor e nunca chega ao navegador. O `.env` está no `.gitignore`.

---

## 4. Arquitetura

```
 Navegador (terceiro)
      │  HTTPS (proxy reverso)
      ▼
 ┌──────────────────────────── VM Linux / Docker ───────────────────────────┐
 │  Fastify (API + arquivos estáticos do React)                             │
 │   ├─ Autenticação OIDC com Entra ID                                       │
 │   ├─ Autorização: usuário → frentes → grupos/pastas                       │
 │   ├─ Cliente Freshservice (chave do usuário de integração)                │
 │   │    ├─ cache em memória por fila/registro (TTL curto)                  │
 │   │    └─ controle de vazão (respeita x-ratelimit-*)                      │
 │   └─ Registro de ações (auditoria)                                        │
 └───────────────┬──────────────────────────────────────┬────────────────────┘
                 │                                      │
                 ▼                                      ▼
          SQL Server (Cockpit)                  Freshservice API v2
   usuários, frentes, configurações,         chamados, problemas, mudanças,
   auditoria, marcações de leitura           artigos (fonte da verdade)
```

### 4.1 Cache e consumo de API

- **Sem banco de chamados e sem cópia local.** As leituras passam por um **cache compartilhado em memória**, por fila e não por usuário. Se dez terceiros olham a mesma fila, isso vira uma chamada só.
- **Sugestão de TTL** *(o construtor pode ajustar)*: listas de fila de 30 a 60 s; detalhe de registro de 30 s; definições de formulário (campos, opções, status) de algumas horas.
- **Toda gravação feita pelo Cockpit invalida o cache** do registro e da fila afetados, para que o próprio terceiro veja o resultado na hora.
- **Webhooks não são usados na V1.** Com esse volume, o cache curto basta. Se forem adotados depois, o Freshservice chama um endereço do Cockpit pelo mesmo proxy.
- **Formulários dinâmicos:** os campos (tipos, opções, dependências, obrigatoriedade) são lidos das APIs de definição de formulário (`/api/v2/ticket_form_fields`, `/api/v2/problem_form_fields` e a equivalente de mudanças) e montados na tela. Assim, um campo novo no Freshservice aparece no Cockpit sem desenvolvimento. O Cockpit **escolhe quais campos mostrar** em cada situação (ver §7.3).
- O cliente do Freshservice deve respeitar os cabeçalhos `x-ratelimit-remaining` e o `Retry-After` em caso de 429, com fila e retentativa.

### 4.2 Banco do Cockpit (SQL Server)

Entidades mínimas *(o desenho exato fica com o construtor)*:

- **Usuário:** identificador do Entra, nome, e-mail, ativo, papel (terceiro ou administrador), último login.
- **Frente:** nome; grupos do Freshservice de chamados, de problemas e de mudanças; pastas da base de conhecimento liberadas. Cada grupo e pasta é guardado com o **workspace** do Freshservice a que pertence, já que formulários, status e pastas variam por workspace. Hoje o processo usa o workspace de TI (ID 7 em PRD), mas o modelo não deve assumir um único workspace. *(Decisão do assistente.)*
- **Usuário ↔ Frente:** vínculo entre pessoas e frentes.
- **Configuração de campos por status:** para cada status (e, se necessário, por grupo), quais campos aparecem e quais são obrigatórios.
- **Tipo de documento (base de conhecimento):** nome, pasta de destino, prefixo de numeração, template HTML, campos obrigatórios.
- **Registro de ações (auditoria):** quem, quando, o quê, em qual registro do Freshservice, resultado (sucesso ou erro) e dados enviados (resumidos).
- **Marcação de leitura:** por usuário e registro, quando ele abriu pela última vez. Serve para o destaque de "novidade" (§7.1).

---

## 5. Identidade, acesso e administração

- **Login:** OIDC com o Entra ID, usando o app próprio do Cockpit.
- **Quem entra:** só membros de **um grupo de segurança do Entra** (por exemplo, "Cockpit – Terceiros"). Desativado no Entra significa sem acesso ao Cockpit.
- **O que cada um vê:** definido **dentro do Cockpit**, numa tela de administração. Para cada terceiro, as frentes que ele atende. Cada frente tem seus grupos do Freshservice. É isso que substitui os grupos de agentes do Freshservice para os terceiros.
- **Primeiro acesso:** um membro novo do grupo do Entra que faz login aparece na administração como **"sem frentes"** e não vê nada até um administrador liberar. *(Decisão do assistente.)*
- **Administradores:** equipe interna. O primeiro administrador é definido por configuração (variável de ambiente) e os demais na própria tela de administração. *(Decisão do assistente.)*
- **Regra de visibilidade:** o terceiro vê e atua **somente** em registros cujo grupo pertence a uma de suas frentes. A checagem é feita **no servidor**, a cada leitura e gravação, nunca só na tela.
- **Exceção controlada, o chamado pai:** o pai normalmente está em outro grupo. O terceiro tem acesso ao pai **somente através de um filho visível**, e só para ler os anexos do pai e gravar nele respostas e notas com anexo (§7.4). Ele não navega pelo pai nem altera seus campos. *(Decisão do assistente.)*

---

## 6. Regras gerais de integração com o Freshservice

### 6.1 Usuário de integração e autoria

- Todas as chamadas usam a chave de **um usuário de integração**. A chave fornecida pertence a "INT_USER – Fresh Admin QA" no QAS e a "INT_USER – Fresh Admin PRD" em PRD. Esse usuário é quem aparece como autor das gravações.
- Ele é **diferente do agente "Integracao Freshservice"**, que é usado como agente responsável quando o Freshservice exige um (§6.3).
- No Freshservice, tudo vai aparecer como feito por esse usuário. Por isso:
  - toda nota ou resposta escrita pelo Cockpit leva o **carimbo do autor** (por exemplo, uma linha "— Nome do Terceiro (Frente X) via Cockpit"). O formato exato fica com o construtor;
  - o registro de ações do Cockpit guarda o autor real de cada operação.

### 6.2 Campo "Consultores" (existente)

- O Cockpit usa o campo **`consultores`**, uma lista suspensa que **já existe** nos formulários de **chamado** (32 opções em PRD), **problema** (40) e **mudança** (41). *(Decisão da empresa. Substitui a ideia anterior de criar um campo de texto novo.)*
- As **listas de opções são diferentes** em cada formulário. O mesmo consultor pode existir em uma e faltar em outra.
- A API **não permite incluir opções** em listas suspensas (verificado). Por isso, o cadastro de novos consultores continua **manual, no Freshservice**, feito pela equipe.
- **Vínculo usuário ↔ opção:** na administração do Cockpit, cada terceiro é vinculado à opção correspondente **em cada um dos três formulários**. O Cockpit sugere o vínculo pelo nome, e o administrador confirma.
- **Aviso:** a tela de administração lista os terceiros **sem opção correspondente** em algum formulário, para a equipe incluir o nome no Freshservice.
- **Regra de gravação:** quando um terceiro **assume** um chamado ou problema, ou abre ou assume uma mudança, o Cockpit grava a opção vinculada a ele. Se outra pessoa assumir, o campo é atualizado.
- **Terceiro sem opção vinculada:** se o campo for obrigatório no registro (a tela do Freshservice mostra "Consultores *" em alguns casos), a gravação é **bloqueada**, com a mensagem "seu nome ainda não está cadastrado no campo Consultores; fale com o administrador". Se o campo não for obrigatório, a gravação segue sem ele, e o administrador é avisado. *(Decisão do assistente.)*
- O construtor deve verificar **quando** o campo é obrigatório. Pela API ele não aparece como obrigatório, então a regra deve vir de uma regra de negócio do formulário (ver V12).

### 6.3 Campo "Agente" do Freshservice

Os terceiros não são agentes. O responsável real fica sempre no campo "Consultores" (§6.2).

**Verificado no QAS:**

- em **mudanças**, o agente é **obrigatório também pela API** (sem ele, a criação é recusada);
- o agente **precisa ser membro do grupo** da mudança ("Assigned agent isn't a member of the group");
- o dono da chave (INT_USER) não é aceito como agente.

**Regras:**

- **Chamados e problemas:** ficam **sem agente**. O responsável é só o campo "Consultores". Ver V8, sobre automações que possam exigir agente.
- **Mudanças:** o agente é sempre o **agente "Integracao Freshservice"** (PRD `19000130143`, QAS `41000806184`). Isso vem de configuração, não fica fixo no código.
  - **Pré-requisito de implantação:** a equipe inclui esse agente como **membro de todos os grupos** usados pelos terceiros em mudanças. Hoje ele está em 4 de 78 grupos de TI em PRD e em 5 no QAS.
  - A tela de administração mostra um **aviso** quando um grupo configurado numa frente não tem esse agente como membro.
  - Também é possível definir um agente diferente por frente, se um dia for necessário.
- O campo **"Consultores"** (§6.2) também é preenchido nas **mudanças**.

---

## 7. Chamados

### 7.1 Fila

- Mostra os chamados dos grupos das frentes do usuário. Na prática, são os **chamados filhos** criados pela automação para cada fila, mais os **chamados sem pai** que caem direto na fila (por exemplo, os do catálogo técnico).
- **Colunas fixas:** número, assunto, **grupo (fila)**, solicitante, status, prioridade, **SLA** (vencimento), consultor e última atualização.
- **SLA visível:** cor para "perto de vencer" e "vencido", e ordenação por prazo.
- **Destaque de novidade:** chamado novo ou alterado desde a última vez que **aquele usuário** o abriu (comparando a data de atualização com a marcação de leitura). Não gasta API extra.
- Filtros por **grupo (fila)**, status, prioridade, consultor e "meus chamados". O filtro de grupo permite escolher uma, várias ou todas as filas do usuário, e é essencial para quem atende duas ou mais filas. A seleção fica lembrada por usuário. *(Lembrar a seleção é decisão do assistente.)* O mesmo vale para as filas de problemas e mudanças.
- Os terceiros ficam sabendo das novidades **por aqui**, já que não recebem os e-mails de agente.

### 7.2 Detalhe do chamado

- **Cabeçalho e campos:** os campos escolhidos para o status atual (§7.3).
- **Conversa:** mostra **só a conversa do chamado filho**. A automação existente já traz para o filho as respostas do solicitante. Juntar com a do pai duplicaria.
- **Anexos:** mostra **só os anexos do chamado pai**. Os anexos copiados de pai para filho chegam corrompidos com frequência. Se não houver pai, mostra os do próprio chamado.
- **Ticket pai e relacionados:** visualização com link e status.
- **Atividade:** histórico de alterações (status, transferências, atualizações), se a API permitir. Ver §14.

### 7.3 Status e regras

O formulário muda conforme o status escolhido. Cada status mostra seus campos e só grava quando as regras são cumpridas. O mapeamento "status → campos" é **configurável** na administração. Os valores iniciais estão abaixo e vêm do documento N1-KBA150.

| Status | Disponível ao terceiro | Regras no Cockpit |
|---|---|---|
| Aberto | **Não** (estado inicial) | — |
| Em Atendimento | Sim | Ao assumir: grava o campo "Consultores" (§6.2). Chamados ficam sem agente (§6.3). |
| Pendente Fornecedor | Sim | Obrigatórios: **Nº do Chamado no Parceiro**, **Justificativa da Pendência**, **Motivo**. Obrigatório também enviar, na mesma ação, uma **resposta ao solicitante** informando a pendência. |
| Pendente Área Interna | Sim | Obrigatórios: Justificativa e Motivo, mais a resposta ao solicitante. |
| Pendente Solicitante | Sim | Obrigatórios: Justificativa e Motivo, mais a resposta ao solicitante. |
| Resolvido | Sim | Obrigatório: **anotação de resolução** (§7.5). |
| Transferir | Sim | Obrigatórios: **Grupo destino** e **anotação de resolução** (análise feita antes da transferência). |
| Cancelado | Sim | Obrigatório: **motivo do cancelamento**, gravado como **nota pública no filho**. Avisar que o cancelamento não permite reabertura. |
| Reaberto | Sim, só a partir de **Resolvido** | Permitido enquanto o chamado estiver Resolvido, ou seja, antes de virar Encerrado. Fluxo guiado: Reaberto, depois atribuição, depois Em Atendimento. *(O documento fala em 72 h, e a automação encerra após 3 dias úteis. O Cockpit usa "enquanto estiver Resolvido" para unir as duas regras; decisão do assistente.)* |
| **Encerrado** | **Nunca** | Status aplicado só pelo Freshservice, quando o usuário encerra ou 3 dias úteis após a resolução (período de garantia). |

Os nomes, IDs e valores reais dos status e campos são lidos da API do ambiente. Não ficam fixos no código.

### 7.4 Onde cada informação é gravada (filho × pai)

| Informação | Onde grava |
|---|---|
| Status, campos, pendência, consultor, agente | **Filho** |
| Notas e respostas **sem anexo** | **Filho**. A automação existente leva ao pai o que precisa ir. |
| Anotação de resolução | **Sempre no filho** (na transferência, ela passa de um filho para o outro). |
| Motivo de cancelamento | **Filho**, como nota pública. |
| **Resposta com anexo** | **Pai**: texto + anexo como **resposta pública**. **Filho**: o texto como **nota privada**, com a referência "anexo enviado no chamado pai #SR-xxxx". |
| **Nota interna com anexo** | **Pai**: texto + anexo como **nota privada**. **Filho**: o texto como nota privada, com a mesma referência. |
| Chamado **sem pai** | Tudo nele mesmo, anexos inclusive, sem divisão. |

- **Como achar o pai:** o histórico de um filho real mostra que a automação faz duas coisas: grava o campo personalizado **`ticket_pai`** (por exemplo, 342365) e cria a **associação nativa pai/filho** do Freshservice. O Cockpit usa a associação nativa e, se ela não vier, recorre ao campo `ticket_pai`. *(Decisão do assistente; o nome exato dos campos deve ser confirmado na API.)*
- O arquivo anexo é enviado **só ao pai**, para não subir duas vezes.
- **Cuidado na virada:** se a automação atual copia interações do filho para o pai, ela precisa ser conferida para não duplicar o que o Cockpit grava direto no pai. Ver §13.

### 7.5 Anotação de resolução

- Obrigatória em **Resolvido** e **Transferir**.
- Gravada no **campo de resolução do filho**. Além disso, o Cockpit cria **no filho uma nota pública** com o mesmo texto.
- **Verificado no QAS:** `PUT /tickets/{id}` com `{"resolution_notes": "..."}` grava, e `GET` devolve `resolution_notes` e `resolution_notes_html`. O campo `resolution_notes_html` **não é aceito** na gravação. O construtor deve testar se `resolution_notes` aceita HTML.
- Se a resolução for editada de novo, cada gravação gera uma nova nota pública com o texto atualizado. *(Decisão do assistente.)*
- Ver §14: confirmar se o campo de resolução é acessível pela API.

### 7.6 Respostas, notas e anexos

- Editor de texto com formatação básica.
- Anexos por **seleção de arquivo, arrastar ou colar (Ctrl+V)**. Em respostas e notas, **um print colado vira anexo**, não imagem dentro do texto. *(Decisão do assistente; anexos em respostas são documentados na API.)*
- **Limite de anexos: 40 MB no TOTAL por registro**, somando todos os anexos já existentes no registro com os novos. Não é 40 MB por arquivo: se um único anexo tiver 40 MB, o registro não aceita mais nada.
  - Como os anexos de chamados vão para o **pai** (§7.4), a conta é feita sobre o **total de anexos do pai**.
  - **Antes de enviar**, o Cockpit soma o tamanho dos anexos já existentes (a API informa o `size` de cada um) com os novos. Se passar de 40 MB, ele bloqueia o envio e mostra quanto espaço resta.
  - O mesmo vale para notas de mudança, problemas e artigos.
  - Se o Freshservice recusar mesmo assim, o texto digitado não é perdido.
  - O valor de 40 MB fica em configuração.

### 7.7 Pesquisa de histórico

- Encontra chamados das frentes do usuário **em qualquer status e período**, inclusive Encerrado e Cancelado. O mesmo vale para problemas e mudanças. *(Estender a problemas e mudanças é decisão do assistente.)*
- Critérios: **número**, período (criação ou atualização), status, grupo, solicitante, consultor e categoria. Os resultados abrem o detalhe em modo leitura quando o registro já está encerrado.
- Implementação pela API de filtro do Freshservice (`/api/v2/tickets/filter`, com consulta por campos). Essa API **não faz busca por texto livre** no assunto ou na descrição. Ver V10 em §14.
- Pesquisas não usam o cache de fila. Elas consomem API sob demanda, então devem ter **período máximo** e **paginação**. Os valores ficam com o construtor.

---

## 8. Problemas

No processo da SSA, o problema é **o registro de documentação e análise de causa raiz**. Todo o processo acontece nele, inclusive a visão dos chamados associados.

- **Fila:** problemas dos grupos das frentes do usuário, com o mesmo padrão da fila de chamados (colunas, novidade, filtros).
- **Abertura:** o próprio terceiro abre o problema, com o formulário vindo de `/api/v2/problem_form_fields`. Em PRD, no espaço de TI, os campos incluem solicitante, departamento, prioridade, impacto, grupo, categoria, título, descrição, status, agente, **Justificativa da pendência** e tags.
- **Status de PRD:** Open, Em análise, Documentação, Melhoria Requisitada, Change Requested, Pendente Solicitante, Pendente Fornecedor e Closed. Os valores são lidos da API.
- **Análise de causa raiz:** os campos **causa, impacto e sintomas** (`analysis_fields`, confirmados na API) são a área central da tela de problema.
- **Erro conhecido:** marcação `known_error` e solução de contorno.
- **Notas e tarefas:** as APIs `/problems/{id}/notes` e `/problems/{id}/tasks` respondem. Notas aceitam anexos, e tarefas podem ser criadas, atualizadas e concluídas.
- **Vínculos:**
  - **mudanças associadas:** já vêm no próprio problema;
  - **chamados associados (requisito obrigatório: ver e associar).** Situação verificada no QAS:
    - **associar funciona** pelo lado do chamado: `PUT /tickets/{id}` com `{"problem":{"display_id":N}}`;
    - o chamado informa o problema dele (`GET /tickets/{id}?include=problem`);
    - **não existe** consulta "chamados deste problema". `/problems/{id}/tickets`, `include=tickets` e filtros por `problem_id` são recusados.
  - **Solução: índice de vínculos no Cockpit.** Uma tabela pequena guarda os pares chamado ↔ problema e é mantida por um processo em segundo plano:
    1. a cada poucos minutos, lista os chamados alterados desde a última execução (`GET /tickets?updated_since=...`, que funciona);
    2. para cada um, lê `include=problem` e atualiza o índice;
    3. associações feitas pelo próprio Cockpit entram no índice na hora;
    4. uma **carga inicial** do histórico roda uma vez, fora do horário e com vazão limitada. O período mínimo fica com o construtor, sugerido em 12 meses.
  - Em PRD, houve cerca de 690 chamados alterados em pouco mais de um dia, no espaço de TI. São algumas centenas de leituras por dia, dentro do limite com folga.
  - **Pendente de verificação (V2):** confirmar que associar um chamado a um problema **pela tela do Freshservice** também altera a data de atualização do chamado. Se não alterar, o índice só percebe essas associações na carga periódica completa.
- **Consultor:** campo "Consultores" (§6.2).
- **Regras:**
  - Pendente Solicitante ou Fornecedor: **Justificativa da pendência obrigatória**.
  - **Closed:** exige **causa raiz preenchida**. *(Decisão do assistente.)*
  - Change Requested: oferecer o atalho "abrir mudança a partir deste problema", já vinculada.

---

## 9. Mudanças

Referência: **CH-KBA005, Procedimento de Gestão de Mudanças no Freshservice (v1.3)**. O terceiro atua como **criador/solicitante** e como **agente/implementador**. A aprovação continua com o núcleo de aprovadores e o CAB, dentro do Freshservice.

### 9.1 Abertura e formulário

- Abrir do zero, ou a partir de um chamado ou problema. Nesse caso, a mudança já fica **associada** ao registro de origem.
- O formulário vem da definição de campos de mudança da API. Pelo CH-KBA005, ele inclui: espaço de trabalho, solicitante, **Tipo** (Padrão, Normal, Emergencial), grupo, agente, assunto, descrição, status, prioridade, impacto, risco, **Ambiente** (Produção ou DEV/QA), **Altera tela e/ou funcionalidade**, **Haverá indisponibilidade**, **Impacto em processo core**, **Número da Request (ToC)**, datas planejadas de início e término, **Data de Início Rollback** e demais campos de rollback, departamento, categoria, **Janela de Manutenção** e **Classificação** (Corretiva, Preventiva, Melhoria, Ajuste, Projetos).
- **Planejamento**, com quatro textos: **Justificativa**, **Análise de risco e impacto**, **Plano de implementação** e **Plano de rollback**.
- **Tipo Padrão:** lembrar que o Plano de Implementação precisa ter o **link da KBA** da mudança padrão. Oferecer a busca na base de conhecimento para inserir o link. *(Sugestão do assistente.)*

### 9.2 Status disponíveis ao terceiro

| Status | Uso pelo terceiro |
|---|---|
| Aberto → Planejamento → Pré-Análise | Transições feitas pelo **criador**. "Pré-Análise" sinaliza que a mudança está pronta para o CAB. |
| Aguardando Aprovação | Acompanhar. |
| Em execução | Executar e registrar evidências. |
| Encerrado / Encerrada com Falha | Encerrar conforme o resultado real, depois da execução. |
| Cancelado | Descartar um registro incorreto **após rejeição**. |
| Liberação Pendente, Revisão Pendente | **Ocultos.** Não são usados no processo atual. |

- O Cockpit **não permite encerrar** (Encerrado ou Encerrada com Falha) uma mudança que não tenha todas as aprovações concluídas. *(Decisão do assistente, coerente com o procedimento.)*
- As regras exatas de quem faz cada transição depois da Pré-Análise ainda estão em aberto no próprio CH-KBA005 ("devem ser formalizados pela governança interna"). A tabela fica **configurável**.

### 9.3 Anotações, tarefas e associações

- **Anotações:** é o lugar obrigatório para evidências e anexos (DTI em PDF, Runbook em Excel, prints de teste, validação em produção, comunicações, formalizações). O Cockpit precisa **criar anotações com anexos** e listar as existentes.
  - **Verificado no QAS:** `POST /changes/{id}/notes` aceita JSON e `multipart/form-data` com `attachments[]`. O anexo é gravado e abre pelo endereço `/helpdesk/attachments/<id>`.
  - **Atenção:** a **listagem** de notas (`GET /changes/{id}/notes`) **não traz os anexos**. Eles só aparecem ao ler cada nota (`GET /changes/{id}/notes/{note_id}`). O cache precisa considerar esse custo.
  - As notas criadas pela API ficam como **privadas** por padrão.
- **Tarefas, verificado no QAS:** criar (`POST /changes/{id}/tasks`) e concluir (`PUT .../tasks/{id}` com `status: 3`) funcionam.
- **Tarefas:** criar, atribuir responsável, definir prazo, atualizar andamento e concluir. Por exemplo, o snapshot de VMs em Operações. Os responsáveis podem ser terceiros, que não são agentes. Nesse caso, o nome fica no título ou na descrição da tarefa, e a tarefa fica **sem agente** (o teste no QAS criou tarefa sem agente). *(Decisão do assistente.)*
- **Associações:** associar o chamado de origem e os problemas relacionados. Associação de ativos também, se a API permitir.

### 9.4 Aprovações

- **As solicitações de aprovação são enviadas pelo Gestor de Mudanças, no Freshservice.** O Cockpit não envia nem decide aprovações.
- O Cockpit mostra o **estado das aprovações** (quem aprovou, quem falta), apenas para leitura.

### 9.5 Avisos de prazo *(sugestão do assistente)*

- Prazo de cadastro para o CAB: até **12h** do dia da reunião (terças e quintas, das 15h às 16h30).
- Prazo de encerramento: **10 dias** após a "Data de Término Planejada".
- Lista "minhas mudanças pendentes de encerramento", porque pendências bloqueiam novas defesas.

---

## 10. Base de conhecimento

Referência: **SD-KBA009, Processo de criação de artigos na base de conhecimento**.

### 10.1 Consulta

- Busca em artigos **publicados** e nos **rascunhos das frentes do usuário**.
- Visualização do artigo com imagens.

### 10.2 Tipos de documento (configuração)

Cada **tipo de documento** define a pasta de destino, o **prefixo de numeração** (por exemplo, `N1-KBA`) e o **template**. A frente do usuário define quais pastas e tipos ele pode usar.

### 10.3 Criação

- **Título numerado automaticamente:** o Cockpit encontra o maior número existente na pasta (publicados e rascunhos) e sugere o próximo (por exemplo, N1-KBA034 → N1-KBA035). O número é **conferido de novo na publicação**, e o Cockpit avisa se houver conflito. *(Decisão do assistente.)*
- **Corpo pré-preenchido com o template:** OBJETIVO, APLICAÇÃO, DEFINIÇÕES, PROCESSOS, ANEXOS ("Não há") e DOCUMENTOS RELACIONADOS ("Não há").
- **Preenchidos automaticamente:** categoria e pasta (pelo tipo), tipo do artigo sempre **Permanente**, **data de revisão = criação + 6 meses**.
- **Obrigatórios:** título, corpo, **tags** e **palavras-chave**.
- **Autor:** o Freshservice grava o usuário de integração como autor. O nome real de quem escreveu precisa ficar registrado. Se a API tiver um campo de autor editável, usar esse campo. Senão, incluir uma linha no rodapé do artigo, por exemplo "Autor: Nome · Validado por: Nome". *(Decisão do assistente; ver §14.)*
- **Salvar cria o artigo como rascunho** (`status = 1`).

### 10.4 Imagens coladas (verificado no QAS)

- O editor aceita **colar imagens (Ctrl+V)** e arrastar arquivos, como no Freshservice.
- **Caminho escolhido (testado e confirmado visualmente no QAS):**
  1. ao salvar, o Cockpit envia as imagens como **anexos do artigo** (`multipart/form-data`, campo `attachments[]`). Isso funciona, embora não esteja documentado;
  2. o Cockpit reescreve o corpo, apontando cada `<img>` para `https://<domínio>/helpdesk/attachments/<id-do-anexo>`. Esse endereço abre **sem login**, porque redireciona para um link assinado gerado a cada acesso;
  3. a imagem aparece no corpo e também na lista de anexos do artigo. Esse efeito colateral é aceitável.
- **Plano B, também comprovado:** imagem em base64 dentro do HTML. Funciona, mas deixa o artigo pesado.
- Durante a edição, antes de salvar, a imagem é mostrada a partir do navegador.
- **Cuidado de implementação:** o envio multipart deve tratar os campos de texto como texto literal. No teste, o `curl -F` interpretava `<` como "ler arquivo". Bibliotecas HTTP do Node não têm esse problema, mas o teste de integração deve cobrir HTML no corpo.

### 10.5 Validação e publicação

- Pelo passo 2.6 do SD-KBA009, o artigo precisa ser **validado por um par ou superior, e quem valida é quem publica**.
- No Cockpit, **qualquer usuário com acesso à mesma frente** pode editar e publicar um rascunho, **menos quem o criou**.
- O Cockpit registra quem criou, quem editou e quem publicou.

---

## 11. Rastreabilidade

- Toda ação de gravação passa pelo registro de ações: autor real, data e hora, tipo de ação, registro afetado no Freshservice, resultado e mensagem de erro, se houver.
- Tela de consulta do registro para administradores, com filtros por pessoa, frente, registro e período. *(Decisão do assistente.)*
- O carimbo do autor nas notas e respostas (§6.1) mantém a rastreabilidade também dentro do Freshservice.

---

## 11A. Relatórios das filas (exportação)

O relatório de hoje é **o filtro que está na tela, exportado**. Por exemplo, um gestor de consultoria exporta todas as filas a que tem acesso, com os analistas dessas filas. Todos exportam do mesmo jeito, com a opção de marcar **todos os campos do chamado**.

- Botão **Exportar** na fila e na pesquisa de histórico (§7.7). Ele exporta **exatamente o conjunto filtrado na tela**, em todas as filas selecionadas, não só a página visível.
- **Seletor de campos:** o usuário marca as colunas desejadas, entre todos os campos do formulário, padrão e personalizados, com a opção "marcar todos". Campos de lista saem com o rótulo, não com o ID.
- Filtros mínimos: filas (uma, várias ou todas as do usuário), status, período, consultor (analista), prioridade e SLA vencido.
- **Formatos:** Excel (.xlsx) e CSV. *(A escolha dos formatos é decisão do assistente.)*
- **Consumo de API:** a exportação pagina a API e respeita o controle de vazão. Exportações grandes rodam em segundo plano, com aviso quando o arquivo fica pronto. Também precisa haver um período máximo por exportação. Os valores ficam com o construtor.
- O mesmo vale para as filas de problemas e mudanças. *(Decisão do assistente.)*

## 12. Interface — diretrizes

- Ferramenta de trabalho, densa e rápida, com navegação por **Chamados, Problemas, Mudanças, Base de conhecimento** e (para administradores) **Administração**.
- Funciona bem em telas de notebook. Celular não é prioridade. *(Decisão do assistente.)*
- O formulário mostra os erros de obrigatoriedade **antes** de enviar ao Freshservice.
- Qualquer erro do Freshservice é mostrado em linguagem clara, e nada é perdido: o texto digitado permanece na tela.

O desenho visual fica com o construtor.

---

## 13. Virada e corte de licenças

1. **Piloto com uma frente.** Ela passa a usar o Cockpit, em paralelo com a conta compartilhada, por um período combinado.
2. **Conferir a automação pai ↔ filho**, para não duplicar o que o Cockpit grava direto no pai (§7.4).
3. **Antes de cancelar cada licença**, confirmar com a frente que **nada** mais exige a conta compartilhada: relatórios, dashboards e tarefas esporádicas.
4. Cancelar a licença da frente e repetir com as demais. Também é possível virar todas de uma vez.
5. **Contrato com a Freshworks:** a empresa considera que o uso de uma chave de API para várias ações não é problema. Nenhuma checagem prévia é necessária. *(Decisão da empresa.)*

---

## 14. Verificações obrigatórias antes de construir

Cada item deve ser testado no **QAS**. Leituras podem ser feitas em PRD. Gravações de teste, **só no QAS**.

| # | Verificar | Se falhar |
|---|---|---|
| V1 | ~~Solicitar aprovação de mudança pela API~~ | **Descartada.** As solicitações são enviadas pelo Gestor de Mudanças no Freshservice. |
| V2 | **Associação chamado ↔ problema feita pela tela do Freshservice altera o `updated_at` do chamado?** | O índice de vínculos (§8) passa a depender de uma varredura periódica completa, mais cara. |
| V3 | ✅ **Verificada.** `resolution_notes` grava e lê. Falta só confirmar se aceita HTML. | — |
| V4 | ✅ **Verificada.** Notas de mudança com anexo, criação e conclusão de tarefas funcionam. | — |
| V5 | **Campo "Grupo destino"** no Transferir: nome real e se a automação de transferência dispara ao gravar pela API | Ajustar a automação. |
| V6 | **Autor do artigo**: existe campo editável na API? | Rodapé com autor e validador (§10.3). |
| V7 | **Histórico/atividade** do chamado pela API | Mostrar só a conversa e os dados do chamado. |
| V8 | **Chamados e problemas aceitam um agente que não seja membro do grupo?** Quais automações dependem do agente? (§6.3) | Usar o agente responsável padrão da frente também em chamados e problemas. |
| V9 | **Associação de ativos** em mudanças pela API | Fica com a equipe interna. |
| V10 | **Pesquisa de histórico:** campos aceitos pela API de filtro (incluindo campos personalizados como "Consultores"), limite de resultados por consulta, e o equivalente para problemas e mudanças | Restringir os critérios e o período, e paginar por janelas de tempo. |
| V12 | **Quando o campo "Consultores" é obrigatório:** regras de negócio do formulário por grupo, categoria ou status | Tratar como obrigatório sempre que o Freshservice recusar a gravação sem ele, com mensagem clara. |
| V11 | **Exportação:** se a listagem já traz todos os campos personalizados ou se exige uma leitura por chamado | Se exigir uma leitura por chamado, a exportação fica cara. Avaliar o limite de volume. |

Já verificado:

- Anexo em artigo com imagem no corpo (§10.4).
- Base64 no corpo do artigo (plano B).
- Ausência de API para incluir opções em listas suspensas (por isso o vínculo manual usuário ↔ opção do campo "Consultores", §6.2).
- `analysis_fields` dos problemas e as APIs de notas e tarefas de problema respondem.
- Limite de API em PRD: 500 por minuto.

---

## 15. Decisões tomadas pelo assistente (para revisão)

Estas decisões foram tomadas para fechar o desenho e **não foram confirmadas explicitamente**:

1. SQL Server corporativo em vez de container (§3).
2. Novos usuários entram "sem frentes" até um administrador liberar; primeiro administrador por variável de ambiente (§5).
3. ~~Agente do Freshservice = usuário de integração~~. **Substituída por decisão da empresa:** chamados e problemas sem agente; mudanças com o agente "Integracao Freshservice" (§6.3).
4. "Aberto" não é oferecido ao terceiro; reabertura permitida enquanto estiver "Resolvido" (§7.3).
5. Arquivo anexo enviado só ao pai; no filho fica só a referência (§7.4).
6. Cada edição da resolução gera nova nota pública (§7.5).
7. Print colado em resposta ou nota vira anexo, não imagem no texto (§7.6).
8. Problema só fecha (Closed) com causa raiz preenchida (§8).
9. Mudança só encerra com todas as aprovações concluídas (§9.2).
10. Tarefas de mudança atribuídas a terceiros: nome no texto, sem agente (§9.3).
11. Avisos de prazo de CAB e encerramento (§9.5).
12. Numeração conferida de novo na publicação (§10.3).
13. Autor do artigo no rodapé, caso a API não tenha o campo (§10.3).
14. Tela de consulta do registro de ações (§11).
15. Prioridade para notebook, não para celular (§12).
16. Acesso ao chamado pai só através de um filho visível, limitado a anexos e respostas/notas com anexo (§5).
17. Pai localizado pela associação nativa, com o campo `ticket_pai` como alternativa (§7.4).
18. Frentes guardam o workspace de cada grupo e pasta (§4.2).
19. Pesquisa de histórico e exportação também para problemas e mudanças (§7.7, §11A).
20. Exportação em Excel e CSV, com as grandes em segundo plano (§11A).
21. Índice local de vínculos chamado ↔ problema, mantido por sincronização incremental, com carga inicial de 12 meses (§8).

---

## 16. Deixado ao construtor

- Estrutura de pastas, bibliotecas (ORM, editor de texto, componentes de UI), modelo exato das tabelas.
- Valores finais de TTL do cache e estratégia de controle de vazão.
- Formato exato do carimbo do autor.
- Desenho visual.
- Estratégia de testes e de ambientes do próprio Cockpit (desenvolvimento, homologação, produção), considerando que QAS e PRD do Freshservice têm IDs diferentes.

---

## 17. Referências

- N1-KBA150, Processo de Tratamento de Chamados no Freshservice Conforme Práticas ITSM.
- SD-KBA009, Processo de criação de artigos na base de conhecimento.
- CH-KBA005, Procedimento de Gestão de Mudanças no Freshservice, v1.3 (arquivo na pasta do projeto).
- Freshservice API v2: https://api.freshservice.com/
