# Plano Técnico-Editorial: Dashboard de Cobertura Eleitoral em Tempo Real
### Integração com dados públicos da Justiça Eleitoral (TSE) — apuração nacional e estadual, com camada de IA

**Documento elaborado em:** 11/09/2026
**Autor:** Inteligência híbrida (jurista + engenheiro de software) a pedido de Ângelo Fernando Da
**Status:** Plano para validação — nenhuma linha de código de produção foi implementada nesta entrega

---

## 0. Aviso crítico de prazo — leia isto primeiro

O 1º turno das Eleições Gerais 2026 ocorre em **04/10/2026** e o eventual 2º turno em **25/10/2026**. A data deste documento é 11/09/2026 — ou seja, **faltam aproximadamente 23 dias corridos para o 1º turno**. Isso muda fundamentalmente a natureza deste plano: não é viável, neste prazo, desenhar e homologar do zero uma integração formal com a Secretaria de Tecnologia da Informação (STI) do TSE, participar dos simulados oficiais do ciclo (que já devem estar em estágio avançado ou concluídos) e construir uma stack de IA sofisticada com o mesmo rigor de um projeto de 6 a 12 meses.

Por isso, este plano é deliberadamente estruturado em **duas trilhas paralelas**, não sequenciais:

1. **Trilha Emergencial (Eleições 2026)** — um produto mínimo viável (MVP), tecnicamente honesto e jornalisticamente confiável, construído em ~3 semanas a partir de fontes públicas já disponíveis sem necessidade de credenciamento.
2. **Trilha Estruturante (produto duradouro)** — a versão robusta, com credenciamento formal junto ao TSE, apuração paralela via Boletim de Urna, e uma camada de IA plena — com horizonte nas **eleições municipais de 2028**, mas que pode e deve começar a ser desenhada já em paralelo à trilha emergencial, para não reiniciar do zero em 2028.

A recomendação central deste plano é: **não tentar construir a versão robusta a tempo do 1º turno**. O risco reputacional de um dashboard jornalístico que erra números na noite da eleição é desproporcional ao ganho de tentar fazer tudo de uma vez em três semanas.

---

## 1. Sumário executivo

Sim — **existe acesso público e gratuito aos dados de apuração eleitoral do TSE**, sem necessidade de chave de API, cadastro prévio ou convênio formal, através de arquivos JSON estáticos servidos publicamente pela própria infraestrutura de divulgação de resultados do Tribunal. Esse é, tecnicamente, o mesmo mecanismo que toda a grande imprensa nacional (G1, UOL, Folha, Estadão etc.) usa para alimentar seus próprios painéis de apuração ao vivo. Não se trata de uma API REST formalmente versionada com documentação OpenAPI/Swagger — é uma infraestrutura pública, estável e de uso corrente há vários ciclos eleitorais, cuja legalidade de uso jornalístico decorre diretamente da Lei de Acesso à Informação (Lei nº 12.527/2011).

Complementarmente, o TSE mantém um **Portal de Dados Abertos** (dadosabertos.tse.jus.br), baseado em CKAN, com datasets estruturados (candidatos, prestação de contas, eleitorado, resultados consolidados de pleitos anteriores) acessíveis via API RPC em JSON/CSV — útil para a camada histórica e de enriquecimento (não para a apuração ao vivo da noite da eleição).

Este plano propõe uma arquitetura de ingestão → normalização → distribuição em tempo real → camada de IA editorial (estritamente "grounded", nunca geradora de números) → produto de dashboard nacional e estadual (27 UFs), com uma trilha emergencial de 3 semanas para 2026 e uma trilha estruturante mirando 2028.

---

## 2. Verificação de fontes de dados públicas — resultado da pesquisa

### 2.1 Resultados em tempo real (a fonte principal para a noite da apuração)

**URL base:** `https://resultados.tse.jus.br/oficial/`

Durante a apuração, o TSE publica arquivos JSON estáticos, hierarquicamente organizados por eleição/pleito/UF/município/zona/seção, atualizados continuamente conforme os boletins de urna são recebidos e totalizados. Exemplos de padrão observado em ciclos anteriores (a estrutura exata de 2026 deve ser confirmada no dia, pois o código do pleito muda a cada eleição):

```
https://resultados.tse.jus.br/oficial/ele{ANO}/{CODIGO_PLEITO}/dados-simplificados/{uf}/{uf}-c{cargo}-e{eleicao}-r.json
https://resultados.tse.jus.br/oficial/ele{ANO}/{CODIGO_PLEITO}/dados/{UF}/{MUNICIPIO}/{ZONA}/{SECAO}/{hash}/{arquivo}
```

