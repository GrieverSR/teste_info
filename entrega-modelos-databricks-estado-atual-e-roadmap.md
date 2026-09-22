# Entrega de Modelos de ML em Databricks — Estado Atual, Lacunas e Roadmap

> **Responsável:** [Diogo], Analytics Enablement · **Estado:** Rascunho para revisão · **Última atualização:** [data]
> **Destinatários:** Direção do CAI, equipa de Data Science, equipa de Analytics Enablement
> **Objetivo:** Descrever como funciona hoje a entrega de modelos em Databricks, o que está errado, como deve ser uma solução preparada para produção e o roadmap para lá chegar.

---

## 1. Sumário

O processo de entrega de modelos da equipa de Data Science em Databricks **funciona**: todos os meses os modelos são treinados, registados e entregues. No entanto, **não está preparado para produção**.

Hoje, o processo depende de dois robôs de browser que movem dados de e para o Cloudera através da interface web do Hue, de cinco passos manuais em cada entrada de um modelo em produção e de uma configuração da plataforma que não segue as recomendações publicadas pela Databricks para machine learning em produção. Nada disto é visível enquanto as entregas chegam a tempo. Torna-se visível quando algo para — ou, pior, quando uma entrega termina com sucesso e os números estão errados sem que ninguém dê por isso.

Esta página propõe uma solução-alvo alinhada com a arquitetura de referência de MLOps da Databricks, adaptada à nossa escala, e um roadmap por fases em que cada fase tem um entregável claro, critérios de conclusão e indicadores mensuráveis.

**Indicadores atuais**

| Indicador | Valor atual |
|---|---|
| Robôs de browser no fluxo de dados | 2 (um de entrada, um de saída) |
| Passos manuais por entrada de modelo em produção | 5 |
| Estrangulamentos / pontos únicos de falha por entrega | 4 |
| Modelos registados no schema de produção | 95 (número em uso efetivo: [a confirmar]) |
| Jobs de produção a correr com uma identidade de sistema | 0% |
| Jobs de produção em compute dedicado (job clusters) | 0% |
| Entregas rastreáveis até ao código, modelo e dados exatos | 0% |
| Implementação automatizada (CI/CD) | Inexistente |

---

## 2. Como está implementada a solução atual

### 2.1 Fluxo de ponta a ponta

| # | Etapa | Ferramenta | O que acontece | Manual | Estrangulamento |
|---|---|---|---|---|---|
| 1 | Origem | Cloudera (on-premise) | As tabelas de origem estão no data warehouse on-premise | | |
| 2 | Ingestão | Robô Selenium + Hue | O robô lê uma tabela de agendamento no Databricks e descarrega ficheiros parquet a navegar na interface web do Hue | | ✅ |
| 3 | Armazenamento | Volumes do Unity Catalog | Os ficheiros parquet extraídos ficam em volumes dentro de `data_science.prod` | | |
| 4 | Desenvolvimento | Notebooks + toolkit de DS | Análise e treino em notebooks em `Workspace/Shared`, com o toolkit copiado como scripts avulsos | ✅ | |
| 5 | Registo | MLflow | Modelo registado à mão; os encoders e transformações treinados são guardados fora do modelo | ✅ | ✅ |
| 6 | Configuração | GitHub (YAML de entrega) | O id do modelo, caso de uso, probabilidade de corte e período de execução são copiados para o YAML de entrega | ✅ | |
| 7 | Sincronização | Databricks Repos | Os repositórios são atualizados no workspace à mão | ✅ | |
| 8 | Execução | Databricks Jobs | Job criado/editado na interface; o notebook lê o YAML e chama o toolkit com caminhos fixos no código; corre num cluster all-purpose partilhado com uma conta pessoal | ✅ | ✅ |
| 9 | Entrega | Robô Selenium + Hue | O robô carrega os resultados de volta no Cloudera através da interface web do Hue | | ✅ |
| 10 | Monitorização | Email | É enviado um email apenas se o job falhar | | |

### 2.2 Configuração da plataforma

