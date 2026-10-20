# Dashboard-Barragens
# Análise Integrada de Barragens de Mineração

**Risco, estrutura e situação operacional das barragens do SIGBM (ANM), em um painel de Power BI.**

> Projeto de portfólio por **Alessa Cruz** · Aletheia Analytics

## Resumo

Painel interativo em Power BI com **3 páginas** que analisa as **908 barragens de mineração** cadastradas no SIGBM (Sistema Integrado de Gestão de Barragens de Mineração), da Agência Nacional de Mineração (ANM). O projeto cobre o ciclo completo: leitura de um CSV público com 124 colunas, limpeza e modelagem no Power Query, medidas em DAX e um dashboard construído para responder perguntas de negócio.

**Destaques**

- Dados reais, abertos e atualizados em 16/10/2026.
- 124 colunas reduzidas a 23, cada uma justificada por uma pergunta de análise.
- Tabela separada para materiais de construção, ligada por relação um-para-muitos.
- Conversão de coordenadas em graus-minutos-segundos para decimal, com sinalização de valores inválidos na fonte.
- Anonimização de pessoas físicas antes da publicação.

---

## Problema

Barragens de mineração concentram risco e exigem acompanhamento regulatório. Os dados públicos existem, mas estão em um arquivo único, largo (124 colunas), com marcadores de ausência inconsistentes, textos que misturam informações e coordenadas em formato inadequado para mapas.

O painel busca responder:

1. **Onde** estão as barragens com maior risco e maior dano potencial?
2. **Quais empresas** concentram barragens, inclusive as de risco alto?
3. **Altura, volume e método construtivo** se relacionam com a categoria de risco?
4. **Qual a situação das declarações de estabilidade**, e quais barragens exigem atenção?
5. **O que os dados permitem concluir, e o que não permitem?**

---

## Dados

| Item | Detalhe |
|---|---|
| Fonte | ANM, conjunto de dados abertos "Gestão de Barragens da Mineração (SIGBM)", publicado no portal dados.gov.br |
| Arquivo | "Dados das Barragens" (CSV) e arquivo de metadados (ODS) |
| Atualização do arquivo | 16/10/2026 |
| Volume | 908 linhas (uma por barragem) e 124 colunas |
| Codificação e separador | Windows-1252, campos separados por vírgula e entre aspas (o dicionário de dados indica ponto e vírgula, mas o arquivo real difere) |

**Os dados brutos não estão neste repositório.** O arquivo contém campos com dados pessoais (documentos mascarados e nomes de responsáveis técnicos). Para reproduzir o projeto, baixe o CSV direto do portal da ANM.

---

## Metodologia

Todo o tratamento foi feito no **Power Query**, de modo que cada decisão fica registrada como uma etapa e é reproduzível.

### 1. Importação e seleção de colunas
- Ajuste de delimitador e codificação na importação.
- 124 colunas reduzidas a **23**, escolhidas a partir das perguntas de análise. Ficaram de fora colunas quase vazias (como as de Back Up Dam), avaliações técnicas detalhadas e dados administrativos.

### 2. Tipos de dados
- Números no formato brasileiro (`4.106.897,00`) convertidos com **localidade pt-BR**. A detecção automática leu `8,00` como 800 sem gerar erro, e o erro só foi percebido ao comparar valores com o CSV original.
- Datas convertidas com localidade pt-BR.

### 3. Valores ausentes
- O marcador `-` foi tratado **antes** da conversão de tipo. Em datas, volume, método construtivo e material, virou `null`.
- Nas colunas de declaração (DCE), `-` significa "sem obrigação de envio" segundo o dicionário de dados, e foi mantido como categoria **"Sem obrigação"**, e não como dado faltante.
- Na DCO, o dicionário não explica o `-`, e foi rotulado como **"Sem informação"** (ver Limitações).
- "Desconhecido" em método construtivo foi mantido como **categoria própria**, porque é uma resposta declarada pela empresa, diferente de campo não preenchido.

