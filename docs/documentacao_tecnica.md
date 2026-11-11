# Documentação técnica: Segmentação de Clientes com K-Means

Este documento registra as decisões tomadas ao longo do projeto e a justificativa de cada uma. O código completo está em `notebooks/segmentacao_clientes_kmeans.ipynb`.

## 1. Contexto e objetivo

Base `Mall_Customers.csv` (200 clientes de um shopping): gênero, idade, renda anual e pontuação de gastos. O objetivo é identificar segmentos de clientes com comportamento de compra semelhante para orientar ações de marketing. Como não há variável alvo, o problema é de **aprendizado não supervisionado** (clusterização).

## 2. Análise exploratória

| O que foi analisado | Resultado | Decisão |
|:--|:--|:--|
| Valores ausentes | Nenhum | Nenhum tratamento necessário |
| Distribuição por gênero | 112 mulheres e 88 homens | Base equilibrada; gênero usado só para descrever os segmentos |
| Renda anual | 15 a 137 mil dólares, mediana 61,5 | Padronizar antes do modelo |
| Pontuação de gastos | 1 a 99, mediana 50 | Padronizar antes do modelo |
| Pairplot | Grupos visíveis no gráfico renda x score; relação entre idade e score (scores altos concentrados até 40 anos) | Segmentar por renda e score |

## 3. Pré-processamento

**Seleção de variáveis.** O modelo usa apenas `Annual Income (k$)` e `Spending Score (1-100)`. Motivos:

1. São as duas variáveis que descrevem diretamente o comportamento de compra.
2. A análise exploratória mostrou estrutura de grupos justamente nesse par.
3. Com duas variáveis, os clusters são visualizáveis e explicáveis para a área de negócio, o que é essencial em segmentação de marketing.

Idade e gênero foram mantidos fora do treinamento e usados depois para caracterizar cada segmento.

**Padronização.** `StandardScaler` (média 0, desvio padrão 1) nas duas variáveis. O K-Means minimiza distâncias euclidianas; sem padronização, a renda (amplitude 122) dominaria o score (amplitude 98) na formação dos grupos. Os centróides são convertidos de volta à escala original com `inverse_transform` para interpretação.

## 4. Escolha do número de clusters

A escolha de k combinou dois critérios, calculados para k de 2 a 10 com `n_init=10` e `random_state=42`:

| k | Inércia | Silhouette |
|--:|--:|--:|
| 2 | 269,7 | 0,321 |
| 3 | 157,7 | 0,467 |
| 4 | 108,9 | 0,494 |
| 5 | 65,6 | 0,555 |
| 6 | 55,1 | 0,540 |
| 7 | 44,9 | 0,528 |
| 8 | 37,2 | 0,455 |
| 9 | 32,4 | 0,457 |
| 10 | 30,0 | 0,443 |

- **Método do cotovelo**: a inércia cai 40 % de k=4 para k=5 (108,9 para 65,6) e apenas 16 % de k=5 para k=6. O cotovelo está em 5.
- **Silhouette score**: máximo em k=5 (0,555). Valores acima de 0,5 indicam estrutura de clusters razoável a forte.

Os dois critérios concordam, o que dá segurança à escolha. Mesmo assim, os modelos com 3 e 4 clusters foram mantidos no notebook para mostrar como a segmentação evolui e por que 5 é o ponto certo.

## 5. Modelo

`KMeans(n_clusters=5, n_init=10, random_state=42)` sobre as duas variáveis padronizadas.

- `n_init=10`: o algoritmo roda 10 vezes com inicializações diferentes e mantém a melhor, reduzindo o risco de um mínimo local ruim.
- `random_state=42`: reprodutibilidade.

Centróides na escala original (renda em k$, score):

| Cluster | Renda | Score | Segmento |
|--:|--:|--:|:--|
| 0 | 55,3 | 49,5 | Renda média, gasto médio |
| 1 | 86,5 | 82,1 | Renda alta, gasto alto |
| 2 | 25,7 | 79,4 | Renda baixa, gasto alto |
| 3 | 88,2 | 17,1 | Renda alta, gasto baixo |
| 4 | 26,3 | 20,9 | Renda baixa, gasto baixo |

Os nomes dos segmentos foram atribuídos por regra sobre o centróide: renda alta acima de 70, baixa abaixo de 40; gasto alto acima de 60, baixo abaixo de 40.

## 6. Evolução de 3 para 5 clusters

| k | O que o modelo separa |
|--:|:--|
| 3 | Um grupo de renda até 70 mil e dois grupos de renda alta, divididos pelo score (alto ou baixo, nunca médio) |
| 4 | Os grupos de renda alta se mantêm; o grupo de renda até 70 mil se divide entre score baixo e score médio/alto |
| 5 | O grupo central (renda e score médios) se isola e os quatro cantos do gráfico viram grupos próprios |

## 7. Perfil dos segmentos e recomendações

| Segmento | Clientes | Idade média | Mulheres | Leitura | Ação |
|:--|--:|--:|--:|:--|:--|
| Renda alta, gasto alto | 39 | 32,7 | 54 % | Clientes mais valiosos | Programa de fidelidade, lançamentos, atendimento prioritário |
| Renda alta, gasto baixo | 35 | 41,1 | 46 % | Maior oportunidade: têm renda e gastam pouco | Produtos exclusivos, experiências de valor percebido, reativação |
| Renda média, gasto médio | 81 | 42,7 | 59 % | Maior grupo, comportamento regular | Promoções sazonais e cross-sell para elevar o ticket médio |
| Renda baixa, gasto alto | 22 | 25,3 | 59 % | Jovens que gastam acima da renda | Parcelamento, cashback, produtos de entrada; monitorar inadimplência |
| Renda baixa, gasto baixo | 23 | 45,2 | 61 % | Menor valor atual | Ofertas de estoque antigo, baixo investimento em marketing |

## 8. Limitações

1. Base pequena (200 clientes) e de origem didática; os segmentos precisam ser validados com dados reais de campanhas.
2. A pontuação de gastos é calculada pelo shopping e sua fórmula não é conhecida.
3. O modelo usa duas variáveis; idade e gênero só entram na descrição dos grupos.
4. O K-Means assume clusters aproximadamente esféricos e de tamanho semelhante, o que vale para esta base mas não para qualquer uma.

## 9. Próximos passos

1. Incluir idade (e frequência de visitas, se disponível) no modelo e comparar o silhouette.
2. Testar clusterização hierárquica e DBSCAN como contraponto ao K-Means.
3. Medir o resultado das ações por segmento (taxa de resposta, ticket médio) e reavaliar os grupos periodicamente.

## 10. Ambiente

Python 3, pandas, NumPy, scikit-learn, Matplotlib, Seaborn. Versões em `requirements.txt`.
