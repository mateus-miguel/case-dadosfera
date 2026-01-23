# Pipeline de Enriquecimento de Dados com LLM (GPT-4o-mini)

Este projeto implementa um pipeline de ETL (Extract, Transform, Load) inteligente e escalável, focado na normalização e enriquecimento semântico de uma base de dados de produtos (1M+ registros). A solução utiliza LLMs para descoberta dinâmica de esquemas e processamento em lote para garantir resiliência.

## 🏗️ Arquitetura do Sistema

A solução foi desenhada para operar de forma modular, utilizando o **AWS S3** como camada de persistência (Data Lake) e o modelo **GPT-4o-mini** da OpenAI como motor de inteligência.

### Fluxo de Dados (Pipeline)

O processo está dividido em quatro etapas principais, conforme ilustrado no diagrama `Dadosfera ETL LLM.png`:

<p align="center">
   <img src='https://raw.githubusercontent.com/mateus-miguel/case-dadosfera/refs/heads/main/metadata/Dadosfera%20ETL%20LLM.png' width='600'/>
</p>

1.  **Limpeza e Normalização (`ETL 1`):**
    * **Entrada:** `products_raw.jsonl` (Dados brutos).
    * **Processamento:** Limpeza de strings, tratamento de valores nulos e filtragem inicial.
    * **Saída:** `products_clean.json` (Armazenado no bucket S3 `dadosfera-datalake`).

2.  **Descoberta de Atributos com LLM (`ETL 2`):**
    * **Inovação:** Em vez de um schema fixo, o script utiliza o LLM para analisar uma amostra dos produtos e inferir quais atributos são relevantes para extração.
    * **Saída:** `schema.json` (Um contrato de dados dinâmico).

3.  **Enriquecimento Semântico em Lote (`ETL 3`):**
    * **Lógica de Batching:** Utiliza a estratégia detalhada no diagrama `Batch LLM.png`. O processamento é feito em lotes de **10 a 20 produtos**.
    * **Resiliência:** A abordagem incremental evita a perda de progresso em caso de interrupções e otimiza o uso da API da OpenAI.
    * **Saída:** `products_enriched.json`.

4.  **EDA e Análise Visual (`EDA Visuals`):**
    * **Ação:** Script final que consome os dados enriquecidos para gerar insights, gráficos de distribuição de atributos e validação da normalização realizada.

---

## 🧼 Limpeza e Normalização (ETL 1)

O objetivo desta etapa é realizar o "saneamento" dos dados. Trabalhar com arquivos massivos (mais de 1 milhão de registros) em formatos semi-estruturados como `.jsonl` exige um tratamento robusto contra erros de sintaxe e registros incompletos que poderiam comprometer o desempenho do LLM ou gerar custos desnecessários.

### 📋 Detalhes do Processo

1.  **Leitura Resiliente (Tratamento de Erros de Sintaxe):**
    * O arquivo original `products_raw.jsonl` continha inconsistências de formatação (valores faltantes, linhas vazias).
    * **Solução:** Implementação de um bloco `try-except` com `json.JSONDecodeError`. Isso permite que o script ignore linhas corrompidas e continue o processamento sem interromper o pipeline, garantindo a integridade da ingestão.

2.  **Filtragem de Atributos Essenciais:**
    * Para que o enriquecimento semântico funcione, o produto **precisa** ter conteúdo textual. 
    * **Ação:** Remoção automática de qualquer registro onde os campos `title` (título) ou `text` (descrição/corpo) estivessem vazios ou nulos.

3.  **Normalização de Codificação:**
    * Forçamento do padrão `utf-8` na leitura e escrita para evitar erros de caracteres especiais (acentuação e símbolos comerciais), comuns em bases de produtos brasileiros.
    * Normalização por dicionário remove aspas não fechadas, tipográficas “ ”, outros símbolos irrelevantes como *, ✔, ➤, ™, ® e trechos de marketing como "Product Description", "Next Page".

4.  **Gestão de Volume e Amostragem:**
    * Dada a escala de 1M+ de produtos, o script foi configurado para segmentar os dados (ex: processando os primeiros 100.000 registros para validação inicial), permitindo testes de custo-benefício antes do processamento total via API da OpenAI.

5.  **Persistência no Data Lake (AWS S3):**
    * O resultado limpo é exportado para o arquivo `products_clean.json`.
    * O upload é feito via biblioteca `boto3` diretamente para o bucket `dadosfera-datalake`, servindo como a "Single Source of Truth" (datalake) para os scripts subsequentes de LLM.

