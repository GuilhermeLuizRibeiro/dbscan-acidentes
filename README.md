# Concentrações espaciais de acidentes de trânsito em Nova York com DBSCAN

Seminário avaliativo de **Aprendizagem de Máquina Não Supervisionada** (Grupo 03 · Clustering baseado em densidade).

- **Disciplina:** Data Science  
- **Docente:** Ricardo Stegh Camati
- **Integrantes:**

## Pergunta de pesquisa

Onde estão as concentrações espaciais de acidentes de trânsito em Nova York e como elas dependem dos parâmetros de densidade (`eps` e `min_samples`)?

O DBSCAN é ajustado apenas sobre latitude e longitude, com distância **haversine** (coordenadas em radianos). Não há rótulos nem variável-alvo. Colunas como feridos e mortos só são usadas depois do ajuste, para descrever os clusters.

> **Aviso:** número de acidentes não é risco. A base não traz volume de tráfego nem de pedestres (exposição). Os clusters mostram onde os registros se concentram, não onde é mais perigoso circular.

## Dados

- **Fonte:** [Motor Vehicle Collisions – Crashes](https://data.cityofnewyork.us/Public-Safety/Motor-Vehicle-Collisions-Crashes/h9gi-nx95), NYC Open Data (identificador `h9gi-nx95`), obtida pela API Socrata com filtro em `crash_date`.
- **Recorte:** ano de **2025** completo (base principal) e **2024** (apenas para a análise de estabilidade).
- **Extração:** 11 de novembro de 2026. O portal pode corrigir registros antigos, então uma nova extração pode alterar os números levemente.
- **Volumetria (2025):**

Cada linha da tabela mostra quantas linhas **restam** após a etapa de limpeza indicada:

| Etapa | Linhas | % do total |
|---|---:|---:|
| Linhas baixadas | 85.559 | 100,0 |
| Após remover coordenadas ausentes (com latitude e longitude) | 84.833 | 99,2 |
| Após remover coordenadas (0, 0) | 82.918 | 96,9 |
| Após filtrar a caixa geográfica de NYC | 82.918 | 96,9 |
| Pontos únicos (lat, lon) — diagnóstico, não é filtro | 44.111 | 51,6 |

Foram removidas 726 linhas sem coordenadas e 1.915 linhas em (0, 0); nenhuma ficou fora da caixa de NYC. Em 2024 restam 83.697 linhas válidas após o mesmo tratamento. Detalhes sobre os arquivos e a regra de download estão em [`data/README.md`](data/README.md).

## Como reproduzir

Requer **Python 3.12 ou superior** (o notebook foi executado em Python 3.14).

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
pip install jupyterlab           # ou use VS Code / Colab com kernel Python
cd notebooks
jupyter lab
```

Abra `dbscan-acidentes.ipynb` e execute **Kernel → Restart & Run All**.

- O notebook usa caminhos relativos (`../data`, `../resultados`), então o diretório de trabalho deve ser `notebooks/`.
- Os CSVs brutos já estão em `data/`, então a execução é **offline**. Se um arquivo não existir, o notebook baixa o ano da API e o salva em `data/`. Para atualizar a base, apague o CSV do ano e rode tudo de novo.
- Todas as tabelas e figuras em `resultados/` são regeneradas pelo notebook.
- A pasta `data/base-processada/` (2025 já limpo) **não vem no repositório**: ela é gerada pelo notebook na etapa de limpeza e está no `.gitignore`.

## Metodologia

1. **Validação e limpeza.** Conferência dos 12 meses, de datas fora do ano e de IDs duplicados. Remoção de coordenadas ausentes, de coordenadas (0, 0) (marcador de posição desconhecida) e de pontos fora da caixa de NYC. O funil de linhas fica registrado em `resultados/tabelas/funil_volumetria.csv`.
2. **Pontos únicos com peso.** Muitos acidentes compartilham a mesma coordenada. O DBSCAN é ajustado sobre os pontos únicos, usando o número de acidentes de cada ponto como `sample_weight`, o que equivale a usar todas as linhas com menos memória.
3. **Escolha dos parâmetros.** Gráfico de k-distância (haversine, em metros) para delimitar a faixa de `eps`, sem joelho nítido, e grade de `eps` ∈ {50, 100, 150, 250, 400} m × `min_samples` ∈ {5, 10, 20}.
4. **Três configurações.** A e B ficam antes do encadeamento de clusters. C usa `eps` maior, como contraste.
5. **Avaliação.** Proporção de ruído, número e tamanho dos clusters, extensão dos clusters e Silhouette com haversine (apenas pontos fora do ruído, amostra de 10 mil pontos, 5 sementes).
6. **Robustez.** Estabilidade espacial entre 2024 e 2025 (com grupo de controle) e sensibilidade a um artefato de geocodificação (ponte Verrazzano).
7. **Interpretação.** Maiores clusters, vias envolvidas e gravidade descritiva dos acidentes, sem leitura de risco.

## Resultados principais

| Config | `min_samples` | `eps` (m) | Clusters | Ruído (%) | Maior cluster (%) | Silhouette | Extensão mediana (m) | Acidentes em clusters > 2 km (%) | Cobertura 25→24 / 24→25 (%) |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| **A** | 10 | 100 | 1.620 | 35,7 | 3,7 | 0,271 ± 0,006 | 241 | 15,5 | 87,4 / 86,9 |
| B | 20 | 150 | 605 | 40,3 | 5,0 | 0,191 ± 0,006 | 454 | 37,6 | 90,5 / 91,0 |
| C | 10 | 250 | 117 | 2,3 | 57,4 | −0,584 ± 0,013 | 564 | 96,3 | n/a |

- **Efeito do raio.** Com `min_samples = 10`, ao passar de `eps` = 100 m para 400 m o ruído cai de 35,7% para 0,4%, o número de clusters vai de 1.620 para 25 e o maior cluster passa de 3,7% para 92,8% dos acidentes (encadeamento).
- **Estabilidade.** Cerca de 87–91% dos acidentes agrupados em um ano ficam a até `eps` metros de pontos agrupados no outro. Para o ruído de 2025 o controle fica em 27–28%.
- **Ponte Verrazzano.** São 224 registros (0,27% da base) em apenas 2 coordenadas. Removê-los não altera a partição das configurações A e B (ARI = 1,0).
- **Maiores clusters.** São corredores de avenidas, por exemplo 2 Ave / 3 Ave / W 42 St em Midtown (A: 3.100 acidentes) e E/W 125 St no Harlem, e não pontos isolados.

## Limitações

- O Silhouette favorece grupos compactos, e o DBSCAN encontra formas arbitrárias. Ele é usado junto com ruído, tamanho, extensão, mapas e estabilidade, e cada configuração avalia uma fração diferente da base (a que não é ruído).
- O `eps` limita a distância entre vizinhos conectados, não o tamanho do cluster. Mesmo em A e B há clusters de vários quilômetros ao longo de avenidas.
- A coordenada de pontes e vias expressas representa um trecho longo, e não um cruzamento.
- A estabilidade mede proximidade entre anos, não identidade de clusters.
- Sem dados de exposição, os clusters não permitem concluir que um local é mais perigoso.

## Sementes e determinismo

O DBSCAN é determinístico. A semente `SEED = 40` só controla a amostragem do Silhouette (`random_state = SEED + i`, com `i` de 0 a 4).

## Referências

- Ester, M.; Kriegel, H.-P.; Sander, J.; Xu, X. *A density-based algorithm for discovering clusters in large spatial databases with noise.* KDD, 1996.
- Pedregosa, F. et al. *Scikit-learn: Machine Learning in Python.* JMLR, 12, 2011.
- NYC Open Data. *Motor Vehicle Collisions – Crashes.* <https://data.cityofnewyork.us/Public-Safety/Motor-Vehicle-Collisions-Crashes/h9gi-nx95>