Existem três famílias de arquivo relevantes, conforme a documentação técnica do TSE ("Informações técnicas sobre a divulgação de resultados 2026") e a literatura de terceiros que consome esses dados há anos:

- **Arquivos de configuração** (`config`) — metadados do pleito, municípios, candidatos, partidos.
- **Arquivos de monitoramento** (`monitor-br`, `monitor-uf`) — indicadores agregados de progresso da apuração (percentual de seções totalizadas, votos brancos/nulos, comparecimento) em nível nacional e por UF.
- **Arquivos de resultado** (`dados-simplificados`, resultado por candidato) — votação nominal e por legenda, atualizada incrementalmente.

Esses arquivos **não exigem autenticação** e são servidos via CDN, o que os torna tecnicamente equivalentes a uma API pública read-only, mesmo sem um "portal de desenvolvedor" formal com chave de API.

**Fonte oficial primária (não fu acessível via fetch automatizado neste ambiente devido a bloqueio de bot — recomenda-se validação manual pela equipe):**
`https://www.tse.jus.br/eleicoes/informacoes-tecnicas-sobre-a-divulgacao-de-resultados`

### 2.2 DivulgaCandContas — o que é, e o que não fornece

`https://divulgacandcontas.tse.jus.br` é o sistema oficial de **divulgação de candidaturas e prestação de contas** — consulta de dados de registro de candidatura, bens declarados e contas de campanha. **Não é uma fonte de apuração ao vivo**: não publica resultado por urna, seção ou zona em tempo real, e não deve ser tratado como fallback do feed de resultados (correção em relação a uma versão anterior deste documento, que o descrevia incorretamente como fonte complementar de resultados — agradecemos a revisão que identificou o erro).

O fallback tecnicamente correto para a apuração ao vivo, caso o feed principal de `resultados.tse.jus.br` (Seção 2.1) fique instável, são os próprios **arquivos brutos do Boletim de Urna (.bu) e do RDV** (Seção 2.3) — chegam por seção/zona diretamente da urna, com schema e cadência compatíveis com apuração, sem depender do totalizador. A equipe de ingestão deve tratar 2.1 como fonte primária e 2.3 como fallback/auditoria; o DivulgaCandContas nunca deve substituir nenhum dos dois no pipeline de resultados.

### 2.3 Boletim de Urna (.bu) e Registro Digital do Voto (RDV) — a camada de auditoria

O TSE disponibiliza, para cada seção eleitoral, os arquivos brutos assinados digitalmente gerados pela urna eletrônica:

- **`.bu`** — Boletim de Urna em formato binário (ASN.1), com a contagem de votos daquela seção.
- **`.rdv`** — Registro Digital do Voto, o log individual (anonimizado) de cada voto computado.
- **`.logjez`** — log textual compactado da urna.
- **`.vscmr`** — arquivo de assinatura digital/verificação de integridade.

Esses arquivos são publicados **sem qualquer alteração**, exatamente como saem da urna, e permitem a qualquer interessado — inclusive um portal de notícias — realizar uma **apuração paralela e independente**, comparando o resultado oficial totalizado com a soma bruta dos boletins de urna. Isso é tecnicamente possível e há projetos open source de terceiros (ex.: `doccaz/urnas-br` no GitHub) que já fazem essa engenharia reversa de download e leitura.

> **Relevância editorial:** esta é a peça mais valiosa do ponto de vista de credibilidade jornalística. Um dashboard que apenas replica o número do totalizador oficial tem o mesmo valor de qualquer concorrente. Um dashboard que **verifica de forma independente** o totalizador contra os boletins de urna brutos — e sinaliza publicamente que a verificação bateu — constrói uma camada de confiança que nenhum concorrente sem essa engenharia tem.

### 2.4 Portal de Dados Abertos do TSE (dadosabertos.tse.jus.br)

Baseado em CKAN, expõe uma API em estilo RPC retornando JSON, com documentação própria (aba "Documentos" de cada dataset). Contém, entre outros, os grupos: **Candidatos**, **Eleitorado**, **Prestação de Contas**, **Pesquisas Eleitorais**, **Resultados** (históricos/consolidados de pleitos já encerrados, não a apuração ao vivo). Uso previsto neste plano: enriquecimento de contexto (histórico de votação do candidato/partido no município, dados socioeconômicos cruzados, base para os modelos de projeção estatística).

### 2.5 Base legal de uso

O acesso a esses dados é amparado pela **Lei de Acesso à Informação (Lei nº 12.527/2011)** — são dados públicos, sem necessidade de autorização individualizada para consulta. Não há, a partir da pesquisa realizada, indício de que o TSE exija um "convênio" formal para o simples consumo do feed público de resultados — o convênio/credenciamento (ver 2.6) é relevante para acesso privilegiado a infraestrutura dedicada e suporte técnico, não para o consumo básico do dado público.

