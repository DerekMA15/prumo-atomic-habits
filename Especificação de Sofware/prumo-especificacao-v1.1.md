# Prumo — Especificação de Software v1.1

> Bullet journal digital com métricas: planejamento, hábitos, sono, check-in diário e diário pessoal em um único app Android, 100% local.

- **Versão:** 1.1 (MVP, revisada)
- **Data:** 16/11/2026
- **Plataforma:** Android
- **Uso:** pessoal, usuário único, sem login. O bloqueio do app existe por **privacidade dos dados**, não por multiusuário.

### Histórico de versões

| Versão | Mudanças |
|---|---|
| 1.0 | Primeira versão: requisitos, modelo, casos de uso, telas, stack e plano |
| 1.1 | Fecha as decisões em aberto da revisão: chave do banco, sono (cama vs. dormindo), unicidade, períodos de meta/revisão, backup criptografado, sono em "Hoje", navegação, DI, testes e plano reordenado |

---

## 1. Visão

O Prumo une a lógica do **Bullet Journal** (Future Log, Monthly Log, Daily/Weekly Log, coleções e migração) com **medição**: nota diária de 1 a 5 para saúde e bem-estar, sono detalhado, hábitos e comparação entre horas planejadas e reais por área da vida.

**Problema:** a gestão do tempo é escassa (faculdade, estudos por conta própria, trabalho, treino, família/relacionamento e um empreendimento), e o papel nem sempre está à mão.

**Objetivo:** registrar o dia em poucos toques, enxergar onde o tempo e a energia estão indo e ajustar prioridades com dados.

### 1.1 Escopo da v1

**Dentro:** tarefas diárias com área e horas, check-in diário 1–5 com nota opcional, sono detalhado, hábitos, diário curto, metas (ano/mês/semana), migração de pendências, revisão semanal/mensal, consulta de dias passados, backup criptografado e **um lembrete diário simples** de check-in (horário configurável).

**Fora:** sincronização em nuvem, login, notificações inteligentes ou adaptativas, integração com relógio ou Health Connect, iOS, compartilhamento.

### 1.2 Critério de sucesso da v1

A v1 é considerada validada quando, em **30 dias de uso real**:
- o check-in foi feito em pelo menos **25 dos 30 dias**;
- o check-in completo leva **menos de 60 segundos** (cronometrado manualmente em uma amostra);
- foram feitas **pelo menos 4 revisões semanais** e **1 revisão mensal**;
- o usuário ajustou ao menos uma prioridade ou rotina com base nos dados da revisão.

---

## 2. Requisitos funcionais

| ID | Requisito |
|---|---|
| RF01 | Cadastrar áreas da vida (ex.: Faculdade, Trabalho 1, Trabalho 2, Cybersec, Treino, Família/Relacionamento, Empreendimento), editáveis e reordenáveis |
| RF02 | Cadastrar metas em três níveis (ano, mês, semana), cada uma ligada a uma área, com status. O período é **calculado** a partir do nível e de uma data âncora (ver §4.1) |
| RF03 | Criar tarefas com título, data, área, meta (opcional), prioridade, horas estimadas e horas reais |
| RF04 | Estados da tarefa: pendente, feita, migrada, descartada |
| RF05 | Migração: listar pendências e decidir entre reagendar, descartar ou transformar em meta (ver UC09) |
| RF06 | Check-in diário com métricas de 1 a 5 (energia, humor, foco, treino, alimentação), configuráveis e reordenáveis |
| RF07 | Cada métrica do check-in aceita uma nota opcional de texto |
| RF08 | Registro de sono: hora de deitar, hora de acordar, minutos para dormir, despertares, minutos acordado na madrugada, qualidade 1–5 e nota opcional. O app calcula **tempo na cama** e **tempo dormindo** (ver §4.2) |
| RF09 | Hábitos: criar hábitos com frequência (diário ou X vezes na semana) e marcar o cumprimento |
| RF10 | Diário: texto livre por dia, até 2.000 caracteres |
| RF11 | Tela "Hoje" reunindo tarefas, check-in, hábitos, **resumo do sono** e diário do dia |
| RF12 | Revisão semanal e mensal: média das métricas, horas planejadas vs. reais por área, hábitos cumpridos, sono (tempo dormindo médio e regularidade) e tarefas migradas |
| RF13 | Exportar e importar backup **criptografado** em arquivo, com versão de schema |
| RF14 | Lembrete diário simples de check-in, com horário configurável e possibilidade de desligar |
| RF15 | Editar e excluir registros do dia (tarefa, check-in, sono, hábito marcado, diário) |
| RF16 | Consultar qualquer dia passado (tarefas, check-in, sono, hábitos e diário) e abri-lo a partir da revisão |
| RF17 | Prioridade de tarefa em três níveis (alta, média, baixa); a lista do dia ordena por prioridade e depois por criação |
| RF18 | Excluir área, métrica ou hábito **sem histórico**; com histórico, apenas arquivar |