### 🛠️ Especificações Técnicas
* **Input:** `s3://dadosfera-datalake/bronze/products_raw.jsonl`
* **Output:** `s3://dadosfera-datalake/silver/products_clean.json`
* **Principais bibliotecas:** `json`, `boto3`, `os`.
* **Lógica de Filtro:** `if doc['title'] != '' and doc['text'] != '':`

---

## 🔍 Descoberta de Atributos (ETL 2)

Nesta etapa, o pipeline utiliza inteligência artificial para transitar de um dado semi-estruturado para um esquema (schema) rigorosamente definido. Em vez de mapear manualmente centenas de possíveis colunas, o processo utiliza o modelo **GPT-4o-mini** para inferir a estrutura ideal com base no conteúdo real dos produtos.

### 📋 Detalhes do Processo

1.  **Amostragem Inteligente:**
    * O script consome uma amostra representativa do arquivo `products_clean.json` (armazenado no S3). Essa amostra é enviada ao LLM para que ele entenda a diversidade de categorias e propriedades presentes na base.

2.  **Engenharia de Prompt e Inferência:**
    * O LLM é instruído a identificar atributos técnicos (como voltagem, cor, material, dimensões, marca) que são recorrentes nas descrições.
    * O modelo retorna não apenas o nome do atributo, mas também o **tipo de dado** (string, float, integer) e uma **descrição funcional** do que aquele campo representa.

3.  **Extração via Regex (Expressões Regulares):**
    * Como a saída de um LLM pode conter textos explicativos, o script utiliza **Regex** (`re.findall`) para capturar com precisão os padrões de atributos, tipos e descrições dentro da resposta bruta da API.
    * **Lógica de Mapeamento:** Um dicionário `type_mapping` é utilizado para converter as sugestões de tipos do LLM (ex: "texto", "número") em tipos Python/JSON válidos (`str`, `float`, `int`).

4.  **Construção do Schema Dinâmico:**
    * O script consolida uma estrutura base fixa (contendo `id`, `title` e `text`) e anexa a ela todos os novos atributos "descobertos" pela IA.
    * Isso garante que o pipeline seja flexível: se a base de produtos mudar de "Eletrônicos" para "Moda", o schema se adaptará automaticamente sem alteração de código.

5.  **Persistência do Contrato de Dados:**
    * O esquema final é salvo no AWS S3 como `schema.json` (ou `schema_v2.json`). Este arquivo servirá como o guia mestre para a etapa subsequente de enriquecimento.

### 🛠️ Especificações Técnicas
* **Modelo:** `gpt-4o-mini` (OpenAI).
* **Parsing:** Biblioteca `re` (Regex) para tratamento de strings.
* **Tipagem:** Conversão dinâmica via `type_mapping.get(attr_type.lower(), str)`.
* **Output:** `s3://dadosfera-datalake/metadata/schema.json`.

---

## 🧠 Enriquecimento Semântico (ETL 3)

Esta é a etapa central de inteligência do pipeline. Nela, o conteúdo textual limpo é transformado em dados estruturados de alto valor, utilizando o modelo **GPT-4o-mini** para extrair informações específicas baseadas no esquema definido anteriormente.

### 📋 Detalhes do Processo

1.  **Orquestração de Lotes (Batch Processing):**
    * Conforme ilustrado no diagrama `Batch LLM.png`, o processamento não é feito um a um, o que seria ineficiente. Os produtos são agrupados em **lotes de 10 a 20 itens** por chamada de API.
    * **Objetivo:** Maximizar o uso da janela de contexto do modelo e reduzir a latência total do pipeline.

2.  **Estratégia de Resiliência e Salvamento Incremental:**
    * Dado o volume massivo de dados, o script foi desenhado para ser "stateful". Os resultados de cada lote enriquecido são gravados imediatamente ou em intervalos regulares no S3.
    * **Benefício:** Se houver uma interrupção na conexão ou limite de taxa (rate limit), o progresso não é perdido. O pipeline pode ser reiniciado a partir do último ponto de salvamento.

3.  **Extração Guiada por Contexto:**
    * O prompt enviado ao LLM inclui o `schema.json` gerado na etapa 2. Isso força a IA a devolver apenas os atributos desejados, seguindo rigorosamente os tipos de dados (String, Float, Integer) e descrições técnicas.
    * **Ação:** O modelo atua como um "parser" inteligente, identificando características como categoria, material, marca e *features* dentro de descrições muitas vezes confusas.

