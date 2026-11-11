# Dados

Esta pasta guarda as bases usadas no projeto de clusterização (DBSCAN) de acidentes de trânsito em Nova York que foram extraidas na data de 11 de novembro de 2026.

## Fonte

[Motor Vehicle Collisions – Crashes](https://data.cityofnewyork.us/Public-Safety/Motor-Vehicle-Collisions-Crashes/h9gi-nx95), do NYC Open Data (identificador `h9gi-nx95`). Os dados são obtidos pela API Socrata do portal, filtrando pela coluna `crash_date`.

| Arquivo | Conteúdo |
|---|---|---|
| `colisoes_2024_bruto.csv` | Todas as linhas de 2024, sem nenhum tratamento. Usado na análise de estabilidade 2024 × 2025. |
| `colisoes_2025_bruto.csv` | Todas as linhas de 2025, sem nenhum tratamento. Base principal da análise. |
| `base-processada/colisoes_2025_limpo.csv` | 2025 sem coordenadas ausentes, sem coordenadas zero e restrito à caixa geográfica de NYC. |

Os arquivos `_bruto` não são modificados pelo notebook. Toda limpeza acontece em memória, e o resultado vai para `base-processada/`.

## Como o notebook usa estes arquivos

O notebook chama `carregar_ano(ano)`, que segue esta regra:

1. Se `data/colisoes_<ano>_bruto.csv` **existe**, lê o arquivo e **não acessa a API**.
2. Se **não existe**, baixa o ano inteiro da API, salva o arquivo em `data/` e segue usando os dados baixados. Nas execuções seguintes, vale o caso 1.

Ou seja, a API só é chamada na primeira execução de cada ano. Depois disso o notebook roda offline e sempre sobre os mesmos dados.

## Como atualizar os dados

O portal da NYC pode corrigir ou complementar registros antigos. Se você quiser uma versão mais nova de um ano, apague o arquivo correspondente e rode de novo. Se fizer isso, rode também o notebook inteiro, porque os resultados em `resultados/` dependem dos dados.

## Observações sobre os dados

- `collision_id` e `zip_code` são lidos como **texto** (`str`), para não perder zeros à esquerda. O notebook e o script usam a mesma configuração ao ler a API e os CSVs.
- `crash_date` é lido como texto e convertido para data no notebook.
- A coluna `location` é redundante com `latitude` e `longitude` e é descartada na limpeza.
- Uma parte das colisões não tem coordenadas (ou vem com 0). Esses registros permanecem nos arquivos `_bruto` e só são removidos na limpeza; o funil de volumetria está em `resultados/tabelas/funil_volumetria.csv`.