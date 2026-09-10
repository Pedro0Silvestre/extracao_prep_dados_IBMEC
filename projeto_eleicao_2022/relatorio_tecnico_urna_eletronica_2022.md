# Relatório Técnico

## Investigação Econométrica do Impacto do Modelo da Urna Eletrônica no 2º Turno das Eleições Presidenciais de 2022

---

# 1. Extração de Dados e Peculiaridades Estruturais

A pesquisa utilizou como base primária os microdados oficiais disponibilizados pelo Tribunal Superior Eleitoral (TSE) referentes ao 2º Turno da Eleição Geral Federal de 2022.

Para viabilizar a análise estatística, foram combinadas duas fontes distintas de dados com alta granularidade (nível de seção eleitoral):

- **Boletins de Urna (BUs):** Arquivos consolidados de votação por seção eleitoral em todas as 27 Unidades da Federação.
- **Perfil do Eleitorado por Seção Eleitoral:** Cadastros demográficos agregados pelo TSE para cada seção física do país.

## Peculiaridades e Desafios de Engenharia de Dados

### Volume e Consumo de Memória

Os arquivos brutos de perfil e de boletins de urna ultrapassam dezenas de gigabytes com mais de 30 colunas por tabela.

Para viabilizar a execução em ambiente local sem estouro de memória (RAM), foi aplicado carregamento estrito via parâmetro `usecols`, descartando colunas administrativas e metadados redundantes.

### Estrutura "Longa" vs. "Larga" no Perfil do Eleitor

Diferente de uma tabela tabulada tradicional, a base demográfica do TSE é empilhada: as categorias de escolaridade, gênero e idade constam como registros textuais em linhas (`DS_GRAU_ESCOLARIDADE`, `DS_FAIXA_ETARIA`, `DS_GENERO`) acompanhadas de um contador escalar (`QT_ELEITORES_PERFIL`).

### Identificação do Modelo de Hardware

O modelo da urna não vinha explicitamente rotulado como "novo" ou "antigo" no BU padrão, exigindo mapeamento determinístico a partir do campo `NR_URNA_EFETIVADA` / especificação técnica fornecida pelo tribunal.

Dessa forma, separou-se o hardware de última geração (**UE2020**) dos modelos legados fabricados em ciclos anteriores:

- UE2009
- UE2010
- UE2011
- UE2013
- UE2015

---

# 2. Integração de Dados e Construção do Dataset Analítico

O processamento foi dividido em etapas de pré-agregação para homogeneizar a chave relacional primária composta pela quádrupla:

```text
['SG_UF', 'CD_MUNICIPIO', 'NR_ZONA', 'NR_SECAO']
```

Fluxo de integração:

```text
[Microdados BU TSE] ──> Agregação Votos Válidos ──┐
                                                  │
                                                  ├──> Merge Exato (Inner Join)
                                                  │
[Perfil TSE (Longo)] ──> Pivot & Proporções ──────┘
                                                  │
                                                  ▼
                                   df_analise (471.691 seções)
```

## O Pré-Processamento Demográfico

A partir da base longa de eleitores, foram criadas contagens condicionais via `np.where` e calculadas as seguintes variáveis sociodemográficas para cada seção:

- **PROP_SUPERIOR:** Proporção de eleitores com Ensino Superior (completo ou incompleto).
- **PROP_BAIXA_ESC:** Proporção de eleitores analfabetos ou que declararam apenas ler e escrever.
- **PROP_FEMININO:** Proporção de eleitoras do gênero feminino.
- **PROP_JOVENS:** Proporção de eleitores na faixa etária de até 24 anos.
- **PROP_IDOSOS:** Proporção de eleitores com 60 anos ou mais.

## O DataFrame Analítico Final (`df_analise`)

A base final consolidou **471.691 seções eleitorais válidas** em todo o território nacional.

Isso corresponde a uma taxa de preservação e pareamento de **95,0%** em relação ao cadastro eleitoral bruto, descartando apenas:

- Seções agregadas;
- Seções no exterior;
- Voto em trânsito;
- Seções sem comparecimento.

O dataset final contém os seguintes campos estruturais:

### Identificadores Geográficos

- `SG_UF`
- `CD_MUNICIPIO`
- `NM_MUNICIPIO`
- `NR_ZONA`
- `NR_SECAO`
- `REGIAO`
- `TIPO_LOCALIDADE` (Capital vs. Interior)

### Métricas Eleitorais

- `QT_APTOS`
- `QT_COMPARECIMENTO`
- `QT_ABSTENCOES`
- `VOTOS_VALIDOS`
- `LULA`
- `JAIR BOLSONARO`

### Variáveis Dependentes

**PROP_LULA**

$$
PROP\_LULA = \frac{Lula}{Votos\ Válidos}
$$

**PROP_BOLSONARO**

$$
PROP\_BOLSONARO = \frac{Bolsonaro}{Votos\ Válidos}
$$

