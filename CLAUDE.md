# CLAUDE.md

> Documento de contexto para agentes de IA (Claude, Gemini e similares) trabalhando no **Prumo**.
> Fontes da verdade: `prumo-especificacao-v1.2.md` e `prumo-modulos-v1.2.md`. Este arquivo resume o essencial para começar; em caso de dúvida, os dois documentos vencem.
> Versão: 1.2 · Data: 16/11/2026 · Plataforma: Android (Kotlin + Jetpack Compose)

---

## 1. O que é o Prumo

Bullet journal digital com métricas, para **uso pessoal, usuário único, 100% local**. Une a lógica do Bullet Journal (Future Log, Monthly Log, Daily Log, coleções, migração) com medição: check-in diário 1–5, sono detalhado, hábitos e comparação entre horas planejadas e reais por área da vida.

**Problema:** gestão de tempo escassa (faculdade, estudos, trabalho, treino, família, empreendimento) e o papel nem sempre está à mão.
**Objetivo:** registrar o dia em poucos toques, enxergar onde tempo e energia vão, ajustar prioridades com dados.

---

## 2. Escopo da v1

**Dentro:** tarefas diárias com área e horas, check-in 1–5 com nota opcional, sono detalhado, hábitos, diário curto, metas (ano/mês/semana), migração de pendências, revisão semanal/mensal, consulta de dias passados, backup criptografado e **um lembrete diário simples** de check-in.

**Fora:** sincronização em nuvem, login, **bloqueio do app por biometria/PIN** (versão futura — §12 da spec), notificações inteligentes, integração com relógio/Health Connect, iOS, compartilhamento.

> ⚠️ **Não implemente** nada da lista "Fora" sem pedido explícito. A biometria futura já tem o caminho desenhado (M02 → `PassphraseProvider`), mas **não** deve ser codada agora.

---

## 3. Stack e decisões fechadas

| Tema | Decisão |
|---|---|
| Linguagem/UI | Kotlin + Jetpack Compose, **uma única Activity** |
| Navegação | **Navigation Compose** (rotas tipadas) |
| Arquitetura | **MVVM** com camada de repositório e UDF (`StateFlow<UiState>`, `UiEvent`, `Channel<UiEffect>`) |
| DI | **Hilt** |
| Persistência | **Room** com **SQLCipher** (`SupportFactory`), `exportSchema = true` |
| Chave do banco | Senha aleatória (32 bytes) cifrada com chave do **Android Keystore** (`setUserAuthenticationRequired(false)`). **Sem autenticação do usuário na v1** |
| Gráficos | **Vico** (Compose-native) |
| Lembrete | `AlarmManager.setAndAllowWhileIdle` ou WorkManager + notificação local (sem alarme exato) |
| Backup | AES-256-GCM, chave derivada de senha do usuário (PBKDF2-HMAC-SHA256 ou Argon2), `schema_version` no cabeçalho |
| Testes | JUnit, Turbine (Flow), Room em memória, teste de migração com SQLCipher, Compose UI test |
| minSdk | **26 (Android 8.0)** |
| Idioma | Interface em **pt-BR**; todas as strings em `strings.xml` |
| Horas | Guardar **minutos inteiros** (`horas_estimadas_min`, `horas_reais_min`); UI mostra em horas |
| Frequência do hábito | `tipo` (DIARIO / VEZES_NA_SEMANA) + `vezes_por_semana` (nulo se diário) |
| Datas no banco | `LocalDate` como texto ISO `YYYY-MM-DD`, `LocalTime` como `HH:mm`, instantes como epoch ms |
| Hierarquia de metas | **Fora da v1** (sem meta-pai; ligação só tarefa → meta) |
| Sem internet | O manifesto **não declara** `INTERNET` |
| Backup do Android | `allowBackup="false"` + regras de extração vazias |
| Logs | Timber só em debug; **nenhum** log de dados do usuário em release |
| Bloqueio do app | **Fora da v1** (banco criptografado em repouso, app abre sem autenticação) |

---

## 4. Estrutura do projeto

Um único módulo Gradle (`:app`), organizado **por funcionalidade** (*package-by-feature*).