| Área | Implementação atual |
|---|---|
| Workspace | Um único workspace Databricks em GCP |
| Catálogo | Um catálogo, `data_science`, com os schemas `default`, `dev`, `dev_cq` e `prod` usados como ambientes |
| Schema de produção | 7 volumes (ex.: `clean_data`, `prep_data`, `csv`, `models`, `models_migration`, `model_migration`, `validation`), 1 tabela e 95 modelos registados numa lista plana |
| Localização do código | Notebooks em `Workspace/Shared` e em `/Repos/prod`; as pastas pessoais dos utilizadores também estão dentro de `/Repos/prod` |
| Repositórios | Dois repositórios GitHub: o toolkit de DS e um monorepo com notebooks e ficheiros YAML de entrega |
| Toolkit | Scripts Python avulsos, sem ser um pacote versionado |
| Jobs | Tarefas de notebook com origem = Workspace (uma pasta editável, não um commit de Git), criadas e editadas na interface |
| Compute | Cluster all-purpose partilhado (`AIE`, DBR 16.4 LTS ML) usado para as entregas agendadas |
| Identidade | Os jobs agendados correm com a conta pessoal de um antigo membro da equipa |
| Registo de modelos | MLflow no Unity Catalog; artefactos dos modelos incompletos (encoders guardados à parte) |
| CI/CD | Inexistente |
| Monitorização | Apenas emails de falha; sem verificações da qualidade dos resultados |

---

## 3. O que está errado

### 3.1 Ponto crítico 1 — Os robôs que nos ligam ao Cloudera

Os robôs são agendados (leem uma tabela de agendamento no Databricks), portanto o problema não é esforço manual. É fragilidade e falta de controlo:

- **Não é uma interface.** Navega numa página web feita para pessoas. Qualquer alteração ao Hue parte-o, sem aviso.
- **Depende de dois sistemas ao mesmo tempo.** Precisa que o Databricks e a interface web do Hue estejam ambos disponíveis e inalterados.
- **Não garante a escrita.** Se parar a meio, ficam dados parciais escritos e a execução pode aparentar ter corrido bem.
- **Autentica-se como uma pessoa e não deixa rasto de auditoria.** Para o Cloudera, parece trabalho manual. Este caminho de dados nunca foi revisto.

**No roadmap atual:** mover os robôs para Cloud Run. Isso exige acesso de rede do GCP ao Cloudera. Com esse mesmo acesso, o Databricks pode ler e escrever diretamente no data warehouse, e os robôs deixam de ser necessários. Mudá-los de sítio mantém a parte frágil e acrescenta mais uma plataforma para manter, para um data warehouse que está previsto migrar para BigQuery.

### 3.2 Ponto crítico 2 — Cada entrada em produção é feita à mão

Colocar em produção um modelo novo ou retreinado exige cinco passos manuais (etapas 4 a 8 acima). Consequências:

- **Nada regista o que correu.** Ninguém consegue dizer com certeza que código, versão do toolkit e modelo produziram a entrega de um determinado mês.
- **Não existe uma barreira à volta da produção.** Desenvolvimento e entregas partilham o mesmo catálogo, as mesmas pastas e o mesmo cluster. Uma alteração feita durante uma experiência pode chegar a uma entrega real.
- **Errado parece certo.** Somos alertados quando um job para, mas não quando termina com resultados errados (por exemplo, quando surge uma categoria nova nos dados de entrada e é tratada em silêncio).

### 3.3 Lacunas face às recomendações da Databricks

| A Databricks recomenda | O que fazemos hoje |
|---|---|
| Catálogos separados por ambiente (dev, staging, prod) | Um catálogo, com os ambientes como schemas lá dentro |
| Código promovido a partir do Git através de uma pipeline automatizada | Repositórios e notebooks atualizados à mão |
| Jobs definidos como código e implementados como Databricks Asset Bundles | Jobs criados e editados na interface |
| Compute dedicado (job compute) para cargas agendadas | Cluster all-purpose partilhado, faturado a uma tarifa mais alta |
| Cargas de produção a correr com service principals | Conta pessoal |
| Modelos registados como artefactos completos e autónomos | Encoders e transformações guardados fora do modelo |
| Modelos versionados e promovidos com aliases | 95 modelos numa lista plana, promovidos copiando ids |
| Verificações de qualidade às entradas e saídas de cada execução | Email apenas quando o job falha |

