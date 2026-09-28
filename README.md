# Avaliação — APIs de Energia Renovável e Aprendizado de Máquina

## 1. Visão geral

Este projeto utiliza duas APIs públicas para coletar dados relacionados à geração de energia renovável e aplica técnicas de aprendizado de máquina em duas tarefas independentes.

### Objetivos

- Consultar dados públicos da **ANEEL/SIGA** sobre empreendimentos de geração de energia.
- Consultar dados históricos meteorológicos da **Open-Meteo**.
- Gerar dois arquivos CSV a partir das consultas.
- Comparar três algoritmos de classificação para identificar a fonte de um empreendimento.
- Comparar três algoritmos de regressão para estimar a radiação solar horária em Petrolina (PE).
- Avaliar os modelos utilizando métricas apropriadas para cada tarefa.

O notebook não utiliza tokens ou credenciais privadas.

---

## 2. Estrutura do projeto

O notebook está organizado nas seguintes etapas:

1. Consulta às APIs e geração dos CSVs
2. Imports e configuração
3. Tarefa 1 — Classificação da fonte de geração
4. Tarefa 2 — Regressão da radiação solar
5. Conclusões gerais

Os arquivos CSV gerados pelo notebook são:

- `aneel_classificacao_orange.csv`
- `meteo_regressao_orange.csv`

---

# 3. Coleta dos dados

## 3.1 ANEEL/SIGA

A primeira parte utiliza a API pública de dados abertos da ANEEL/SIGA.

São consultadas as seguintes siglas de geração:

- `UFV` → Solar
- `EOL` → Eólica
- `UHE` → Hidráulica
- `PCH` → Hidráulica
- `CGH` → Hidráulica

Para cada empreendimento são utilizadas três características:

- `potencia_kw` — potência outorgada em kW;
- `latitude` — latitude do empreendimento;
- `longitude` — longitude do empreendimento.

A variável alvo é:

- `fonte` — Solar, Eólica ou Hidráulica.

Registros sem potência ou coordenadas válidas são descartados.

### Observação importante

A potência utilizada é a **potência outorgada**, e não a energia efetivamente gerada.

Além disso, a distribuição das classes não representa a matriz elétrica real do Brasil, pois a consulta possui limite de registros por sigla e as classes hidráulicas agrupam três siglas diferentes.

---

## 3.2 Open-Meteo

A segunda fonte de dados é a API histórica da Open-Meteo.

O local utilizado é **Petrolina (PE)**, aproximadamente:

- Latitude: `-9.39`
- Longitude: `-40.50`

Período:

- Início: `01/04/2025`
- Fim: `30/06/2025`
- Fuso horário: `America/Recife`

Foram coletadas variáveis meteorológicas horárias:

- temperatura a 2 m;
- umidade relativa;
- cobertura de nuvens;
- velocidade do vento a 10 m;
- radiação solar de onda curta.

A análise utiliza apenas horários entre **7h e 17h** e remove registros com valores ausentes.

A variável alvo é:

`radiacao_w_m2`

Ela representa a radiação solar horizontal média da hora, em W/m².

> Os dados são provenientes de modelo/reanálise da Open-Meteo e não de um sensor físico ou de um painel solar.

---

# 4. Tarefa 1 — Classificação da fonte

## Pergunta

É possível classificar um empreendimento como **Solar, Eólica ou Hidráulica** utilizando apenas sua potência e localização?

## Entradas

As variáveis utilizadas como entrada foram:

```text
potencia_kw
latitude
longitude
```

A variável alvo foi:

```text
fonte
```

Não foram utilizadas informações que revelariam diretamente a classe, como a sigla do tipo de geração, nome, código CEG ou descrições do combustível.

## Divisão dos dados

Os dados foram divididos em:

- **80% para treinamento**
- **20% para teste**

A divisão foi:

- estratificada;
- realizada com `random_state=42`;
- igual para os três modelos.

## Modelos

### Regressão Logística

Utilizada como modelo de referência linear e interpretável.

Foi aplicada padronização com `StandardScaler` dentro de um `Pipeline`.

### KNN — k=5

Classifica cada empreendimento considerando os vizinhos mais próximos.

A padronização também foi feita dentro de um `Pipeline`.

### Random Forest

Utiliza várias árvores de decisão para capturar relações não lineares e interações entre potência e localização.

Não necessita de padronização.

---

# 5. Resultados da classificação

As métricas foram calculadas no mesmo conjunto de teste para os três modelos.