## 3. Requisitos não funcionais

| ID | Requisito |
|---|---|
| RNF01 | **Rapidez:** check-in completo em menos de 60 segundos, com no máximo dois toques por métrica |
| RNF02 | **Offline e local-first:** todos os dados ficam no aparelho |
| RNF03 | **Privacidade:** banco criptografado (SQLCipher)
| RNF03a | **Chave do banco:** gerada aleatoriamente no primeiro uso e guardada cifrada com uma chave do **Android Keystore** 
| RNF03b | O bloqueio é reaplicado após **1 minuto** em segundo plano (valor padrão, configurável) |
| RNF04 | **Evolução:** métricas, áreas e hábitos configuráveis, sem número fixo no código |
| RNF05 | **Integridade:** itens com histórico são arquivados, não apagados (ver RF18); restrições de unicidade no banco (ver §4) |
| RNF06 | **Desempenho:** abrir "Hoje" em menos de 1 segundo com 1 ano de dados; revisão mensal em menos de 2 segundos |
| RNF07 | **Acessibilidade:** alvos de toque de pelo menos 48 dp, contraste adequado, rótulos para TalkBack e escala de fonte do sistema respeitada |
| RNF08 | **Backup:** arquivo com `schema_version`, verificação de integridade e criptografia obrigatória (AES-256-GCM, chave derivada de uma senha de backup definida pelo usuário). Restaurar um backup de versão anterior aplica as migrações necessárias |
| RNF09 | **Migrações de schema:** toda mudança de schema tem migração e teste dedicado, inclusive com SQLCipher |

---

## 4. Modelo de dados

| Entidade | Campos |
|---|---|
| **Area** | id, nome, cor, ordem, arquivada |
| **Meta** | id, nivel (ano/mes/semana), data_ancora, periodo_inicio, periodo_fim, titulo, area_id, status (ativa/concluída/abandonada) |
| **Tarefa** | id, titulo, data, area_id, meta_id?, prioridade (alta/média/baixa), horas_estimadas?, horas_reais?, status (pendente/feita/migrada/descartada), tarefa_origem_id?, criada_em, atualizada_em |
| **Metrica** | id, nome, ordem, ativa, arquivada |
| **CheckInItem** | id, data, metrica_id, valor (1–5), nota? |
| **Sono** | id, data_referencia, hora_deitar, hora_acordar, minutos_para_dormir, despertares, minutos_acordado, qualidade (1–5), nota? |
| **Habito** | id, nome, frequencia, ordem, arquivado |
| **HabitoRegistro** | id, habito_id, data, feito |
| **Diario** | id, data, texto (máx. 2.000 caracteres) |
| **Configuracao** | chave, valor (horário do lembrete, tempo de bloqueio, tema, início da semana) |

### 4.1 Restrições e regras de integridade

- `CheckInItem`: **UNIQUE(data, metrica_id)**
- `HabitoRegistro`: **UNIQUE(habito_id, data)**
- `Diario`: **UNIQUE(data)**
- `Sono`: **UNIQUE(data_referencia)** — um registro por dia; sonecas ficam fora da v1
- `Meta`: UNIQUE(nivel, periodo_inicio, titulo, area_id) para evitar duplicatas acidentais
- **Semana:** segunda a domingo (ISO). **Mês:** calendário. **Ano:** calendário.
- **Período da meta:** calculado a partir de `nivel` + `data_ancora`; o usuário escolhe a data, não digita início e fim.
- `Tarefa.horas_reais = null` significa **não informado**. A revisão mostra a cobertura, ex.: "horas reais informadas em 12 de 15 tarefas concluídas".
- `Tarefa.tarefa_origem_id` guarda a linhagem de migrações, permitindo contar quantas vezes uma tarefa foi migrada.

### 4.2 Regras de cálculo do sono