Referência: Databricks, *The Big Book of MLOps* e documentação sobre workflows de MLOps.

### 3.4 Registo de riscos

| Risco | Impacto | Probabilidade | Mitigação atual |
|---|---|---|---|
| A conta pessoal usada pelos jobs é desativada | Todas as entregas param | Alta | Nenhuma |
| Uma alteração na interface do Hue parte um robô | A ingestão ou a entrega param | Média | Email de falha |
| Escrita parcial no Cloudera reportada como sucesso | Entrega de resultados incompletos | Média | Nenhuma |
| Entrega termina com scores errados | O negócio atua com base em números errados | Média | Nenhuma |
| Uma alteração experimental chega à produção | Entrega errada ou falhada | Média | Nenhuma |
| Conhecimento concentrado em poucas pessoas | Recuperação lenta, alterações bloqueadas | Alta | Nenhuma |
| Custo de compute all-purpose para jobs agendados | Gasto evitável todos os meses | Certa | Nenhuma |

---

## 4. Solução-alvo

### 4.1 Princípios

1. **Seguir a arquitetura de referência da Databricks** e documentar qualquer desvio, com a respetiva justificação.
2. **Manter a experimentação rápida.** O controlo aplica-se na fronteira com a produção, não no desenvolvimento diário.
3. **Nada chega à produção à mão.** Código, jobs e modelos passam por uma pipeline.
4. **A produção depende da instituição, não de pessoas.**
5. **Falhar de forma visível.** Uma entrega errada tem de falhar de forma tão visível como uma entrega que parou.
6. **Preparar o futuro com BigQuery.** A ligação ao Cloudera fica isolada numa única interface, para poder ser substituída.

### 4.2 Arquitetura-alvo

| Área | Alvo |
|---|---|
| Workspace | Um único workspace (desvio deliberado — ver 4.5) |
| Catálogos | Um catálogo por ambiente: `ds_dev`, `ds_cq`, `ds_prod` |
| Schemas | Um schema por caso de uso em cada catálogo (ex.: `ds_prod.ciamr_tpa_churn`), mais schemas partilhados (`common`) |
| Ligação de dados | Ligação direta Databricks ↔ Cloudera, substituindo ambos os robôs; mais tarde BigQuery |
| Código | Toolkit como wheel versionado; código, configuração e definições de jobs de cada caso de uso no repositório de projetos |
| Jobs | Definidos como Databricks Asset Bundles, um bundle por caso de uso, com targets `dev`, `cq` e `prod` |
| CI/CD | GitHub Actions: validação em cada pull request, implementação em `cq` no merge, implementação em `prod` com uma tag de release |
| Identidade | Service principal para todas as cargas de produção; o GitHub Actions autentica-se por workload identity federation |
| Compute | Job clusters (ou serverless jobs) para trabalho agendado; clusters pessoais com políticas para desenvolvimento |
| Modelos | Artefactos pyfunc completos, incluindo todas as transformações; versões e aliases (champion/challenger) no Unity Catalog |
| Configuração | O YAML de entrega mantém-se; os valores de cada ambiente são injetados pelos targets do bundle, nada fixo no código |
| Monitorização | Verificações às entradas e saídas em cada entrega, ligadas aos alertas existentes |
| Rastreabilidade | SHA do Git, versão do toolkit e versão do modelo registados em cada execução |

**Fluxo entre ambientes**

| Ambiente | Catálogo | Quem implementa | Como |
|---|---|---|---|
| Desenvolvimento | `ds_dev` | Cada data scientist | Deploy pessoal do bundle, agendamentos em pausa |
| Validação (CQ) | `ds_cq` | Pipeline | Automaticamente no merge para main |
| Produção | `ds_prod` | Pipeline, com o service principal | Automaticamente com uma tag de release |

### 4.3 Como vai trabalhar um data scientist

1. **Explorar** — notebooks sobre dados de desenvolvimento, como hoje.
2. **Registar** — cada treino fica registado no MLflow; o modelo completo, incluindo transformações, é guardado como um único artefacto.
3. **Propor** — pull request com o código e o YAML de entrega, revisto por um colega.
4. **Validar** — verificações automáticas e, depois, implementação automática em CQ a correr sobre dados reais.
5. **Lançar** — uma tag de release implementa em produção. Sem cópias, sem editar jobs à mão.