```
com.prumo.app
├── core
│   ├── database      (M01)
│   ├── security      (M02)
│   ├── common        (M03)
│   ├── designsystem  (M04)
│   └── navigation    (M05)
├── feature
│   ├── areas         (M06)
│   ├── tarefas       (M07)
│   ├── metas         (M08)
│   ├── checkin       (M09)
│   ├── sono          (M10)
│   ├── habitos       (M11)
│   ├── diario        (M12)
│   ├── hoje          (M13)
│   ├── dias          (M14)
│   ├── migracao      (M15)
│   ├── revisao       (M16)
│   ├── lembrete      (M17)
│   ├── backup        (M18)
│   └── ajustes       (M19)
└── di                (módulos Hilt)
```

Cada feature segue as mesmas camadas:

```
feature/<nome>
├── data      Entity (se dona da tabela), DAO, RepositoryImpl
├── domain    modelos, interface Repository, casos de uso
└── ui        Screen (Compose), ViewModel, UiState, UiEvent, UiEffect
```

### 4.1 Regras de dependência (não negociáveis)

1. `ui` depende de `domain`. `domain` **não** depende de `ui` nem de Android.
2. `data` implementa as interfaces de `domain`.
3. **Uma funcionalidade nunca acessa o DAO ou a tabela de outra.** Conversa pelo repositório (interface) ou caso de uso da outra.
4. `hoje`, `dias` e `revisao` são **agregadores**: só leem, nunca escrevem em tabelas alheias.
5. Escritas que atravessam funcionalidades (ex.: migração cria meta e marca tarefa) acontecem em **uma transação**, orquestrada pelo módulo que dispara o fluxo (`TransactionRunner` do M01).
6. `SettingsRepository` fica no **núcleo** (M01/M03), não no M19 — o M19 é só a UI de Ajustes. Isso evita ciclo M17 ↔ M19.
7. Datas e horas só por `Clock` injetado (nunca `LocalDate.now()` direto).

---

## 5. Mapa de módulos

| ID | Módulo | Tipo | Fase |
|---|---|---|---|
| M01 | Núcleo de dados | Transversal | 0 |
| M02 | Segurança (chave do banco) | Transversal | 0 |
| M03 | Núcleo comum (tempo e cálculos) | Transversal | 0 |
| M04 | Design system e UI base | Transversal | 0 |
| M05 | Navegação e shell | Transversal | 0 |
| M06 | Áreas | Funcional | 1 |
| M07 | Tarefas | Funcional | 1 |
| M08 | Metas | Funcional | 4 |
| M09 | Check-in e métricas | Funcional | 2 |
| M10 | Sono | Funcional | 2 |
| M11 | Hábitos | Funcional | 3 |
| M12 | Diário | Funcional | 3 |
| M13 | Hoje (agregador) | Agregador | 1–4 |
| M14 | Dias (histórico) | Agregador | 3 |
| M15 | Migração de pendências | Funcional | 4 |
| M16 | Revisão e análises | Agregador | 5 |
| M17 | Lembrete diário | Funcional | 6 |
| M18 | Backup | Funcional | 6 |
| M19 | Ajustes | Funcional | 1–6 |

O detalhamento de API de domínio, regras e "pronto quando" de cada módulo está em `prumo-modulos-v1.2.md` — **consulte antes de codar qualquer módulo**.

---

## 6. Modelo de dados (resumo)

| Entidade | Campos |
|---|---|
| **Area** | id, nome, cor, ordem, arquivada |
| **Meta** | id, nivel (ano/mes/semana), data_ancora, periodo_inicio, periodo_fim, titulo, area_id, status (ativa/concluída/abandonada) |
| **Tarefa** | id, titulo, data, area_id, meta_id?, prioridade (alta/média/baixa), horas_estimadas_min?, horas_reais_min?, status (pendente/feita/migrada/descartada), tarefa_origem_id?, criada_em, atualizada_em |
| **Metrica** | id, nome, ordem, ativa, arquivada |
| **CheckInItem** | id, data, metrica_id, valor (1–5), nota? |
| **Sono** | id, data_referencia, hora_deitar, hora_acordar, minutos_para_dormir, despertares, minutos_acordado, qualidade (1–5), nota? |
| **Habito** | id, nome, tipo (DIARIO/VEZES_NA_SEMANA), vezes_por_semana?, ordem, arquivado |
| **HabitoRegistro** | id, habito_id, data, feito |
| **Diario** | id, data, texto (máx. 2.000 caracteres) |
| **Configuracao** | chave, valor (horário do lembrete, tema, início da semana) |