4.  **Monitoramento de Performance:**
    * Implementação de barras de progresso (`tqdm`) para acompanhar o tempo médio de resposta por produto (aprox. 9min por iteração de lote) e estimar o tempo total de conclusão para a base de 1 milhão de produtos.

### 🛠️ Especificações Técnicas
* **Inputs:** `s3://dadosfera-datalake/silver/products_clean.json` e `s3://dadosfera-datalake/metadata/schema.json`.
* **Output:** `s3://dadosfera-datalake/gold/products_enriched.json` (Versão incremental).
* **Motor de IA:** OpenAI `gpt-4o-mini`.
* **Principais bibliotecas:** `openai`, `boto3`, `json`, `tqdm`.

---

## 📊 EDA e Análise Visual

Após o enriquecimento dos dados via LLM, esta etapa final foca na extração de inteligência e validação da qualidade do pipeline. O objetivo é transformar os arquivos JSON estruturados em insights visuais e métricas de consistência.

### 📋 Detalhes do Processo

1.  **Carregamento e Unificação de Dados Enriquecidos:**
    * O script consome os diversos arquivos `products_enriched.json` gerados pelo processo de batching no S3.
    * **Ação:** Consolidação em algumas Series Pandas para análise estatística de certos atributos. Alternativa seria consolidação em um único DataFrame Pandas (mais elaborado e demorado).

2.  **Normalização de Atributos Extraídos:**
    * Como o LLM pode extrair valores em diferentes formatos, o script realiza uma normalização final (ex: converter 'EUA', 'US' e 'North America' em formatos únicos como 'United States').
    * **Foco:** Garantir que os atributos descobertos pelo `schema.json` sejam comparáveis graficamente.

3.  **Visualizações Gráficas:**
    * **Distribuição de Países de Origem:** Gráficos de barras mostrando a quantidade de países que mais produzem os produtos.
    * **Pie Chart de Categorias:** Análise de porcentagem de macro-categorias às quais os produtos pertencem, filtradas das categorias geradas pelo LLM.
    * **Gráfico de Barras Agrupado:** Análise de produtos a prova d'água agrupados em DataFrame com base nos países de origem.
    * **Gráfico de Barras Horizontais de Garantias:** Avaliação dos tempos de garantias mais ofertados para os produtos do dataset enriquecido.
    * **Violin Plot de Preços:** Utilizando dados de macro-categoria e países de origem, a cada produto é atribuído um valor sintético de preço para análise da distribuição contínua de valores.

### 🛠️ Especificações Técnicas
* **Input:** `s3://dadosfera-datalake/gold/products_enriched.json` ou `s3://dadosfera-datalake/gold/products_enriched_batch<number>.json`
* **Output:** Dashboard de visualizações e relatório de qualidade.
* **Principais bibliotecas:** `pandas`, `matplotlib`, `seaborn`, `boto3`.
* **Destaque:** Uso de técnicas de filtragem para lidar com a memória do Colab ao processar o volume massivo de dados enriquecidos.

---

## 🛠️ Stack Tecnológica

* **Linguagem:** Python (Executado via Google Colab).
* **Orquestração de Dados:** OS, Boto3 (Integração com AWS S3).
* **Inteligência Artificial:** OpenAI API (Modelo: `gpt-4o-mini`).
* **Processamento & Análise:** Pandas, JSONL, Matplotlib, Seaborn.
* **Armazenamento:** AWS S3 Bucket (`dadosfera-datalake`).

---

## 📂 Organização dos Arquivos

| Arquivo | Descrição |
| :--- | :--- |
| `ETL 1 - Limpeza.ipynb` | Tratamento inicial de 1 milhão de produtos. |
| `ETL 2 - Descoberta de Atributos (LLM).ipynb` | Geração automática do `schema.json`. |
| `ETL 3 - Enriquecimento Semântico (LLM).ipynb` | Extração de atributos via GPT em batches. |
| `EDA Visuals.ipynb` | Análise exploratória e visualização dos dados finais. |
| `Dadosfera ETL LLM.png` | Diagrama principal da arquitetura do pipeline. |
| `Batch LLM.png` | Diagrama do fluxo de processamento em lote para o LLM. |

---

## 🚀 Destaques do Projeto

* **Escalabilidade:** Capaz de lidar com grandes volumes de dados JSONL.
* **Economia de Tokens:** O uso do `gpt-4o-mini` aliado à limpeza prévia reduz drasticamente o custo operacional.
* **Recuperação de Falhas:** O sistema de save incremental nos batches garante que o processo possa ser retomado de onde parou.

---
*Este projeto foi desenvolvido como parte de um estudo de caso para a Dadosfera.*
