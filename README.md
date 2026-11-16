# Prumo

> Bullet journal digital com métricas — planejamento, hábitos, sono, check-in diário e diário pessoal em um único app Android, 100% local.

**Versão:** 1.2 (MVP) · **Status:** em especificação, desenvolvimento não iniciado · **Plataforma:** Android

---

## O que é

O Prumo une a lógica do **Bullet Journal** (Future Log, Monthly Log, Daily Log, coleções e migração) com **medição**: nota diária de 1 a 5 para saúde e bem-estar, sono detalhado, hábitos e comparação entre horas planejadas e reais por área da vida.

A ideia central é simples: **registrar o dia em poucos toques** e, com o tempo, enxergar onde o tempo e a energia estão indo para ajustar prioridades com dados — não com impressão.

## Por que existe

A gestão do tempo é escassa: faculdade, estudos por conta própria, trabalho, treino, família/relacionamento e um empreendimento. O papel nem sempre está à mão, e planilhas não acompanham o dia. O Prumo é um app pessoal, de usuário único, sem login e sem nuvem — os dados ficam no aparelho.

## O que a v1 entrega

- **Tarefas diárias** com área, prioridade, horas estimadas e horas reais
- **Check-in diário** de 1 a 5 em métricas configuráveis (energia, humor, foco, treino, alimentação), com nota opcional
- **Sono detalhado**: hora de deitar/acordar, tempo para dormir, despertares, qualidade e nota; o app calcula tempo na cama e tempo dormindo
- **Hábitos** diários ou X vezes na semana, com sequência
- **Diário** curto por dia (até 2.000 caracteres)
- **Metas** de ano, mês e semana ligadas a áreas
- **Migração de pendências** (reagendar, descartar ou transformar em meta)
- **Revisão semanal e mensal** com médias, horas planejadas vs. reais por área, hábitos e sono
- **Consulta de dias passados** e edição
- **Backup criptografado** em arquivo
- **Um lembrete diário simples** de check-in

**Fora da v1:** nuvem, login, bloqueio por biometria/PIN, notificações inteligentes, Health Connect, iOS, compartilhamento.

## Stack

- **Kotlin + Jetpack Compose** (uma única Activity, Navigation Compose)
- **MVVM** com camada de repositório e UDF (`StateFlow` / `UiEvent` / `UiEffect`)
- **Hilt** para injeção de dependência
- **Room + SQLCipher** (banco criptografado em repouso); chave no **Android Keystore**
- **Vico** para gráficos
- **WorkManager/AlarmManager** para o lembrete diário
- Testes: JUnit, Turbine, Room em memória, teste de migração com SQLCipher, Compose UI test

**minSdk 26 (Android 8.0)** · Interface em **pt-BR** · Sem permissão de internet

## Como o projeto está organizado

Um único módulo Gradle (`:app`), organizado **por funcionalidade**:

```
core/      database, security, common, designsystem, navigation
feature/   areas, tarefas, metas, checkin, sono, habitos, diario,
           hoje, dias, migracao, revisao, lembrete, backup, ajustes
di/        módulos Hilt
```

Cada feature segue as camadas `data` / `domain` / `ui`. Regra de ouro: **uma feature nunca acessa o DAO ou a tabela de outra** — conversa por interface de domínio.

## Roadmap (fases)

| Fase | Entrega |
|---|---|
| 0 | Projeto base: Hilt, Navigation, Room + SQLCipher + Keystore, tema, navegação |
| 1 | Áreas e tela "Hoje" com tarefas |
| 2 | Check-in e sono |
| 3 | Hábitos, diário e consulta de dias passados |
| 4 | Metas e migração de pendências |
| 5 | Revisão com gráficos |
| 6 | Lembrete diário e backup criptografado |
| 7 | Uso real por 30 dias (validação) |

**Critério de sucesso da v1:** em 30 dias de uso real, check-in em ≥ 25 dias, check-in completo em < 60 s, ≥ 4 revisões semanais + 1 mensal, e ao menos um ajuste de prioridade feito com base nos dados.

## Documentação

- [`prumo-especificacao-v1.2.md`](./prumo-especificacao-v1.2.md) — visão, requisitos, modelo de dados, casos de uso, telas, stack e plano
- [`prumo-modulos-v1.2.md`](./prumo-modulos-v1.2.md) — detalhamento de cada módulo (API de domínio, regras, testes, "pronto quando")
- [`CLAUDE.md`](./CLAUDE.md) — contexto para agentes de IA (Claude, Gemini) e contrato de trabalho

## Como contribuir (eu do futuro)

1. Leia o `CLAUDE.md` antes de abrir o editor.
2. Trabalhe **uma fatia vertical por vez**, na ordem das fases.
3. Antes de codar uma fase, resuma o plano em poucas linhas.
4. Toda regra de negócio precisa de teste (ver §9 do `CLAUDE.md`).
5. Mudou o schema? Atualize a spec e crie a migração com teste.
6. Sem dados do usuário em logs. Sem internet no manifesto.

## Licença

Uso pessoal. Sem licença de distribuição definida.