### Variável de Tratamento

**IS_UE2020**

- `1` para o modelo UE2020;
- `0` para os modelos legados.

### Variáveis de Controle

- `PROP_SUPERIOR`
- `PROP_BAIXA_ESC`
- `PROP_FEMININO`
- `PROP_JOVENS`
- `PROP_IDOSOS`

Além da variável:

$$
PROP\_ABSTENCAO = \frac{Abstenções}{Aptos}
$$

Para fins de integridade de tipos e alta performance de I/O, o dataset foi persistido em formato binário colunar **Parquet** com compressão **Snappy**.

---

# 3. Metodologia Estatística e Resultados Passo a Passo

A análise foi estruturada sob o paradigma de inferência causal e controle de viés de seleção, executada em camadas sucessivas de controle econométrico.

## Etapa 1: Diagnóstico Bivariado Ingênuo (Baseline OLS)

### Objetivo

Reproduzir o argumento da divergência bruta entre os dois modelos de equipamento em escala nacional, sem quaisquer controles geográficos ou demográficos.

### Equação

$$
PROP\_LULA_i = \alpha + \beta_1 \cdot IS\_UE2020_i + \varepsilon_i
$$

### Resultado

- **Intercepto ($\alpha$):** `0,5337`
  - Lula obteve **53,37%** nas urnas legadas.

- **Coeficiente $\beta_1$ (`IS_UE2020`):** `-0,0419`
  - `p < 0,001`
  - `z = -4,214`

### Diagnóstico

Uma leitura ingênua sugere que a urna UE2020 esteve associada a uma desvantagem de **4,19 pontos percentuais** para o candidato Lula.

Essa especificação, no entanto, sofre de **severo viés de variável omitida**.

---

## Etapa 2: Regressão Multivariada com Controles Sociodemográficos

### Objetivo

Testar a estabilidade do coeficiente ao controlar o perfil do eleitorado de cada seção:

- Escolaridade;
- Idade;
- Gênero;
- Abstenção.

### Equação

$$
PROP\_LULA_i =
\alpha +
\beta_1 \cdot IS\_UE2020_i +
\mathbf{X}'_i \boldsymbol{\gamma} +
\varepsilon_i
$$

### Resultado

- **Coeficiente $\beta_1$ (`IS_UE2020`):** `+0,0227`
- `p = 0,013`
- `z = 2,476`

### Diagnóstico — Paradoxo de Simpson

A simples inclusão do perfil demográfico inverteu o sinal do coeficiente:

- De **-4,19 p.p.**
- Para **+2,27 p.p.** a favor de Lula.

Isso prova que as urnas legadas apresentavam maior votação em Lula unicamente porque estavam alocadas em contingentes com menor escolaridade.

A variável `PROP_BAIXA_ESC` apresentou:

- $\beta = +1,036$
- `z = 36,5`

---

## Etapa 3: Modelo com Efeitos Fixos de Município (Within-Transformation)

### Objetivo

Isolar completamente a variação entre cidades e comparar urnas novas vs. legadas exclusivamente dentro do mesmo município, com erros-padrão clusterizados por município.

### Equação

$$
PROP\_LULA_{im} =
\beta_1 \cdot IS\_UE2020_{im} +
\mathbf{X}'_{im}\boldsymbol{\gamma} +
\mu_m +
\varepsilon_{im}
$$

### Resultado

- **Coeficiente $\beta_1$ (`IS_UE2020`):** `+0,0006`
- `p = 0,846`
- `z = 0,194`

### Intervalo de Confiança de 95%

$$
[-0,005,\ +0,006]
$$

Ou seja:

- De **-0,5 p.p.**
- Até **+0,6 p.p.**

### Diagnóstico

Com a introdução do efeito fixo municipal, o efeito estimado da urna colapsa para zero:

- **0,06 p.p.**
- Imperceptível na prática.

Além disso, perde qualquer significância estatística:

- `p = 0,846`

---

## Etapa 4: Teste de Robustez em Unidades Mistas (Zonas Eleitorais)

### Objetivo

Testar seções restritas às **460 Zonas Eleitorais** que operaram com distribuição simultânea de ambos os tipos de urnas.

A amostra incluiu:

- **102.700 seções**

### Execução 1 — Média Não Paramétrica Bruta

$$
\bar{Y}_{UE2020} - \bar{Y}_{Legadas} = -1,75\ p.p.
$$

- `p = 0,000`

### Execução 2 — Efeitos Fixos de Zona + Controles Demográficos

- **Coeficiente $\beta_1$ (`IS_UE2020`):** `-0,0012`
- `p = 0,523`
- `z = -0,639`

### Intervalo de Confiança de 95%

$$
[-0,005,\ +0,002]
$$

### Diagnóstico

Mesmo dentro das zonas mistas, a diferença bruta de **-1,75 p.p.** decorre da distribuição das urnas entre escolas de bairros centrais e periféricos.