| Modelo | Accuracy | Precision (macro) | Recall (macro) | F1 (macro) | F1 (weighted) |
|---|---:|---:|---:|---:|---:|
| Regressão Logística | 0.825 | 0.828 | 0.821 | 0.820 | 0.823 |
| KNN (k=5) | 0.965 | 0.966 | 0.964 | 0.965 | 0.965 |
| Random Forest | 0.976 | 0.977 | 0.974 | 0.975 | 0.975 |

Na execução registrada no notebook, o **Random Forest apresentou o maior F1 macro**, com:

- Accuracy: **0.976**
- Precision macro: **0.977**
- Recall macro: **0.974**
- F1 macro: **0.975**

O conjunto de teste possui **776 empreendimentos**.

### Métricas por classe — Random Forest

| Classe | Precision | Recall | F1-score | Suporte |
|---|---:|---:|---:|---:|
| Eólica | 0.97 | 0.98 | 0.98 | 240 |
| Hidráulica | 0.96 | 0.99 | 0.98 | 296 |
| Solar | 1.00 | 0.95 | 0.97 | 240 |
| **Accuracy** | | | **0.98** | **776** |

### Erros mais frequentes

Os três pares de erros mais frequentes encontrados pelo notebook foram:

1. Solar → Hidráulica: **7 casos**
2. Solar → Eólica: **5 casos**
3. Eólica → Hidráulica: **4 casos**

Isso mostra que potência e localização fornecem bastante informação para a classificação neste conjunto de dados, mas não determinam perfeitamente a fonte.

---

# 6. Limitações da classificação

O próprio notebook destaca algumas limitações importantes:

1. A potência é a potência **outorgada**, não a energia efetivamente gerada.
2. As coordenadas são aproximadas.
3. O cadastro mistura empreendimentos em diferentes fases, incluindo planejados, em construção e em operação.
4. Não são utilizadas variáveis importantes como disponibilidade de rios, vento, irradiação, terreno ou proximidade de linhas de transmissão.
5. As classes são desbalanceadas pela forma como os dados foram coletados.
6. Foi utilizada apenas uma divisão treino/teste; uma validação cruzada poderia fornecer uma estimativa mais estável.

Portanto, o modelo é adequado como **exercício de aprendizado de máquina**, mas não deve ser interpretado como uma ferramenta para decidir a fonte de um empreendimento real.

---

# 7. Tarefa 2 — Regressão da radiação solar

## Pergunta

Dadas condições meteorológicas e a hora local em Petrolina (PE), é possível estimar a radiação solar horizontal média daquela hora?

## Entradas

Foram utilizadas cinco variáveis:

```text
temperatura_c
umidade_pct
nuvens_pct
vento_kmh
hora
```

A variável alvo foi:

```text
radiacao_w_m2
```

A coluna `data_hora` foi utilizada apenas para ordenar os dados e realizar a divisão temporal.

A radiação não foi incluída entre as entradas.

## Divisão temporal

Diferentemente da primeira tarefa, os dados não foram embaralhados.

Foram utilizados:

- primeiras **80% das horas** → treinamento;
- últimas **20% das horas** → teste.

Essa abordagem preserva a ordem temporal.

---

# 8. Modelos de regressão

## Regressão Linear

Serve como uma linha de base simples para verificar o desempenho de uma relação aproximadamente linear.

Foi utilizada padronização das variáveis dentro de um `Pipeline`.

## Random Forest Regressor

Utiliza árvores para aprender relações não lineares entre as variáveis meteorológicas, a hora do dia e a radiação.

## Gradient Boosting Regressor

Constrói árvores sequencialmente, utilizando os erros anteriores para melhorar as previsões.

---

# 9. Resultados da regressão

| Modelo | MAE (W/m²) | MSE ((W/m²)²) | R² |
|---|---:|---:|---:|
| Regressão Linear | 145.205 | 30034.201 | 0.360 |
| Random Forest | 66.252 | 7250.888 | 0.845 |
| Gradient Boosting | 67.082 | 7444.700 | 0.841 |

Na execução registrada no notebook, o **Random Forest apresentou o menor MAE**, com:

- MAE: **66.252 W/m²**
- MSE: **7250.888 (W/m²)²**
- R²: **0.845**

O resultado mostra uma diferença importante entre modelos lineares e modelos capazes de representar relações não lineares neste conjunto de dados.

---

# 10. Importância das variáveis

Na análise de importância das variáveis do Random Forest, os valores registrados foram:

| Variável | Importância |
|---|---:|
| Hora | 0.488 |
| Temperatura | 0.295 |
| Umidade | 0.179 |
| Nuvens | 0.022 |
| Vento | 0.016 |

A variável `hora` foi a mais importante no modelo.

Isso é consistente com o comportamento diário da radiação solar: os valores tendem a aumentar durante a manhã, atingir valores elevados próximos ao meio-dia e diminuir durante a tarde.

A cobertura de nuvens também influencia a radiação, embora sua importância no Random Forest desta execução tenha sido menor que a das demais variáveis listadas.

---

# 11. Por que radiação solar não é o mesmo que geração elétrica?

A variável prevista pelo segundo modelo é a **radiação global horizontal**, medida em W/m².

Ela não representa diretamente a quantidade de energia elétrica produzida por um sistema fotovoltaico.

A geração elétrica depende também de fatores como:

- eficiência dos módulos;
- área dos painéis;
- temperatura das células;
- inclinação e orientação dos módulos;
- perdas no inversor;
- perdas na fiação;
- sujeira;
- sombreamento;
- limitações da rede elétrica.

Além disso:

- `W/m²` representa uma grandeza de potência por área;
- não é diretamente uma medida de energia em `kWh`;
- os dados utilizados vêm de modelo/reanálise, e não de um sensor instalado no local.

---

# 12. Limitações da regressão

A avaliação possui algumas limitações:

1. Foi utilizado apenas um trimestre de dados.
2. O período de teste corresponde às últimas semanas do conjunto, podendo apresentar condições diferentes das observadas durante o treinamento.
3. Não foram realizados ajustes de hiperparâmetros.
4. A radiação utilizada é proveniente de modelo/reanálise.
5. A previsão representa radiação horizontal, não a radiação incidente diretamente sobre um painel com determinada orientação e inclinação.

---

# 13. Conclusões

## Tarefa 1

Neste conjunto de dados, os três modelos apresentaram desempenhos diferentes.

A execução registrada encontrou:

- Regressão Logística: **F1 macro = 0.820**
- KNN: **F1 macro = 0.965**
- Random Forest: **F1 macro = 0.975**

O Random Forest apresentou **F1 macro de 0.975 e accuracy de 0.976** no conjunto de teste.

Os erros mais frequentes foram entre as classes Solar, Hidráulica e Eólica, especialmente em empreendimentos com características de potência e localização semelhantes.

## Tarefa 2

Na previsão da radiação solar:

- Regressão Linear: **MAE = 145.205 W/m²**, R² = 0.360
- Random Forest: **MAE = 66.252 W/m²**, R² = 0.845
- Gradient Boosting: **MAE = 67.082 W/m²**, R² = 0.841

A variável `hora` teve a maior importância no Random Forest, com **0.488**, reforçando a presença de um forte padrão diário na radiação solar.

Os resultados devem ser interpretados dentro das limitações do conjunto de dados e da metodologia utilizada.

---

# 14. Tecnologias utilizadas

- Python
- Pandas
- NumPy
- Matplotlib
- Scikit-learn
- ANEEL/SIGA
- Open-Meteo
- Jupyter Notebook / Google Colab

### Principais algoritmos

**Classificação**
- Logistic Regression
- K-Nearest Neighbors (KNN)
- Random Forest Classifier

**Regressão**
- Linear Regression
- Random Forest Regressor
- Gradient Boosting Regressor

---

# 15. Como executar

1. Abra o arquivo `.ipynb` no Jupyter Notebook ou Google Colab.
2. Certifique-se de que o ambiente possui as bibliotecas utilizadas instaladas.
3. Execute as células na ordem.
4. As APIs públicas serão consultadas automaticamente.
5. O notebook irá gerar:
   - `aneel_classificacao_orange.csv`
   - `meteo_regressao_orange.csv`
6. As tabelas, gráficos e métricas serão produzidos durante a execução.

No Google Colab, uma forma prática é utilizar:

**Kernel → Restart & Run All**

ou executar todas as células em sequência.

---

## 16. Resumo dos resultados

| Tarefa | Modelo | Principal métrica | Resultado |
|---|---|---:|---:|
| Classificação ANEEL | Random Forest | F1 macro | **0.975** |
| Classificação ANEEL | Random Forest | Accuracy | **0.976** |
| Regressão Open-Meteo | Random Forest | MAE | **66.252 W/m²** |
| Regressão Open-Meteo | Random Forest | R² | **0.845** |

> Os números acima correspondem às saídas registradas no notebook fornecido.