- `data_referencia` = **dia do despertar**.
- **Tempo na cama** = `hora_acordar` − `hora_deitar` (somando 24 h se a hora de acordar for anterior à de deitar).
- **Tempo dormindo** = tempo na cama − `minutos_para_dormir` − `minutos_acordado`.
- `despertares` é uma **contagem** (informativa); a duração acordada na madrugada vai em `minutos_acordado`.
- **Regularidade** (revisão) = desvio-padrão, em minutos, do horário de deitar e do horário de acordar no período. Exibida como "± X min".
- **Validações:** aviso (não bloqueio) se o tempo na cama passar de 16 h ou se o tempo dormindo der negativo.

---

## 5. Casos de uso

**Ator único:** Usuário (dono do app).

### 5.1 Visão geral

| ID | Caso de uso | Requisitos |
|---|---|---|
| UC01 | Gerenciar áreas da vida | RF01, RF18 |
| UC02 | Definir e acompanhar metas (ano/mês/semana) | RF02 |
| UC03 | Planejar as tarefas do dia | RF03, RF04, RF17 |
| UC04 | Concluir tarefa e registrar horas reais | RF03, RF04 |
| UC05 | Fazer check-in diário | RF06, RF07 |
| UC06 | Registrar sono | RF08 |
| UC07 | Marcar hábitos do dia | RF09 |
| UC08 | Escrever o diário do dia | RF10 |
| UC09 | Revisar e migrar pendências | RF05 |
| UC10 | Consultar revisão semanal/mensal | RF12 |
| UC11 | Configurar métricas e hábitos | RF06, RF09, RF18 |
| UC12 | Exportar e importar backup | RF13 |
| UC13 | Consultar um dia passado | RF16 |
| UC14 | Editar ou excluir registros do dia | RF15 |
| UC15 | Receber e configurar o lembrete diário | RF14 |

### 5.2 Detalhamento

#### UC03 — Planejar as tarefas do dia
- **Pré-condição:** existe ao menos uma área cadastrada.
- **Fluxo principal:**
  1. O usuário abre "Hoje" e toca em "Nova tarefa".
  2. Informa título, área e prioridade.
  3. Opcionalmente vincula a uma meta e informa as horas estimadas.
  4. O app salva a tarefa como *pendente* e a mostra na lista do dia, ordenada por prioridade.
- **Alternativos:**
  - Sem área escolhida: o app usa a última área usada ou pede para escolher.
  - Horas estimadas acima da carga do dia: o app avisa, sem bloquear.
- **Pós-condição:** tarefa pendente associada à data, com `criada_em` preenchido.

#### UC04 — Concluir tarefa e registrar horas reais
- **Pré-condição:** existe tarefa pendente no dia.
- **Fluxo principal:**
  1. O usuário marca a tarefa como feita.
  2. O app pergunta as horas reais, sugerindo o valor estimado.
  3. O usuário confirma ou ajusta.
  4. O app atualiza o status e os totais de horas por área.
- **Alternativos:**
  - Tarefa sem horas estimadas: o campo de horas reais abre **vazio**, sem sugestão.
  - O usuário pula as horas reais: a tarefa é concluída com `horas_reais = null` e a revisão sinaliza a cobertura.

#### UC05 — Fazer check-in diário
- **Pré-condição:** existem métricas ativas.
- **Fluxo principal:**
  1. O usuário abre o check-in (seção em "Hoje" ou atalho "Check-in rápido").
  2. Para cada métrica, toca em um valor de 1 a 5.
  3. Opcionalmente, abre o campo de nota daquela métrica e escreve.
  4. O app salva os valores do dia.
- **Alternativos:**
  - Check-in parcial: métricas não preenchidas ficam vazias, sem valor padrão.
  - Check-in já existente no dia: o app abre para edição (respeitando a unicidade data + métrica).
- **Regra:** a nota é sempre opcional e nunca bloqueia o salvamento.

#### UC06 — Registrar sono
- **Pré-condição:** nenhuma.
- **Fluxo principal:**
  1. O usuário abre "Sono" (pelo card em "Hoje") e informa a hora que deitou e a hora que acordou.
  2. Informa os minutos para dormir, o número de despertares e os minutos acordado na madrugada.
  3. Dá uma nota de qualidade de 1 a 5 e, se quiser, uma nota de texto.
  4. O app calcula tempo na cama e tempo dormindo (§4.2) e salva com `data_referencia` = dia do despertar.
- **Alternativos:**
  - Hora de acordar anterior à de deitar: cruzou a meia-noite; o cálculo soma 24 h.
  - Tempo na cama acima de 16 h ou tempo dormindo negativo: o app avisa e pede confirmação.
  - Já existe registro para o dia: o app abre para edição.
- **Pós-condição:** registro de sono do dia com os dois tempos calculados.