Ao controlar o perfil intra-zona, o coeficiente retorna para patamar indistinguível de zero:

- **-0,12 p.p.**
- `p = 0,523`

---

# 4. Conclusão Final e Evidências Visuais

## A Resposta à Pergunta de Pesquisa

**É possível identificar associação estatística entre o modelo de urna eletrônica utilizado em uma seção e a distribuição agregada dos votos para presidente no segundo turno de 2022, depois de considerar diferenças geográficas e de composição eleitoral?**

**Não.**

A hipótese nula de ausência de efeito:

$$
H_0: \beta_1 = 0
$$

não pode ser rejeitada em nenhuma especificação controlada.

Não existe associação estatística nem substantiva entre o modelo da urna e o percentual de votos obtido pelos candidatos no segundo turno de 2022.

---

# O Que Sustenta Nossas Descobertas: Quadro Comparativo

| Especificação Econométrica | Coeficiente (`IS_UE2020`) | Erro-Padrão | Valor-p | Conclusão Técnica |
|---|---:|---:|---:|---|
| 1. OLS Ingênuo (Sem Controles) | `-0,0419` | `0,010` | `0,000` | Viés de seleção geográfico severo. |
| 2. OLS com Controles Demográficos | `+0,0227` | `0,009` | `0,013` | Paradoxo de Simpson: reversão do sinal. |
| 3. Efeitos Fixos de Município + Controles | `+0,0006` | `0,003` | `0,846` | Efeito nulo nacional ($\beta \approx 0$). |
| 4. Efeitos Fixos de Zona Mista + Controles | `-0,0012` | `0,002` | `0,523` | Efeito nulo local confirmado ($\beta \approx 0$). |

A trajetória dos coeficientes estimados ao longo do estudo sintetiza o colapso do efeito do hardware.

---

# Análise dos Gráficos Gerados e Evidências Empíricas

## 1. O Contraste Capital vs. Interior: A Gênese do Viés

O cruzamento entre o modelo de urna e o tipo de localidade evidenciou a assimetria logística de distribuição implementada pela Justiça Eleitoral.

![alt text](image-1.png)

### Penetração das Urnas

Nas Capitais:

- **79,6%** das seções operaram com o modelo UE2020.

No Interior:

- As máquinas legadas predominaram amplamente em **69,5%** das seções.

### Resultado Eleitoral

Nas Capitais:

- Jair Bolsonaro venceu com **50,3%**;
- Lula obteve **49,7%**;
- Vantagem de Bolsonaro: **+0,6 p.p.**

No Interior:

- Lula venceu com **51,3%**;
- Bolsonaro obteve **48,7%**;
- Vantagem de Lula: **+2,6 p.p.**

### Evidência

Como quase **80% das urnas das capitais eram UE2020** e quase **70% das urnas do interior eram legadas**, a comparação ingênua de urnas nada mais era do que uma comparação indireta entre a preferência política das capitais contra a do interior.

---

## 2. Distribuição por Macrorregiões: A Desconexão entre Equipamento e Voto

A visualização regional desfez a premissa de que a urna UE2020 beneficiaria ou prejudicaria algum candidato.

![alt text](image-2.png)

### Proporção Uniforme do Hardware

A penetração do modelo UE2020 manteve-se quase homogênea em todas as grandes regiões brasileiras:

| Região | Proporção de urnas UE2020 |
|---|---:|
| Sudeste | 39,6% |
| Sul | 40,3% |
| Nordeste | 41,0% |
| Norte | 43,4% |
| Centro-Oeste | 45,9% |

### Disparidade Extrema na Votação

Enquanto a proporção de urnas novas variou menos de **6 pontos percentuais** entre os extremos regionais, a votação dos candidatos oscilou fortemente por dinâmicas históricas.

#### Nordeste

Com **41,0%** de urnas UE2020:

- Lula obteve **69,3%** dos votos válidos.

#### Sul

Com praticamente a mesma fatia de UE2020:

- **40,3%**

Bolsonaro obteve:

- **61,9%**

#### Centro-Oeste

Região com a maior concentração de UE2020 do país:

- **45,9%**

Bolsonaro atingiu:

- **60,2%**

### Evidência

Se o modelo de equipamento possuísse algum efeito sistemático nos votos, regiões com proporções praticamente idênticas de máquinas novas — como:

- Sul: **40,3%**
- Nordeste: **41,0%**

— apresentariam comportamentos correlacionados.

Isso não ocorre na realidade dos dados.

---

# Considerações Finais

As evidências empíricas, econométricas e visuais convergem para uma constatação inequívoca:

> A aparente discrepância observada na média nacional bruta é inteiramente explicada pela demografia do eleitorado e pela alocação geográfica assimétrica das urnas entre capitais e interior.

Controlados esses fatores, o modelo da urna eletrônica é **estatisticamente neutro em relação aos votos contabilizados**.