### 4.4 Responsabilidades

| A equipa de Analytics Enablement disponibiliza | A equipa de Data Science é responsável por |
|---|---|
| Catálogos, schemas, permissões e service principals | Os modelos e a sua qualidade |
| O toolkit como pacote versionado e lançado | A configuração de entrega de cada caso de uso |
| Templates de bundles e pipelines de CI/CD | Decidir quando um modelo está pronto para produção |
| O standard de produção (Definition of Done) | Aplicar os templates a cada caso de uso |
| A ligação de dados ao Cloudera, mais tarde ao BigQuery | Definir as verificações de qualidade de dados de cada caso de uso |

### 4.5 Desvios deliberados face à arquitetura de referência

| Desvio | Justificação |
|---|---|
| Um workspace em vez de um por ambiente | Equipa de cinco pessoas e cadência mensal; os catálogos e as permissões garantem a separação sem o custo administrativo. A rever se a auditoria exigir isolamento físico, ou quando a equipa de Business Insights passar a usar a plataforma. |
| Modelos promovidos por referência (deploy models) em vez de retreinados em produção (deploy code) | O retreino é conduzido por pessoas e é mensal; o YAML de entrega já promove por referência ao modelo. O código de entrega continua a ser implementado a partir do Git. Só é seguro porque os modelos passam a ser artefactos completos. |
| Sem plataforma de monitorização de drift nem retreino automático, para já | Doze entregas por ano não o justificam; as verificações cobrem o risco imediato. |

---

## 5. Roadmap

As datas são indicativas e dependem das dependências listadas na secção 6. Cada fase termina com um entregável que pode ser demonstrado e reportado numa linha.

### 5.1 Fases

#### Fase 0 — Levantamento e correções urgentes de segurança · [out. 2026]
**Objetivo:** saber exatamente o que corre e eliminar o risco mais urgente.

| Entregável | Critério de conclusão |
|---|---|
| Inventário de todos os jobs (agendamento, identidade, cluster, entradas, saídas, responsável) | 100% dos jobs de produção documentados |
| Auditoria ao registo: quais dos 95 modelos estão em uso efetivo | Lista de modelos ativos acordada com a equipa de DS |
| Acompanhamento de uma entrega completa com a equipa de DS | Processo atual validado e documentado |
| Service principal e federação com o GitHub pedidos à IT | Pedidos submetidos e acompanhados |
| Convenção de catálogos e nomes acordada | Standard publicado no Confluence |
| Definition of Done para um caso de uso em produção | Publicada e revista pela equipa de DS |

#### Fase 1 — Fundações · [out. – nov. 2026]
**Objetivo:** a produção deixa de depender de pessoas e de recursos partilhados.

| Entregável | Critério de conclusão |
|---|---|
| Todos os jobs de produção correm com o service principal | 100% dos jobs; nenhuma conta ou token pessoal em produção |
| Jobs migrados do cluster all-purpose para job compute | 100% dos jobs agendados |
| Políticas de cluster para o compute de desenvolvimento | Políticas aplicadas; auto-terminação obrigatória |
| Catálogos `ds_dev`, `ds_cq` e `ds_prod` criados com modelo de permissões | Permissões aplicadas; DS só com leitura em `ds_prod` |
| Pastas pessoais removidas de `/Repos/prod` | A pasta contém apenas código de produção |

#### Fase 2 — O toolkit como produto · [nov. – dez. 2026]
**Objetivo:** o toolkit partilhado é versionado, testado e lançado automaticamente. *(Já previsto no roadmap da equipa.)*

| Entregável | Critério de conclusão |
|---|---|
| Toolkit empacotado como wheel versionado | Pacote instalável com versionamento semântico |
| Conjunto mínimo de testes às funções usadas na entrega | Testes a correr em CI em cada pull request |
| Pipeline de release em GitHub Actions | Wheel gerado e publicado com uma tag |
| Caminhos fixos no código passados para configuração | Zero caminhos de ambiente fixos no toolkit |