### 6.1 Restrições de integridade

- `CheckInItem`: **UNIQUE(data, metrica_id)**
- `HabitoRegistro`: **UNIQUE(habito_id, data)**
- `Diario`: **UNIQUE(data)**
- `Sono`: **UNIQUE(data_referencia)** — sonecas fora da v1
- `Meta`: UNIQUE(nivel, periodo_inicio, titulo, area_id)

### 6.2 Regras de sono (M03 `SleepCalculator`)

- `data_referencia` = **dia do despertar**.
- **Tempo na cama** = `hora_acordar` − `hora_deitar` (soma 24 h se acordar antes de deitar).
- **Tempo dormindo** = tempo na cama − `minutos_para_dormir` − `minutos_acordado`.
- **Regularidade** = desvio-padrão (min) dos horários de deitar e de acordar no período, exibido como "± X min".
- Aviso (não bloqueio) se tempo na cama > 16 h ou tempo dormindo negativo.

### 6.3 Períodos

- **Semana:** segunda a domingo (ISO). **Mês/Ano:** calendário.
- Período da meta **calculado** a partir de `nivel` + `data_ancora` (o usuário escolhe a data, não digita início/fim).

---

## 7. Fases de desenvolvimento

| Fase | Entrega | Módulos |
|---|---|---|
| 0 | Projeto base: Hilt, Navigation Compose, Room + SQLCipher + chave no Keystore, tema, esqueleto de navegação | M01–M05 |
| 1 | Áreas e tela "Hoje" com tarefas, prioridade e edição | M06, M07, M13 (tarefas), M19 (áreas) |
| 2 | Check-in com notas e registro de sono (com cálculos) | M09, M10, M13, M19 (métricas) |
| 3 | Hábitos e diário; consulta de dias passados | M11, M12, M14, M13, M19 (hábitos) |
| 4 | Metas, períodos e migração | M08, M15, M13 (card de revisão) |
| 5 | Revisão com gráficos e estados vazios | M16 |
| 6 | Lembrete diário e backup criptografado | M17, M18, M19 |
| 7 | Uso real por 30 dias (sem módulo novo) | §1.2 da spec |

**Antes da fase 1:** rodar 2 semanas as mesmas métricas 1–5 em planilha/Notion para descobrir quais você realmente usa.

---

## 8. Contrato com a IA

### 8.1 Como trabalhar

- Peça **uma fatia vertical por vez** (tela + ViewModel + DAO + teste), **na ordem das fases do §7**.
- Antes de codar uma fase, **resuma o plano em poucas linhas e espere confirmação**.
- Consulte o detalhamento do módulo em `prumo-modulos-v1.2.md` antes de implementar.

### 8.2 O que **não** fazer

- ❌ Adicionar dependências sem aprovação explícita.
- ❌ Alterar o schema sem atualizar a spec e criar migração com teste.
- ❌ Implementar funcionalidades fora do escopo do §2.
- ❌ Gravar dados sensíveis (notas, diário, sono) em logs, analytics ou arquivos fora do banco criptografado.
- ❌ Usar `LocalDate.now()` / `System.currentTimeMillis()` direto (use `Clock` injetado).
- ❌ Acessar DAO/tabela de outra feature (regra 3 do §4.1).
- ❌ Fazer `rekey` do banco na v1 (biometria futura não exige — §12 da spec).

---

## 9. Testes obrigatórios (regras de negócio)