#### UC09 — Revisar e migrar pendências
- **Gatilho:** iniciado **pelo usuário**. O app apenas sinaliza com um card na tela "Hoje" quando há pendências de dias anteriores, no domingo (revisão semanal) e no último dia do mês (revisão mensal).
- **Pré-condição:** existem tarefas pendentes de dias anteriores.
- **Fluxo principal:**
  1. O usuário abre a revisão e vê a lista de pendências.
  2. Para cada tarefa, escolhe entre *reagendar*, *descartar* ou *transformar em meta*.
  3. *Reagendar:* o app marca a original como *migrada* e cria uma nova com `tarefa_origem_id` apontando para ela.
  4. *Descartar:* a original fica *descartada*.
  5. *Transformar em meta:* o app cria uma meta de **nível semana**, no período da semana seguinte (o usuário pode trocar para mês), com o título e a área da tarefa; a original fica *migrada*.
- **Alternativo:** o usuário pula a revisão; as pendências continuam listadas.
- **Regra:** só se migra uma tarefa depois de uma decisão explícita.

#### UC10 — Consultar revisão semanal/mensal
- **Fluxo principal:**
  1. O usuário escolhe o período (semana ISO ou mês calendário).
  2. O app mostra a média de cada métrica, as horas planejadas vs. reais por área (com a cobertura de horas informadas), os hábitos cumpridos e o sono (tempo dormindo médio e regularidade).
  3. O usuário toca em um ponto do gráfico para abrir o dia correspondente (UC14).
- **Alternativo:** poucos dados no período; o app mostra um estado vazio explicativo em vez de tendência enganosa.

#### UC13 — Consultar um dia passado
- **Fluxo principal:**
  1. O usuário escolhe uma data (pelo calendário na tela "Planejar" ou pela revisão).
  2. O app mostra tarefas, check-in (com notas), sono, hábitos e diário daquele dia.
  3. O usuário pode editar (UC15).

#### UC14 — Editar ou excluir registros do dia
- **Fluxo principal:** a partir de "Hoje" ou de um dia passado, o usuário abre um registro, altera ou exclui e confirma.
- **Regra:** excluir uma tarefa *migrada* não apaga a nova; a linhagem da nova passa a apontar para vazio.

#### UC15 — Lembrete diário
- **Fluxo principal:** o usuário define um horário em Configurações; no horário, o app mostra uma notificação simples "Hora do check-in"; tocar abre o check-in rápido.
- **Alternativo:** lembrete desligado ou permissão de notificação negada; o app segue funcionando sem lembrete.

#### Demais casos de uso (resumo)
- **UC01 / UC11:** criar, renomear, reordenar, arquivar e (sem histórico) excluir áreas, métricas e hábitos.
- **UC02:** criar uma meta escolhendo nível, data âncora e área; o período é calculado. Status: ativa, concluída, abandonada.
- **UC07:** marcar o hábito como feito no dia, com opção de desmarcar.
- **UC08:** escrever ou editar o texto do dia (até 2.000 caracteres).
- **UC12:** exportar um backup criptografado com senha definida pelo usuário e restaurar a partir dele, com confirmação antes de sobrescrever e migração de schema se necessário.

---

## 6. Telas e navegação

**Navegação:** *bottom navigation* com 4 itens — **Hoje, Planejar, Revisão, Ajustes**. O **Sono** é acessado pelo card de resumo em "Hoje". Navegação via Navigation Compose.

| Tela | Conteúdo |
|---|---|
| **Hoje** | Ordem de cima para baixo: card de check-in rápido, card de sono (resumo ou botão "Registrar sono"), tarefas do dia, hábitos, diário. Card de revisão pendente quando aplicável |
| **Check-in rápido** | Folha (*bottom sheet*) com as métricas 1–5 e notas opcionais, pensada para menos de 60 s |
| **Sono** | Formulário de registro (UC06) |
| **Planejar** | Metas por nível, calendário e agenda de tarefas |
| **Detalhe da meta** | Dados da meta e tarefas vinculadas |
| **Detalhe do dia** | Tudo o que foi registrado em um dia (UC14) |
| **Revisão** | Gráficos semanais/mensais, fluxo de migração |
| **Detalhe do hábito** | Histórico e sequência de cumprimento |
| **Ajustes** | Áreas, métricas, hábitos, lembrete, tempo de bloqueio, backup/restauração |

---

## 7. Arquitetura e stack