#### Fase 3 — Caso de uso piloto de ponta a ponta · [jan. – fev. 2027]
**Objetivo:** um caso de uso (proposta: `ciamr_tpa_churn`) cumpre integralmente a Definition of Done.

| Entregável | Critério de conclusão |
|---|---|
| Schemas do caso de uso nos três catálogos | Dados e modelos movidos para `ds_*.ciamr_tpa_churn` |
| Artefacto de modelo completo, incluindo transformações | O novo modelo reproduz os scores atuais de produção dentro da tolerância acordada |
| Databricks Asset Bundle com targets dev/cq/prod | Implementado nos três ambientes |
| CI/CD para o caso de uso | O merge implementa em CQ; a tag implementa em produção; rollback testado uma vez |
| Verificações às entradas e saídas | Uma falha provocada de propósito e alertada |
| Rastreabilidade | Última entrega reconstruída a partir do SHA, versão do toolkit e versão do modelo registados |
| Job antigo desativado | Execução em paralelo com resultados iguais; job antigo removido |

#### Fase 4 — Substituir os robôs · [1.º trimestre 2027, dependente do acesso de rede]
**Objetivo:** os dados passam a circular por uma ligação suportada e auditável.

| Entregável | Critério de conclusão |
|---|---|
| Caminho de rede do GCP/Databricks para o Cloudera aprovado | Conectividade testada a partir do Databricks |
| Leitura direta do Cloudera para o Databricks | Robô de entrada desligado para o caso de uso piloto |
| Escrita direta do Databricks para o Cloudera, com reconciliação | Robô de saída desligado; contagens de linhas reconciliadas |
| Ligação isolada numa única interface do toolkit | A troca para BigQuery exige alterações num só módulo |
| Robôs desativados para todos os casos de uso | 0 robôs no fluxo de dados |

#### Fase 5 — Extensão aos restantes casos de uso · [mar. – jun. 2027]
**Objetivo:** todos os casos de uso ativos cumprem a Definition of Done.

| Entregável | Critério de conclusão |
|---|---|
| Restantes casos de uso ativos migrados com o template do piloto | 100% dos casos de uso ativos no standard |
| Modelos inativos arquivados | O registo contém apenas modelos ativos e versionados |
| Catálogo antigo `data_science` descontinuado | Nenhum job lê ou escreve nele |

#### Fase 6 — Escalar · [2.º semestre 2027]
**Objetivo:** construir sobre fundações estáveis.

Itens candidatos, a priorizar: retreino automático, AutoML, monitorização de drift, entrada da equipa de Business Insights, migração da ligação de dados para BigQuery.

### 5.2 Marcos

| Marco | Data-alvo | Evidência |
|---|---|---|
| M1 — Produção sem dependência de contas pessoais | [fim de nov. 2026] | Inventário de jobs com 100% em service principal |
| M2 — Toolkit lançado através de CI/CD | [fim de dez. 2026] | Primeira versão do wheel publicada com tag |
| M3 — Primeiro caso de uso totalmente no standard | [fim de fev. 2027] | O piloto cumpre todos os critérios da Definition of Done |
| M4 — Robôs fora do fluxo de dados | [fim de mar. 2027] | Ambos os robôs desativados |
| M5 — Todos os casos de uso ativos no standard | [fim de jun. 2027] | 100% dos casos de uso ativos em conformidade |

### 5.3 Indicadores mensuráveis

| KPI | Valor de partida (hoje) | Alvo | Medido através de |
|---|---|---|---|
| Jobs de produção a correr com service principal | 0% | 100% (M1) | Inventário de jobs |
| Jobs agendados em job compute | 0% | 100% (M1) | Inventário de jobs |
| Custo mensal de compute das entregas agendadas | [valor de partida €] | [redução-alvo %] | Faturação Databricks, por tag |
| Passos manuais por entrada de modelo em produção | 5 | 0 (M5) | Processo de release |
| Tempo para colocar em produção um modelo novo ou retreinado | [valor de partida, a medir] | < 1 dia | Registos temporais da pipeline |
| Robôs no fluxo de dados | 2 | 0 (M4) | Arquitetura |
| Casos de uso que cumprem a Definition of Done | 0 | 100% dos ativos (M5) | Checklist da DoD |
| Modelos registados como artefactos completos | 0% | 100% dos ativos | Auditoria ao registo |
| Entregas rastreáveis até ao código, toolkit e modelo | 0% | 100% | Tags das execuções no MLflow |
| Entregas com verificações às entradas e saídas | 0% | 100% | Definições dos jobs |
| Entregas falhadas ou erradas sem alerta | Desconhecido | 0 | Registo de incidentes |

