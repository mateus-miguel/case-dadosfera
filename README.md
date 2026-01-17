# Pipeline de Enriquecimento de Dados com LLM (GPT-4o-mini)

Este projeto implementa um pipeline de ETL (Extract, Transform, Load) inteligente e escalável, focado na normalização e enriquecimento semântico de uma base de dados de produtos (1M+ registros). A solução utiliza LLMs para descoberta dinâmica de esquemas e processamento em lote para garantir resiliência.

## 🏗️ Arquitetura do Sistema

A solução foi desenhada para operar de forma modular, utilizando o **AWS S3** como camada de persistência (Data Lake) e o modelo **GPT-4o-mini** da OpenAI como motor de inteligência.

### Fluxo de Dados (Pipeline)

O processo está dividido em quatro etapas principais, conforme ilustrado no diagrama `Dadosfera ETL LLM.png`:

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

## 🛠️ Stack Tecnológica

* **Linguagem:** Python (Executado via Google Colab).
* **Orquestração de Dados:** Boto3 (Integração com AWS S3).
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
