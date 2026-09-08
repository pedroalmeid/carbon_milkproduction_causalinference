# Análise Causal: Produtividade e Emissões na Pecuária Leiteira

> DAG de modelagem causal, clustering de dietas em fazendas leiteras e inferência causal para avaliação de trade-offs entre produtividade leiteira e emissões entéricas de carbono.

![Python](https://img.shields.io/badge/Python-3.11-blue.svg)
![DoWhy](https://img.shields.io/badge/DoWhy-Causal%20Inference-orange.svg)

---

## 1. Contexto & Objetivos

Na pecuária leiteira moderna, o equilíbrio entre ganhos de produtividade e mitigação de impactos ambientais — em especial as emissões entéricas de gases de efeito estufa — constitui um desafio de sustentabilidade e eficiência operacional. Sobretudo para pequenos e médios produtores, analisar esse equilíbrio com baixo custo (proporcionado por técnicas de ML) é muito importante para sua competitividade no mercado de carbono. 

Todavia, modelos puramente correlacionais e preditivos de Machine Learning apresentam limitações para apoiar intervenções: eles identificam associações estatísticas, mas não respondem o que aconteceria caso uma nova estratégia alimentar fosse implementada de forma generalizada.

Este projeto aplica técnicas de **Inferência Causal** e **Aprendizado Não Supervisionado** com os seguintes objetivos:
1. **Identificar perfis de dietas** a partir da clusterização de insumos fornecidos aos rebanhos.
2. **Formalizar o sistema produtivo do leite (com perspectiva nas emissões) em um Grafo Acíclico Dirigido (DAG)**, mapeando relações estruturais de causa e efeito.
3. **Simular intervenções contrafactuais** do tipo $do(\text{Dieta} = k)$ via modelos estruturais causais, quantificando o trade-off entre produtividade e emissões.

---

## 2. Fundamentação Teórica & Modelagem de Domínio (DAG)

A etapa de maior dedicação analítica do projeto consistiu na tradução de conceitos do domínio expostos em **Avaliações do Ciclo de Vida (ACV/LCA)** para uma especificação formal em Grafo Causal (DAG) tecnicamente viável.

* **Documentação Teórica**: A fundamentação biológica e agronômica de cada nó e arco causal está documentada em detalhes em [`docs/Documentação Grafo Causal Produção de Leite.pdf`](docs/Documentação%20Grafo%20Causal%20Produção%20de%20Leite.pdf).
* **DAG de Domínio Ideal vs. DAG Representado nos Dados**:
  * [`model/causal_dag.dot`](model/causal_dag.dot): Representação completa de domínio, contemplando variáveis não observadas na base amostral (como raça predominante e sistema de produção).
  * [`model/causal_dag_datarepresented.dot`](model/causal_dag_datarepresented.dot): Subgrafo estritamente mapeado sobre as variáveis medidas disponíveis no experimento que realizamos (dataset que tivemos acesso).

---

## 3. Pipeline Metodológico

O fluxo analítico está dividido em etapas sequenciais:

1. **Pré-processamento & Redução de Dimensionalidade**:
   * Filtragem de 34 variáveis de insumos alimentares presentes no dataset de fazendas.
   * Eliminação de 16 variáveis alimentares com variância nula (consumo zero em todas as propriedades).
2. **Clustering Hierárquico de Estratégias Alimentares**:
   * Aplicação de *Agglomerative Hierarchical Clustering* sobre as 18 variáveis remanescentes.
   * Definição de 4 clusters representativos de estratégias de alimentação (`Dietary_Strategy_Cluster`: 0, 1, 2 e 3).
3. **Consumo da Modelagem Causal**:
   * Instanciações dos modelos causais via biblioteca **DoWhy** (`dowhy.gcm`), estruturados a partir do DAG representado em NetworkX.
   * Parametrização dos mecanismos causais: nós contínuos ajustados por modelos de regressão linear aditiva e nós categóricos por classificadores Random Forest.
4. **Simulação Contrafactual & Avaliação de Trade-off**:
   * Amostragem intervencional de $do(\text{Dieta} = k)$ (via variável `Dietary_Strategy_Cluster`) para cada cluster $k \in \{0, 1, 2, 3\}$.
   * Estimação das médias causais absolutas de Produtividade esperada $E[\text{Produtividade} \mid do(\text{Dieta}=k)]$ e Emissões esperadas $E[\text{Emissoes} \mid do(\text{Dieta}=k)]$ (referentes às variáveis `Productivity` e `co2_enteric_fermentation`).
   * Cálculo da razão de eficiência: $\text{Ratio} = \frac{\text{Produtividade}}{\text{Emissoes}}$.
5. **Inspeção de Força Causal (*Arrow Strength*)**:
   * Avaliação da magnitude direta do arco $\text{Dieta} \to \text{Emissoes}$ e $\text{Dieta} \to \text{Produtividade}$.

---

## 4. Principais Resultados & Análise Crítica

### Trade-off Causal Observado
As intervenções contrafactuais produziram os seguintes resultados de ratio produtividade-emissões entre os perfis alimentares:

| Cluster | Produtividade Média (L/animal) | Emissões Entéricas Médias (kg CO₂eq) | Razão de Eficiência (x10k) |
| :---: | :---: | :---: | :---: |
| **0** | 15.88 | 113.355,75 | 1.40 |
| **1** | **23.00** | 118.764,25 | **1.94** |
| **2** | 16.43 | 115.448,75 | 1.42 |
| **3** | 19.34 | 122.475,75 | 1.58 |

O **Cluster 1** destacou-se pela maior produtividade e melhor razão de eficiência relativa por unidade de emissão gerada.

### Avaliação Crítica da Força Causal (*Arrow Strength*)
A análise de sensibilidade direta dos arcos do modelo revelou:
* **Força direta da Dieta na Produtividade**: `8.5266` (impacto relevante e direto).
* **Força direta da Dieta nas Emissões Entéricas**: `0.0000` (impacto direto nulo).

**Diagnóstico Técnico (Identificação de problema de multicolinearidade no dataset testado)**: O valor nulo deveu-se à multicolinearidade perfeita e à natureza dos dados de emissão da base analisada. Os valores de emissão foram calculados por equações determinísticas que utilizavam diretamente a contagem e o peso das categorias animais como fatores primários de emissão, sem variabilidade atribuível às dietas.

### Se os resultados são limitados para este dataset, qual é o valor que fica do projeto?

A identificação da multicolinearidade e da força de seta nula, sob a perspectiva de auditoria causal de dados, demonstra que as conclusões pontuais de trade-off deste dataset específico não devem ser adotadas para tomadas de decisão. No entanto, o projeto entrega ativos técnicos e analíticos de alto valor:

* **Modelagem Causal de Domínio Reutilizável**: A formalização do DAG (derivada do Ciclo de Vida analisado) mapeia estruturalmente a pecuária leiteira e serve como documentação de domínio robusta para quaisquer análises de dados no setor.
* **Diagnóstico e Segmentação de Dietas (Clustering)**: O pipeline não supervisionado sobre as 18 variáveis alimentares gerou agrupamentos que oferecem insights analíticos válidos sobre as práticas alimentares dos rebanhos estudados.
* **Pipeline em DoWhy Calibrado para Dados Observacionais**: A arquitetura de inferência causal desenvolvida está totalmente pronta  para ser reaproveitada em bases onde as emissões provenham de medições empíricas diretas, permitindo avaliar com real confiabilidade estatística o trade-off entre produtividade e emissões nesses potenciais datasets.
* **Rigor Metodológico e Auditoria de Dados**: Evidencia na prática de como testes de sensibilidade causal (*arrow strength*) evitam conclusões precipitadas em cenários de *target leakage* ou variáveis determinísticas calculadas por fórmula.

Mais detalhes dessa discussão e a análise completa estão disponíveis na seção de conclusões do notebook [`notebooks/causal_inference_analysis.ipynb`](notebooks/causal_inference_analysis.ipynb). 

---

## 5. Confidencialidade dos Dados & Reprodutibilidade

Por razões de **sigilo e proteção de dados das propriedades rurais**, os arquivos CSV contendo os dados brutos e intermediários das fazendas não estão incluídos neste repositório, encontrando-se devidamente ignorados via [`.gitignore`](.gitignore).

---

## 6. Stack Tecnológica & Estrutura do Repositório

### Tecnologias Utilizadas
* **Linguagem**: Python 3.11
* **Inferência Causal**: DoWhy (`dowhy.gcm`), NetworkX
* **Machine Learning & Estatística**: Scikit-Learn, Pandas, NumPy
* **Visualização de Dados**: Seaborn, Matplotlib, Graphviz (PyDot)
* **Ambiente**: VS Code Dev Containers, Docker

### Jupyter Notebooks
Todo o código foi estruturado através de Jupyter Notebooks. Portanto, o pipeline desenvolvido pode ser analisado nas células de código desses notebooks, da mesma forma em que os resultados diretos obtidos também estão dispostos no corpo desses notebooks. 

### Estrutura de Pastas
```text
├── .devcontainer/
│   └── Dockerfile                          # Ambiente Docker com Python 3.11 e Graphviz
├── data/
│   ├── processed/                          # Base processada com clusters (ignorado no git)
│   └── raw/                                # Base bruta de fazendas (ignorado no git)
├── docs/
│   └── Documentação Grafo Causal Produção de Leite.pdf  # Fundamentação teórica do DAG
├── model/
│   ├── causal_dag.dot                      # Grafo causal completo de domínio
│   ├── causal_dag.py                       # Especificação do grafo em formato Python
│   └── causal_dag_datarepresented.dot      # Grafo restrito às variáveis observadas
├── notebooks/
│   ├── dietary_strategy_clustering.ipynb   # Redução e clusterização das dietas
│   └── causal_inference_analysis.ipynb     # Modelagem DoWhy, GCM e contrafactuais
├── .gitignore 
├── README.md 
└── requirements.txt                        # Lista de dependências Python
```

---

## 7. Como Executar o Projeto

### Pré-requisitos
* Python 3.11+
* Pacote de sistema `graphviz` (necessário para manipulação e visualização de grafos com Pydot/NetworkX).

### Opção 1: VS Code Dev Containers (Recomendada)
O repositório já conta com ambiente Docker configurado:
1. Certifique-se de que o Docker está instalado na máquina.
2. Abra a pasta no VS Code com a extensão **Dev Containers** instalada.
3. Selecione a opção **Reopen in Container** (Ctrl + Shift + P). As dependências e o Graphviz serão instalados automaticamente.

### Opção 2: Instalação Local
```bash
# Clone o repositório
git clone https://github.com/seu-usuario/carbon_milkproduction_causalinference.git
cd carbon_milkproduction_causalinference

# Crie e ative um ambiente virtual
python3 -m venv .venv
source .venv/bin/activate   # Linux/macOS
# .venv\Scripts\activate    # Windows

# Instale as dependências
pip install -r requirements.txt
```

> **Nota**: Caso execute localmente fora do container (opção 1), certifique-se de instalar o utilitário `graphviz` em seu sistema operacional (ex.: `sudo apt-get install graphviz` no Ubuntu/Debian).

---

## 8. Autor & Contato

Desenvolvido por **Pedro Almeida**.

* **LinkedIn**: [linkedin.com/in/pedroalmeid](https://www.linkedin.com)
* **GitHub**: [github.com/pedroalmeid](https://github.com/pedroalmeid)
* **E-mail**: pedrojos.campos@protonmail.com
* **Portfólio**: [https://pedroalmeid.github.io/portfolio](https://pedroalmeid.github.io/portfolio)