---

## 6. Dependências e riscos do roadmap

| Dependência / risco | Afeta | Responsável | Mitigação |
|---|---|---|---|
| Service principal e federação com o GitHub pela IT | Fases 1 a 5 | IT / [responsável] | Submeter na Fase 0; escalar através da direção se ficar bloqueado |
| Acesso de rede do GCP ao Cloudera | Fase 4 | Redes / Segurança | Enquadrar o pedido como uma ligação de dados, não como alojamento dos robôs |
| Disponibilidade da equipa de Data Science | Fases 3 a 5 | Responsável de DS | Piloto primeiro; migrar um caso de uso de cada vez |
| Tipo de alojamento do GitHub (Cloud vs Enterprise Server) | Autenticação do CI/CD | Analytics Enablement | Verificar na Fase 0 |
| Compatibilidade do serverless com a ligação ao Cloudera | Escolha de compute | Analytics Enablement | Testar no piloto; por omissão, job clusters |
| Calendário da migração para BigQuery | Desenho da Fase 4 | Projeto do data warehouse | Manter a ligação isolada numa única interface |

---

## 7. Questões em aberto

- Onde correm hoje os robôs e que caminho de rede usam para chegar ao Hue?
- Alguma vez foi pedida uma ligação direta Databricks–Cloudera? Se sim, porque não foi implementada?
- Quais dos 95 modelos registados estão em uso efetivo?
- O pedido de rede para o Cloud Run já foi submetido?
- Quanto tempo demora hoje colocar um modelo novo em produção (valor de partida para o KPI)?

---

## 8. Registo de decisões

| Data | Decisão | Alternativas consideradas | Justificação |
|---|---|---|---|
| [data] | Um workspace, catálogos por ambiente | Um workspace por ambiente | Dimensão da equipa, cadência mensal, custo administrativo |
| [data] | Nomes de catálogo `ds_dev` / `ds_cq` / `ds_prod`, um schema por caso de uso | Catálogos globais `dev`/`prod`; ambiente como schema | Evita ocupar nomes globais; prepara a entrada da equipa de BI; resolve a lista plana de modelos |
| [data] | Um bundle por caso de uso | Um bundle para todos os casos de uso | Implementações independentes, menor impacto em caso de erro |
| [data] | Modelos promovidos por referência, código implementado a partir do Git | Padrão completo de deploy code | Retreino mensal conduzido por pessoas; mantém a abstração do YAML existente |
| [data] | Substituir os robôs por uma ligação direta | Mover os robôs para Cloud Run | Mesmo pré-requisito de rede; elimina a fragilidade em vez de a mudar de sítio |

---

## Anexo A — Definition of Done para um caso de uso em produção

Um caso de uso está no standard quando todas as condições seguintes se verificam:

1. **Namespace isolado** — schema próprio em `ds_dev`, `ds_cq` e `ds_prod`; sem caminhos partilhados.
2. **Artefacto de modelo completo** — um único modelo registado que recebe os dados em bruto e devolve uma previsão, incluindo todas as transformações.
3. **Código versionado** — a entrega corre uma cópia implementada de um commit com tag, com uma versão fixa do toolkit.
4. **Configuração declarada** — todos os valores específicos de cada ambiente vêm da configuração ou das variáveis do bundle.
5. **Identidade de sistema** — as execuções agendadas correm com o service principal em job compute.
6. **Falha de forma visível** — verificações às entradas e saídas no fluxo de entrega, ligadas a alertas.
7. **Rastreável** — para qualquer entrega, é possível identificar o commit, a versão do toolkit, a versão do modelo e os dados de entrada.
8. **Implementado por pipeline** — ninguém implementa em produção à mão; as alterações só lá chegam através de uma tag de release.

Excluído deliberadamente, para já: retreino automático, monitorização de drift, serving em tempo real, AutoML.