> **Correção em relação a uma versão anterior:** a LAI garante *acesso* e *transparência* — ela não é, isoladamente, uma licença irrestrita para *consumo, republicação e uso comercial* de qualquer endpoint do TSE. Este plano inclui widgets embutíveis, geração de conteúdo por IA e, na trilha estruturante, licenciamento white-label — usos que vão além da simples consulta e podem se sujeitar a termos de uso próprios do TSE (requisitos de atribuição, limites de redistribuição, regras específicas de divulgação de resultado). **Antes de depender do feed em produção, a equipe deve localizar e validar os termos de uso/reuso publicados pelo TSE para cada fonte (2.1–2.4), não apenas invocar a LAI de forma genérica.** Essa validação é pré-requisito do item 1 do roadmap (Seção 10).

### 2.6 Centro de Divulgação das Eleições (CDE) e simulados técnicos

O TSE mantém, para cada ciclo eleitoral, o **Centro de Divulgação das Eleições (CDE)**, estrutura física em Brasília voltada a veículos de imprensa credenciados, e conduz **audiências técnicas e simulados** junto a empresas de tecnologia e veículos de comunicação, nos quais é recomendada a participação de um técnico de TI de cada veículo para testar a integração com os dados brutos antes do pleito. Para o ciclo de 2026, essas atividades já estavam em curso em julho/agosto de 2026 — ou seja, **é provável que a janela para credenciamento formal e participação nos simulados oficiais deste ciclo já esteja fechada ou em estágio final**. Isso reforça a decisão estratégica deste plano de não depender desse canal para a trilha emergencial, e reservá-lo como ação prioritária da trilha estruturante, com início imediato do relacionamento institucional visando o próximo ciclo.

### 2.7 Conclusão da verificação

| Pergunta | Resposta |
|---|---|
| Existe API pública para apuração em tempo real? | **Sim**, na prática: feed JSON público, sem autenticação, via `resultados.tse.jus.br/oficial/...`, com o Boletim de Urna bruto (Seção 2.3) como fallback — não o DivulgaCandContas, que é apenas candidaturas/contas. |
| É uma API formalmente documentada e versionada? | Parcialmente. Há documentação técnica oficial (página "Informações técnicas sobre a divulgação de resultados"), mas não no padrão de um developer portal com OpenAPI/chaves de API. |
| É preciso convênio para o uso básico? | Não, para o consumo do dado público. Sim, para acesso privilegiado a infraestrutura/suporte dedicado via CDE. |
| Há dados históricos estruturados? | Sim, via Portal de Dados Abertos (CKAN), formato JSON/CSV. |
| É possível apuração paralela independente? | Sim, via Boletim de Urna (.bu) e RDV brutos, publicados sem alteração. |
| Uso jornalístico é legalmente amparado? | O acesso, sim (LAI). Consumo, republicação e uso comercial exigem validar os termos de uso próprios do TSE — a LAI sozinha não os substitui (ver 2.5). |