### 4. Colunas com duas informações
- `Status DCE RISR`, `Status DCE RPSB` e `Status da DCO Atual` juntavam campanha ou tipo de envio com a situação. Foram separadas em duas colunas cada uma, com a situação (Atestado, Não Atestado, Não Enviado) isolada para análise.

### 5. Materiais de construção em tabela própria
- Uma barragem pode ter até 3 materiais na mesma célula (por exemplo, "Terra Homogênea; Rejeito"). Separar em linhas na tabela principal duplicaria barragens e distorceria todas as contagens.
- Solução: tabela `Materiais` (ID + material, **945 linhas**), ligada a `Barragens` por relação **um-para-muitos**, com filtro em um só sentido.

### 6. Coordenadas
- Latitude e longitude vinham como texto em graus, minutos e segundos (`-09°47'16.100''`).
- O **sinal foi guardado antes** da conversão, porque valores como `-00°44'...` perdem o sinal ao virar número.
- Segundos convertidos com localidade en-US, já que o ponto é decimal.
- Fórmula: `sinal × (graus + minutos/60 + segundos/3600)`.
- **6 barragens** têm minutos ou segundos fora do intervalo 0 a 59 na fonte. Foram mantidas e marcadas na coluna `Coordenada_suspeita`, em vez de removidas.

### 7. Privacidade
- 101 barragens têm empreendedor pessoa física (CPF mascarado), e outras 12 pertencem a 10 empreendedores com CNPJ registrado em nome de pessoa (identificados por revisão manual da lista de nomes). Nas **113 barragens** no total, o nome foi substituído por **"Pessoa física"**. As colunas de documento e de responsáveis técnicos foram removidas do modelo.

### 8. Validações
Conferências feitas durante o tratamento:

- 908 linhas ao final de cada etapa estrutural.
- Categoria de risco: Baixa 543, Média 297 e Alta 68.
- Estados: MG 318, MT 186, PA 120, BA 83 e SP 58.
- Tabela `Materiais`: 945 linhas e 5 materiais distintos.
- Latitude entre -30,9 e 2,3, longitude entre -63,6 e -36,3, e 21 barragens ao norte do Equador, o mesmo número da coluna `Posicionamento`.

---

## Modelo de dados

```
Barragens (1) ───────────< (*) Materiais
   ID                         ID, Material
```

- Relação **um-para-muitos** por `ID`, filtro em um só sentido (de `Barragens` para `Materiais`).
- Para contar barragens por material, conta-se as linhas de `Materiais`.
- Latitude e longitude com categoria de dados definida e sem resumo.

### Principais medidas (DAX)

| Medida | O que calcula |
|---|---|
| Total de barragens | Número de barragens (908) |
| Barragens com risco alto | Categoria de risco igual a Alta (68) |
| Barragens com dano potencial alto | Dano potencial igual a Alta (244) |
| % de risco alto | Risco alto sobre o total (7,5%) |
| Barragens em alerta ou emergência | Nível de emergência diferente de "Sem emergência" (79) |
| Risco alto e dano alto | Interseção das duas classificações (32) |
| % atestadas entre as obrigadas | Atestadas sobre (atestadas + não atestadas + não enviadas), 92,8% |

---

## O painel

### Página 1: Visão geral
Indicadores de risco, mapa por categoria de risco, matriz de risco (categoria de risco × dano potencial) e barras por estado, com filtros de estado, categoria de risco e situação operacional.

### Página 2: Empresas e características construtivas
Empresas com mais barragens e com mais barragens de risco alto, método construtivo por categoria de risco, relação entre altura e volume, materiais e ano de início de operação.

### Página 3: Declarações de estabilidade
Taxa de declarações atestadas, situação da declaração (DCE RISR), situação operacional e tabela de barragens em atenção (declaração não atestada ou não enviada).

Cada página tem seus próprios filtros: estado, categoria de risco e situação operacional (página 1); empreendedor, método construtivo e data de início de operação (página 2); e empreendedor (página 3).

---

## Principais insights