- **Linguagem/UI:** Kotlin + Jetpack Compose, uma única Activity, **Navigation Compose**
- **Arquitetura:** MVVM, com camada de repositório
- **Injeção de dependência:** **Hilt**
- **Persistência:** Room (SQLite) com **SQLCipher** (`SupportFactory`)
- **Gráficos:** **Vico** (nativo de Compose)
- **Lembrete:** WorkManager ou AlarmManager + notificação local
- **Backup:** AES-256-GCM com chave derivada da senha do usuário (PBKDF2 ou Argon2), `schema_version` no cabeçalho
- **Testes:** JUnit, Turbine (Flow), Room em memória para DAOs, teste de migração com SQLCipher, testes de UI com Compose
- **Alternativa:** Flutter (Dart + Drift), caso iOS entre no futuro, ao custo de menos integração nativa

---

## 8. Plano de desenvolvimento

| Fase | Entrega | Requisitos |
|---|---|---|
| 0 | Projeto base: Hilt, Navigation Compose, Room **com SQLCipher** e chave no Keystore (ainda sem exigir autenticação), tema e esqueleto de navegação | RNF02, RNF03a, RNF09 |
| 1 | Áreas e tela "Hoje" com tarefas, prioridade e edição | RF01, RF03, RF04, RF11, RF15, RF17, RF18 |
| 2 | Check-in com notas e registro de sono (com cálculos) | RF06, RF07, RF08 |
| 3 | Hábitos e diário; consulta de dias passados | RF09, RF10, RF16 |
| 4 | Metas, períodos e migração | RF02, RF05 |
| 5 | Revisão com gráficos e estados vazios | RF12 |
| 6 | Bloqueio com autenticação, lembrete diário e backup criptografado | RNF03, RNF03b, RF13, RF14, RNF08 |
| 7 | **Uso real por 30 dias** e verificação do critério de sucesso (§1.2) | §1.2 |

Dica: antes da fase 1, rode por 2 semanas as mesmas métricas 1–5 em uma planilha ou no Notion para descobrir quais você realmente usa.

---

## 9. Orientações para desenvolver com IA (Claude e Gemini)

**Como usar**
- Salve este documento na raiz do projeto como `CLAUDE.md` (ou `AGENTS.md`), para que qualquer ferramenta comece com o mesmo contexto.
- Peça **uma fatia vertical por vez** (tela + ViewModel + DAO + teste), na ordem das **fases do §8**.

**Contrato com a IA**
- **Não** adicionar dependências sem aprovação explícita.
- **Não** alterar o schema sem atualizar este documento e criar a migração com teste.
- **Não** implementar funcionalidades fora do escopo do §1.1.
- **Não** gravar dados sensíveis (notas, diário, sono) em logs, analytics ou arquivos fora do banco criptografado.
- Antes de codar uma fase, resumir o plano em poucas linhas e esperar confirmação.

**Testes obrigatórios para regras de negócio**
- Cálculo de tempo na cama e tempo dormindo, inclusive virada de meia-noite e validações
- Regularidade do sono (desvio-padrão)
- Migração de tarefas e linhagem (`tarefa_origem_id`)
- Totais de horas por área e cobertura de horas informadas
- Unicidade de check-in, hábito, diário e sono por dia
- Cálculo do período de metas (semana ISO, mês, ano)
- Migração de schema com SQLCipher e restauração de backup de versão anterior

---

## 10. Referências

- Na Prática. *Bullet Journal: o que é e como fazer*. Publicado em 01/04/2025. https://napratica.org.br/noticias/bullet-journal (acessado em 16/11/2026)
- Asimov Academy. *O ÚNICO sistema de ORGANIZAÇÃO que funcionou para mim - O Bullet Journal*. YouTube, 05/04/2024. https://www.youtube.com/watch?v=cRVn3AzMcnI (acessado em 16/11/2026)
- Carroll, Ryder. Criador do método Bullet Journal.

---

## 11. Decisões fechadas nesta versão

| Tema | Decisão |
|---|---|
| Chave do SQLCipher | Aleatória, guardada cifrada no Android Keystore, liberada após autenticação |
| Sono | Tempo na cama ≠ tempo dormindo; regularidade = desvio-padrão dos horários |
| Semana e mês | Semana ISO (segunda a domingo), mês calendário |
| Revisão | Iniciada pelo usuário, com card de lembrete |
| Backup | Sempre criptografado, com `schema_version` |
| Notificações | Apenas um lembrete diário simples |
| Gráficos | Vico |
| DI | Hilt |
| Navegação | Bottom nav com 4 itens; sono via card em "Hoje" |