- Cálculo de tempo na cama e tempo dormindo, inclusive virada de meia-noite e validações (>16 h, negativo).
- Regularidade do sono (desvio-padrão, inclusive horários dos dois lados da meia-noite — `CircularTime` por campo: deitar e acordar separadamente).
- Migração de tarefas e linhagem (`tarefa_origem_id`), inclusive contagem de migrações pela cadeia.
- Totais de horas por área e cobertura ("horas reais informadas em X de Y tarefas").
- Unicidade de check-in, hábito, diário e sono por dia (upsert não duplica).
- Cálculo do período de metas (semana ISO cruzando mês/ano, fevereiro bissexto).
- Migração de schema com SQLCipher e restauração de backup de versão anterior.
- M17: cálculo do próximo disparo (hoje/amanhã), não disparar com check-in completo, reagendamento após reinício.
- M18: ida e volta, senha errada, arquivo corrompido, backup de versão anterior migrado, falha no meio da restauração não altera o banco.

---

## 10. Definição de "pronto" (todo módulo)

1. Regras de negócio com testes unitários passando.
2. DAOs com testes em banco em memória.
3. ViewModels com testes dos estados (carregando, vazio, conteúdo, erro).
4. Estados vazios e de erro com texto em pt-BR.
5. Acessibilidade: alvos de 48 dp, `contentDescription` para TalkBack, fonte escalável.
6. Nenhum dado do usuário em logs.
7. Se houve mudança de schema: migração criada e testada, e a spec atualizada.
8. O módulo só conversa com os outros pelas interfaces de domínio (regra 3 do §4.1).

---

## 11. Riscos conhecidos

| Risco | Mitigação |
|---|---|
| Migração de schema com SQLCipher falhar | Teste por versão; backup antes de atualizar |
| Trocar de aparelho/limpar dados: banco e chave não migram | Backup regular (M18); aviso em Ajustes |
| Sem bloqueio na v1: chave do Keystore sem autenticação | `allowBackup="false"`; proteção é contra extração do arquivo, **não** contra aparelho comprometido (limitação consciente) |
| Erros de fuso/horário de verão nos períodos | `Clock` injetado + testes de borda |
| Tela "Hoje" pesada | Fluxos independentes por seção; medir com 1 ano de dados gerados (RNF06: < 1 s) |
| Migração parcial deixar dados inconsistentes | Transação única (M15) |
| Esquecer senha do backup | Aviso explícito antes de exportar; senha não é guardada |

---

## 12. Pendências de sincronização (spec ↔ módulos)

Antes de começar a fase 0, atualizar a spec v1.2 para refletir decisões já tomadas no documento de módulos:

- [ ] §4: `horas_estimadas` → `horas_estimadas_min`; `horas_reais` → `horas_reais_min`.
- [ ] §4: `Habito.frequencia` → `tipo` + `vezes_por_semana`.
- [ ] §3/§7: adicionar minSdk 26, `allowBackup="false"`, ausência de `INTERNET` no manifesto.
- [ ] §3: subir a regra da `CharArray` da senha de backup para o RNF08.
- [ ] §3: registrar no RNF03a a limitação "não protege contra aparelho comprometido".
- [ ] §6: alinhar migração (M15 tem tela própria; spec diz dentro de "Revisão").
- [ ] M19: remover RNF03b da coluna de requisitos (requisito não existe mais na v1.2).
- [ ] M13: subir o check-in para 2º na ordem da tela (RNF01 < 60 s).
- [ ] M07 ↔ M15: compartilhar o mesmo caso de uso de `reschedule`.
- [ ] M01: explicitar mecanismo de seed (callback + flag persistida).
- [ ] M17: definir "check-in completo" (todas as métricas ativas? ao menos uma?).
- [ ] M18: definir comportamento de `inspect` com versão mais nova.
- [ ] §1.2: adicionar meta de backup ao critério de sucesso.
- [ ] §1: glossário "ativa/desativada" (aparece no dia) vs. "arquivada" (só histórico/revisão).

---

## 13. Referências

- `prumo-especificacao-v1.2.md` — requisitos, modelo, casos de uso, telas, stack, plano.
- `prumo-modulos-v1.2.md` — API de domínio, regras, testes e "pronto quando" por módulo.
- Na Prática. *Bullet Journal: o que é e como fazer*. https://napratica.org.br/noticias/bullet-journal
- Asimov Academy. *O ÚNICO sistema de ORGANIZAÇÃO que funcionou para mim - O Bullet Journal*. https://www.youtube.com/watch?v=cRVn3AzMcnI
- Carroll, Ryder. Criador do método Bullet Journal.