> Valores calculados sobre o arquivo de 16/10/2026. Categoria de risco e dano potencial são classificações regulatórias da ANM, e não previsões de rompimento.

1. **Concentração geográfica.** Cinco estados (MG, MT, PA, BA e SP) reúnem 765 das 908 barragens (84%). Minas Gerais sozinha tem 318.
2. **Risco alto é minoria, mas relevante.** São 68 barragens (7,5%) com categoria de risco Alta, e 244 (26,9%) com dano potencial Alto.
3. **Risco e dano potencial são coisas diferentes.** Apenas 32 barragens são Alta nas duas classificações, e 184 têm dano potencial Alto com risco Baixo. Olhar só uma das duas dimensões distorce a leitura.
4. **Método construtivo e risco.** Entre as 39 barragens com alteamento a montante, 14 (36%) têm risco Alto, contra 5% nas de etapa única e 5% nas de alteamento a jusante. É uma associação, e não prova de causa.
5. **Concentração por empresa.** As pessoas físicas somam 113 barragens (12,4%), o maior grupo, seguidas pela Vale, com 104 (11,5%). Em risco alto, as pessoas físicas somam 12, a Vale 7 e a ICOMI 6.
6. **Declarações.** Entre as 460 barragens obrigadas a declarar (DCE RISR), 92,8% estão atestadas, e 33 não atestaram ou não enviaram. Outras 448 não têm obrigação de envio.
7. **Alertas.** 79 barragens (8,7%) estão em alerta ou em algum nível de emergência.

---

## Limitações

- **Coordenadas:** 6 barragens têm minutos ou segundos inválidos na fonte (valores acima de 59). Estão marcadas em `Coordenada_suspeita`, e a posição no mapa pode estar deslocada.
- **Materiais:** 51 barragens não têm material informado e ficam fora das análises por material.
- **Barragens em construção (39):** não têm altura nem volume, e não aparecem nos gráficos que usam essas medidas.
- **Pessoas físicas:** foram anonimizadas pelo CPF mascarado e por uma lista manual de 10 nomes com CNPJ. Outros nomes de pessoas podem permanecer, porque a fonte não informa a natureza jurídica do empreendedor.
- **Nomes de empresas:** a Mineração Paragominas aparece com dois nomes para o mesmo CNPJ (um deles com "Filial:"), o que subconta suas barragens no agrupamento por nome.
- **DCO:** em 639 barragens o campo vem como `-`, e o dicionário de dados da ANM não explica o significado. Foi rotulado como "Sem informação", e nenhuma conclusão é tirada sobre elas.
- **Filtro de data (página 2):** o segmentador de data de início de operação exclui as 53 barragens sem data. Com ele ativo, os totais da página 2 ficam menores que os das demais páginas (por exemplo, 485 em vez de 487 para Terra Homogênea).
- **Dados declaratórios:** a ANM informa que os dados podem conter inconsistências.
- **Conformidade não é segurança:** uma declaração atestada indica cumprimento documental, e não que a barragem esteja segura.
- **Mapa:** depende de um serviço externo de mapas, que pode estar desabilitado em alguns ambientes corporativos.

---

## Como executar

1. Instale o **Power BI Desktop**.
2. Baixe o CSV "Dados das Barragens" no portal de dados abertos da ANM (SIGBM).
3. Abra `painel/Analise_Barragens.pbix`.
4. Em *Transformar dados > Configurações da fonte de dados*, aponte para o CSV baixado.
5. Clique em **Fechar e Aplicar**.

Se o arquivo for atualizado pela ANM, os totais mudam, e as validações listadas acima servem de referência para a versão de 16/10/2026.

---

## Habilidades demonstradas

Power Query (M) · Modelagem de dados · DAX · Visualização de dados · Limpeza de dados reais · Tratamento de dados pessoais · Documentação de decisões

---

## Contato

**Alessa Cruz**


---

## Fonte

Agência Nacional de Mineração (ANM). *Gestão de Barragens da Mineração (SIGBM)*, dados abertos, portal dados.gov.br.