**Fontes consultadas:**
- [Informações técnicas sobre a divulgação de resultados 2026 — TSE](https://www.tse.jus.br/eleicoes/informacoes-tecnicas-sobre-a-divulgacao-de-resultados)
- [Resultados das Eleições 2026 poderão ser acompanhados em tempo real — TSE](https://www.tse.jus.br/comunicacao/noticias/2026/Agosto/resultados-das-eleicoes-2026-poderao-ser-acompanhados-em-tempo-real)
- [Portal de Dados Abertos do TSE — grupo Resultados](https://dadosabertos.tse.jus.br/group/about/resultados)
- [Portal de Dados Abertos do TSE — datasets](https://dadosabertos.tse.jus.br/dataset/)
- [TSE fará simulados com veículos de imprensa — TSE](https://www.tse.jus.br/comunicacao/noticias/2026/Julho/tse-fara-simulados-com-veiculos-de-imprensa-para-testar-divulgacao-dos-resultados-das-eleicoes-2026)
- [CDE 2026 — TSE](https://www.tse.jus.br/eleicoes/cde-2026)
- [GitHub — doccaz/urnas-br (análise de boletins de urna)](https://github.com/doccaz/urnas-br)
- [Dados abertos, webservice e APIs — TRE-SE](https://www.tre-se.jus.br/transparencia-e-prestacao-de-contas/dados-abertos-e-webservice-e-apis)
- [Faltam 26 dias: votação começa e termina no mesmo horário em todo o país — TSE](https://www.tse.jus.br/comunicacao/noticias/2026/Setembro/faltam-26-dias-votacao-comeca-e-termina-no-mesmo-horario-em-todo-o-pais)
- [Eleições 2026: confira as principais datas do calendário eleitoral — TSE](https://www.tse.jus.br/comunicacao/noticias/2026/Marco/eleicoes-2026-confira-as-principais-datas-do-calendario-eleitoral)

**Alerta metodológico:** o ambiente de execução deste agente foi bloqueado (HTTP 403) ao tentar acessar diretamente `tse.jus.br` e `dadosabertos.tse.jus.br` (provável proteção anti-bot do próprio tribunal). As informações acima foram reconstruídas a partir de resumos de busca e de fontes secundárias que citam essas páginas. **Antes de iniciar a implementação, a equipe técnica deve acessar manualmente, em navegador comum, a página "Informações técnicas sobre a divulgação de resultados 2026" e baixar a documentação oficial completa (formatos exatos de JSON, cadência de atualização, código do pleito de 2026)** — esse é o primeiro passo prático do roadmap, na Seção 7.

---

## 3. Enquadramento jurídico-editorial (perfil jurista)

### 3.1 Regime de divulgação de resultados

Segundo apuração de imprensa, a **Resolução TSE nº 23.751/2026** estabeleceu, para este ciclo, votação simultânea em todo o território nacional das 8h às 17h no horário de Brasília (inclusive nas localidades de fuso do Acre, que iniciam às 6h local para coincidir com as 8h de Brasília), eliminando a antiga necessidade de aguardar o fechamento escalonado por fuso horário. A divulgação de resultados pela Justiça Eleitoral pode começar **a partir das 17h, horário de Brasília**, após o encerramento da votação em todo o país.

> **Recomendação de compliance:** confirmar o número e o teor exato dessa resolução junto ao Diário da Justiça Eletrônico do TSE antes de operacionalizar qualquer regra de embargo no sistema — a informação acima foi obtida de fontes de imprensa, não do texto normativo primário, e nenhuma decisão editorial vinculante deve ser tomada sem essa confirmação.

**Consequência de arquitetura:** o dashboard deve ter uma trava de horário (feature flag por horário de Brasília) que impede a exibição de **qualquer** número de apuração antes das 17h no dia da eleição, mesmo que o pipeline de ingestão já tenha recebido dados de teste ou de simulados. Essa trava deve ser redundante (backend + frontend) para evitar vazamento por cache ou erro humano.

### 3.2 Pesquisas de boca de urna e projeções

Pesquisas de boca de urna (exit polls) têm regime de divulgação distinto e mais restritivo do que a apuração oficial — em regra, só podem ser divulgadas após o encerramento da votação. Como este plano **não prevê a contratação de pesquisa de boca de urna própria**, e sim o consumo de dados oficiais de apuração, esse risco específico não se aplica diretamente — mas deve ser reforçado junto à redação: qualquer "projeção" gerada pela camada de IA (Seção 5.3) deve ser claramente rotulada como projeção estatística do próprio portal, nunca confundida com pesquisa de boca de urna nem com resultado oficial.

### 3.3 Responsabilidade editorial sobre conteúdo gerado por IA

Toda peça textual publicada no portal com apoio de IA (resumos, alertas, "viradas" de município) deve passar por **revisão humana obrigatória antes da publicação**. Do ponto de vista de responsabilidade civil (arts. 927 e 931 do Código Civil, e o regime de responsabilidade da imprensa), o erro de um sistema automatizado que resulte em difamação, informação falsa sobre resultado eleitoral ou indução de erro ao eleitor gera responsabilidade do veículo de comunicação, não do fornecedor de IA. Este plano trata esse ponto como não negociável na Seção 5.7 (guardrails).

### 3.4 LGPD

Os dados de candidatos (nome, número, partido, votos recebidos) decorrem do registro de candidatura, ato público por definição, e sua natureza pública os distancia do regime de dados de eleitores individuais. **Isso não significa, porém, que fiquem fora da LGPD.** A publicidade do dado não afasta, por si só, a incidência da lei: o art. 7º, §§ 3º e 4º, ainda exige que o tratamento observe finalidade, boa-fé e interesse público, e que respeite os princípios do art. 6º — salvo quando o tratamento se enquadrar exclusivamente na exceção jornalística (art. 4º, II, "b"). Como este plano vai além da simples exibição factual do resultado — inclui geração de conteúdo por IA, projeções estatísticas (trilha estruturante) e eventual licenciamento white-label — a equipe deve registrar, para os dados de candidatos, a base legal aplicável a cada finalidade, o propósito específico de cada uso e os limites de retenção/compartilhamento, em vez de tratá-los genericamente como "fora da LGPD". Recomenda-se validar esse desenho com a assessoria jurídica antes da trilha estruturante.

Já os dados de **usuários do próprio dashboard** (cadastro para alertas push, newsletter, analytics de navegação) estão sujeitos integralmente à LGPD, sem as ressalvas acima — recomenda-se política de privacidade específica para essa funcionalidade e minimização de coleta (não é necessário, por exemplo, capturar CPF ou título de eleitor do usuário para nenhuma funcionalidade prevista neste plano).

---

## 4. Arquitetura técnica proposta

### 4.1 Visão em camadas

```
[TSE - resultados.tse.jus.br/oficial/...]   [TSE - dadosabertos.tse.jus.br (CKAN)]
                |  polling resiliente                |  ingestão em lote (pré/pós-pleito)
                v                                     v
        +---------------------------------------------------+
        |     CAMADA DE INGESTÃO (serviço Python/FastAPI)    |
        |  - poller com backoff exponencial                  |
        |  - normalização do JSON do TSE -> modelo interno    |
        |  - deduplicação e versionamento por timestamp       |
        +---------------------------------------------------+
                                |
                                v
        +---------------------------------------------------+
        |  BANCO DE DADOS (PostgreSQL + TimescaleDB)          |
        |  - série temporal de apuração por município/UF/BR   |
        |  - tabela de candidatos/partidos (via dados abertos) |
        |  - tabela de checagens de auditoria (BU vs totalizador) |
        +---------------------------------------------------+
                                |
                +---------------+----------------+
                v                                 v
    +----------------------+       +---------------------------+
    | CAMADA DE IA          |       | CAMADA DE DISTRIBUIÇÃO     |
    | (grounded, sem gerar   |       | Redis Pub/Sub + WebSocket/ |
    |  números; ver Seção 5) |       | SSE + CDN edge cache        |
    +----------------------+       +---------------------------+
                \                                 /
                 \                               /
                  v                             v
          +-------------------------------------------+
          |     FRONTEND (Next.js/React)                |
          |  Dashboard nacional + 27 páginas de UF       |
          |  Widgets embutíveis para o portal             |
          +-------------------------------------------+
```

### 4.2 Camada de ingestão

- Serviço dedicado (não acoplar ao backend do Juris Code — projeto e domínio de dados completamente distintos; recomenda-se repositório e deploy independentes).
- Polling com **respeito à infraestrutura do TSE**: intervalo mínimo de 15–30 segundos por recurso (nunca "martelar" o endpoint em loop apertado — isso é tanto uma boa prática técnica quanto uma questão de responsabilidade institucional, já que a infraestrutura do TSE é compartilhada por toda a imprensa nacional na noite da eleição).
- Backoff exponencial e circuit breaker: se o TSE ficar instável (histórico mostra picos de lentidão em noites de eleição), o serviço deve degradar graciosamente (exibir "última atualização há X minutos" em vez de quebrar).
- Cache local de cada arquivo JSON bruto (auditoria própria: guardar o payload originalmente recebido do TSE, com timestamp, para eventual contestação futura de "seu número estava errado").

### 4.3 Modelo de dados

Entidades centrais: `Pleito` (ano, turno, tipo), `UnidadeEleitoral` (BR/UF/Município/Zona/Seção), `Candidato`, `ResultadoParcial` (série temporal: candidato × unidade × timestamp × votos × % apurado), `ChecagemAuditoria` (comparação totalizador × soma de boletins de urna brutos).

### 4.4 Camada de distribuição em tempo real

Dado que o público do dashboard (leitores do portal) pode gerar picos de tráfego **ordens de magnitude maiores** do que o volume de atualizações vindas do TSE, a arquitetura deve desacoplar completamente ingestão de distribuição:

- Redis Pub/Sub (ou Kafka, se o volume justificar) recebe cada atualização normalizada e publica para os clientes conectados via WebSocket/SSE.
- CDN com cache de borda (Cloudflare/Vercel Edge) para as visões agregadas (nacional, por UF), invalidado a cada novo dado — a maioria dos leitores não precisa de uma conexão WebSocket individual, uma página que revalida a cada 15–30s via cache de borda já atende a experiência esperada de um dashboard de apuração.
- Reservar WebSocket/push para funcionalidades de maior valor (alerta de "virada" na corrida que o usuário está acompanhando).

### 4.5 Resiliência de tráfego do lado do leitor

Esse é o risco técnico mais frequentemente subestimado em projetos desse tipo: **a noite da eleição não é apenas um pico de dados vindos do TSE — é um pico de tráfego de leitores do próprio portal**, tipicamente 10x–50x o tráfego normal, concentrado nas duas horas após o fechamento das urnas. Recomenda-se:

- Renderização estática incremental (ISR) ou cache de borda agressivo nas páginas de maior tráfego (home do dashboard nacional).
- Teste de carga simulado antes do dia da eleição (ver Seção 7.1, Semana 3), usando os próprios dados históricos de 2022/2024 como dataset de replay.
- Plano de contingência de infraestrutura (autoscaling configurado e testado, não apenas "disponível na teoria" no provedor de nuvem).

---

## 5. Camada de IA — o diferencial editorial

O objetivo desta camada **não é gerar números de votação** — isso seria um risco jornalístico inaceitável (alucinação de LLM aplicada a resultado eleitoral é o pior cenário possível de erro editorial). O objetivo é **acelerar e enriquecer o trabalho humano da redação** em cima de números que vêm exclusivamente do pipeline determinístico da Seção 4.

### 5.1 Princípio arquitetural inegociável: separação entre "camada de fato" e "camada de narrativa"

Todo número exibido no dashboard (votos, percentual apurado, posição do candidato) **deve vir exclusivamente do pipeline de ingestão determinístico**, nunca de uma resposta de LLM. A camada de IA só tem acesso de leitura a esses números já computados — ela nunca é a fonte da verdade sobre um número. Tecnicamente, isso significa: o LLM é usado em modo *retrieval-augmented / grounded generation*, com os números injetados no prompt como fatos imutáveis, e qualquer resposta que contradiga os dados de entrada é descartada por um validador automático antes de chegar à redação.

### 5.2 Funcionalidades propostas, em ordem de prioridade para a trilha emergencial

1. **Alertas de "corrida apertada" e "viradas"** — monitoramento contínuo da série temporal por unidade eleitoral, disparando alerta para a redação quando a margem entre 1º e 2º colocado cai abaixo de um limiar (ex. 3 pontos percentuais) com apuração acima de determinado percentual, ou quando há troca de líder. Isso é determinístico (regra sobre os dados), não geração de texto — implementável em poucos dias.
2. **Resumos automáticos por região/UF para editores** — um parágrafo gerado por LLM, grounded estritamente nos números do banco, resumindo o estado da apuração em uma UF ("com X% das seções apuradas, Fulano lidera com Y%, seguido por Beltrano com Z%..."), sempre marcado como rascunho para revisão humana antes de qualquer publicação.
3. **Checagem de auditoria (Boletim de Urna vs. totalizador)** — comparação seção a seção, não amostra contra agregado: para cada seção incluída na amostra (não é necessário 100% das seções do país em tempo real), comparar o resultado **daquela mesma seção** no totalizador oficial contra o boletim de urna bruto **da mesma seção, no mesmo instante/versão de dado**. *Correção em relação a uma versão anterior deste documento:* comparar a soma de uma amostra por UF contra o agregado nacional/estadual do totalizador é metodologicamente inválido, porque os dois lados têm cobertura e timestamp diferentes até que 100% das seções estejam incluídas — isso geraria divergências espúrias e comprometeria a credibilidade da própria checagem. Divergência real = mesma seção, mesmo corte temporal, números diferentes; qualquer leitura no nível agregado deve ser apresentada apenas como medição de cobertura ("X% da amostra já confere com o totalizador"), nunca como alerta de divergência. Tecnicamente mais complexo por exigir parsing do formato ASN.1 do `.bu`; recomenda-se escopo reduzido na trilha emergencial (ex. capitais e uma amostra de municípios grandes) e expansão na trilha estruturante.
4. **Assistente de consulta em linguagem natural para a redação** — interface tipo chat interno (não pública) onde um jornalista pergunta "como está a disputa pelo governo de SP" e recebe uma resposta grounded nos dados correntes. Importante: os dados oficiais de apuração **não trazem perfil demográfico do eleitor** (idade, gênero, renda) — qualquer cruzamento desse tipo exigiria fonte externa, como dados socioeconômicos do IBGE. *Correção em relação a uma versão anterior:* o IBGE não publica esses indicadores por seção eleitoral — sua unidade geográfica é o **setor censitário**, que não coincide com a seção eleitoral (uma seção pode cruzar vários setores, e a composição de ambos muda entre pleitos). Um cruzamento direto seção↔setor produziria estimativas sem base geográfica definida. Por isso, esse cruzamento exige uma metodologia própria de correspondência espacial (ano de referência do censo, ponderação por área/população, tratamento explícito de incerteza) a ser desenhada na trilha estruturante e sempre rotulada como estimativa, nunca como dado oficial; até que essa metodologia esteja validada, qualquer cruzamento sociodemográfico deve se limitar ao **nível municipal**, onde a correspondência com os dados do IBGE é direta.
5. **Geração de áudio (text-to-speech) dos resumos** — acessibilidade, baixo custo de implementação dado que a Seção 5.2 já produz o texto-base.

### 5.3 Projeção estatística (swing modeling) — trilha estruturante

Modelo comparando o padrão de apuração corrente com o comportamento histórico do mesmo município/zona em pleitos anteriores (dados do Portal de Dados Abertos, Seção 2.4) para estimar tendência antes dos 100% de apuração. **Alto risco editorial se malfeito** (o caso mais citado internacionalmente de erro de projeção eleitoral é exatamente esse tipo de modelo saindo errado publicamente) — recomenda-se não incluir na trilha emergencial de 2026, e desenvolvê-lo com rigor estatístico e validação retroativa contra pelo menos três pleitos anteriores antes de qualquer publicação ao vivo.

### 5.4 Guardrails obrigatórios (não negociáveis)

- LLM nunca tem permissão de gerar um número de votos ou percentual — todo número em texto gerado é interpolado programaticamente a partir do banco, não "lembrado" ou "calculado" pelo modelo.
- Todo conteúdo gerado por IA carrega metadado de proveniência (qual pipeline, qual timestamp de dado, qual modelo) para auditoria futura.
- Publicação 100% humana no primeiro pleito de operação do sistema (2026) — a IA produz rascunho, nunca publica diretamente. Automação de publicação (com revisão humana amostral, não de 100% das peças) é uma evolução de trilha estruturante, após o sistema provar confiabilidade em pelo menos um ciclo eleitoral completo.

---

## 6. Produto / experiência do usuário

- **Visão nacional**: mapa do Brasil com resultado agregado, cargos majoritários (Presidente) e por UF (Governador, Senador), painel de percentual de seções apuradas, comparecimento, votos brancos/nulos.
- **Visão estadual (27 páginas de UF)**: resultado detalhado por cargo, ranking de candidatos, evolução temporal (gráfico de linha do avanço da apuração).
- **Drill-down municipal**: sob demanda, não como página estática pré-gerada para os ~5.570 municípios (custo/benefício não justifica isso na trilha emergencial — implementar como consulta dinâmica).
- **Widgets embutíveis**: componente isolado (iframe ou web component) que outras editorias do portal podem inserir em matérias durante a cobertura, sem depender da página principal do dashboard.
- **Modo 2º turno**: reconfiguração do mesmo produto para 25/10/2026, caso haja segundo turno para Presidente e/ou governos estaduais.
- **Mobile-first**: a maior parte do tráfego de cobertura eleitoral no Brasil é mobile; qualquer decisão de design deve partir da tela pequena, não da tela grande.

---

## 7. Cronograma proposto

### 7.1 Trilha Emergencial — Eleições 2026 (janela de ~3 semanas até 04/10)

| Semana | Entregas |
|---|---|
| **Semana 1** (imediata) | Validação manual da documentação técnica oficial do TSE (Seção 2.7, alerta metodológico); serviço de ingestão MVP consumindo o feed público; modelo de dados; dashboard nacional básico (sem IA ainda) |
| **Semana 2** | 27 páginas de UF; camada de IA restrita (itens 5.2.1 e 5.2.2 apenas — alertas de corrida apertada e resumos grounded); trava de horário de divulgação (Seção 3.1); teste de carga de leitura |
| **Semana 3** | Ensaio geral ponta a ponta usando replay de dados de 2022/2024 como dry-run; hardening de resiliência (Seção 4.5); plano de contingência escrito e testado para indisponibilidade do TSE; treinamento da redação no fluxo de revisão humana de conteúdo de IA |
| **04/10 (dia da eleição)** | Operação ao vivo, war room técnico + editorial, monitoramento humano contínuo, sem automação de publicação |
| **25/10 (se houver 2º turno)** | Reconfiguração do mesmo sistema para o pleito de 2º turno |

**Escopo explicitamente fora da trilha emergencial:** checagem de auditoria via Boletim de Urna bruto em escala nacional (item 5.2.3, apenas amostra reduzida se houver tempo), projeção estatística (5.3), assistente de linguagem natural completo (5.2.4), credenciamento formal junto ao CDE/STI do TSE.

### 7.2 Trilha Estruturante — rumo a 2028 (municípios) e ciclos seguintes

| Fase | Objetivo |
|---|---|
| **Fase A** (a partir de nov/2026, pós-eleição) | Relacionamento institucional formal com o TSE — credenciamento junto ao CDE, participação nos próximos simulados técnicos oficiais, com bastante antecedência (não repetir o aperto de prazo de 2026) |
| **Fase B** | Implementação completa da apuração paralela via Boletim de Urna bruto em escala nacional (parsing ASN.1, verificação criptográfica de integridade) — a peça que diferencia o produto de qualquer concorrente |
| **Fase C** | Camada de IA expandida: modelos de projeção validados retroativamente, correlação com dados socioeconômicos do IBGE, ferramentas avançadas de apoio à redação (geração de infográficos automáticos, comparação histórica) |
| **Fase D** | Produto white-label — sindicalização/licenciamento do dashboard para outros veículos de imprensa como modelo de receita adicional |

---

## 8. Riscos e mitigação

| Risco | Severidade | Mitigação |
|---|---|---|
| TSE fica lento/instável na noite da eleição (padrão histórico recorrente) | Alta | Cache agressivo, backoff, circuit breaker, comunicação transparente ao leitor ("última atualização há X min") |
| IA gera/alucina um número de votos | Crítica | Guardrail arquitetural (Seção 5.4): números nunca vêm do LLM, sempre do pipeline determinístico |
| Pico de tráfego de leitores derruba o próprio dashboard | Alta | CDN/edge cache, arquitetura stateless, teste de carga antes do dia da eleição |
| Prazo de 3 semanas é insuficiente para o escopo completo | Alta | Escopo mínimo rigorosamente definido (Seção 7.1); tudo que não é essencial vai para a trilha estruturante |
| Publicação de projeção/estimativa mal calibrada e publicamente incorreta | Alta | Projeção estatística (5.3) fora do escopo emergencial; toda peça de IA com revisão humana obrigatória |
| Divulgação antes do horário legal permitido (17h de Brasília) | Crítica (jurídica) | Trava de horário redundante (backend + frontend); confirmação do texto normativo exato antes da operação (Seção 3.1) |
| Dependência de scraping de estrutura de URL não oficialmente versionada | Média | Monitoramento ativo de mudança de formato no dia; fallback manual de atualização; não depender de um único endpoint sem verificação de schema |

---

## 9. Stack técnica recomendada (pragmática, compatível com o prazo)

- **Ingestão/backend**: Python 3.11 + FastAPI (consistência com o time que já opera o Juris Code, reduz curva de aprendizado)
- **Banco relacional**: PostgreSQL 16 com extensão TimescaleDB para a série temporal de apuração
- **Cache/pub-sub**: Redis
- **LLM**: OpenAI GPT-4o (ou modelo equivalente), sempre em modo grounded/RAG restrito, nunca fine-tuned para "prever" resultados
- **Frontend**: Next.js + TypeScript + Tailwind (mesma stack do Frontend V2 do Juris Code, reaproveitando conhecimento do time)
- **Infraestrutura**: deploy em provedor com CDN/edge nativo (Vercel ou Cloudflare) para a camada de leitura pública; ingestão em servidor dedicado com autoscaling
- **Observabilidade**: dashboard interno de saúde do pipeline (latência do TSE, taxa de erro, última atualização por UF) — essencial para o war room do dia 04/10

---

## 10. Próximos passos imediatos (esta semana)

1. Acessar manualmente (navegador comum, fora deste ambiente com bloqueio de bot) a página oficial `tse.jus.br/eleicoes/informacoes-tecnicas-sobre-a-divulgacao-de-resultados` e baixar a documentação técnica completa, incluindo o código do pleito 2026 e a estrutura exata dos arquivos JSON.
2. Validar com a assessoria jurídica do portal o texto exato da Resolução TSE citada na Seção 3.1 antes de codificar qualquer regra de embargo.
3. Definir squad mínimo para a trilha emergencial (sugestão: 1 engenheiro backend, 1 engenheiro frontend, 1 editor(a) responsável pela revisão de conteúdo de IA) — dado o prazo, não há espaço para um time grande coordenar-se.
4. Iniciar imediatamente o serviço de ingestão contra o feed público, mesmo antes de qualquer decisão final de produto — é o item de maior risco de prazo e o que mais precisa de tempo de validação contra dados reais.
5. Registrar interesse institucional junto ao TSE (CDE) para o próximo ciclo, mesmo que tarde demais para 2026 — inicia a Fase A da trilha estruturante sem custo de oportunidade.

---

## 11. Conclusão

A pergunta central do pedido — "existe uma API pública para isso" — tem resposta afirmativa, com a ressalva de que se trata de uma infraestrutura pública de fato (amplamente usada pela imprensa há anos, amparada pela LAI), não de um produto de API formalmente documentado e versionado no sentido de um developer portal comercial. Isso é suficiente e é exatamente o que todo grande veículo de imprensa brasileiro usa hoje.

O maior risco deste projeto não é técnico — é de **prazo e de disciplina de escopo**. Com 23 dias até o primeiro turno, a decisão mais importante que a equipe pode tomar agora é resistir à tentação de construir a versão completa (projeção estatística, apuração paralela nacional, assistente de IA completo) e entregar, em vez disso, um MVP rigorosamente escopado, tecnicamente sólido e editorialmente confiável — reservando a ambição maior para a trilha estruturante com horizonte em 2028.
