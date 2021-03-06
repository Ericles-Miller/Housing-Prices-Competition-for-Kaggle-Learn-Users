# Intermediate Machine Learning

## Missing Values

**Neste tutorial, você aprenderá três abordagens para lidar com valores ausentes.** Em seguida, você comparará a eficácia dessas abordagens em um conjunto de dados do mundo real.

## Introdução

**Há muitas maneiras pelas quais os dados podem acabar com valores ausentes**. **Por exemplo, Uma casa de 2 quartos não incluirá um valor para o tamanho de um terceiro quarto.**
Um respondente da pesquisa pode optar por não compartilhar sua renda. A maioria das bibliotecas de aprendizado de máquina (incluindo scikit-learn) apresenta um erro se você tentar construir um modelo usando dados com valores ausentes. Então você precisará escolher uma das estratégias abaixo.

Uma Opção Simples: Eliminar Colunas com Valores Omissos¶
A opção mais simples é descartar colunas com valores ausentes.

![1](https://user-images.githubusercontent.com/59926475/175111732-f063d582-a5ab-4aad-ac17-d6c04711fc70.png)

A menos que a maioria dos valores nas colunas eliminadas esteja faltando, o modelo perde acesso a muitas informações (potencialmente úteis!) com essa abordagem. Como um exemplo extremo, considere um conjunto de dados com 10.000 linhas, onde uma coluna importante está faltando uma única entrada. Essa abordagem eliminaria totalmente a coluna!
2) Uma opção melhor: imputação
A imputação preenche os valores ausentes com algum número. Por exemplo, podemos **preencher o valor médio ao longo de cada coluna**.

![2](https://user-images.githubusercontent.com/59926475/175111739-1b770759-06b9-49c0-98a0-ec3c0073c8a5.png)


**O valor imputado não será exatamente correto na maioria dos casos, mas geralmente leva a modelos mais precisos do que você obteria descartando a coluna completamente.**

3)  **Uma Extensão à Imputação**
**A imputação é a abordagem padrão e geralmente funciona bem. No entanto, os valores imputados podem estar sistematicamente acima ou abaixo de seus valores reais** (que não foram coletados no conjunto de dados). Ou as linhas com valores ausentes podem ser exclusivas de alguma outra forma. **Nesse caso, seu modelo faria previsões melhores considerando quais valores estavam originalmente ausente**s.
![3](https://user-images.githubusercontent.com/59926475/175113289-6a2c14f1-d9df-4ea1-a2fb-4bcb5f4f3bce.png)

Nesta abordagem, imputamos os valores ausentes, como antes. E, adicionalmente, para cada coluna com entradas faltantes no conjunto de dados original, adicionamos uma nova coluna que mostra a localização das entradas imputadas.

Em alguns casos, isso melhorará significativamente os resultados. Em outros casos, não ajuda em nada.

## Exemplo

No exemplo, trabalharemos com o conjunto de dados Melbourne Housing. Nosso modelo usará informações como o número de quartos e o tamanho do terreno para prever o preço da casa.

Não vamos nos concentrar na etapa de carregamento de dados. Em vez disso, você pode imaginar que está em um ponto em que já tem os dados de treinamento e validação em X_train, X_valid, y_train e y_valid.

```python
import pandas as pd
from sklearn.model_selection import train_test_split

# Load the data
data = pd.read_csv('../input/melbourne-housing-snapshot/melb_data.csv')

# Select target
y = data.Price

# To keep things simple, we'll use only numerical predictors
melb_predictors = data.drop(['Price'], axis=1)
X = melb_predictors.select_dtypes(exclude=['object'])

# Divide data into training and validation subsets
X_train, X_valid, y_train, y_valid = train_test_split(X, y, train_size=0.8, test_size=0.2,
                                                      random_state=0)
```

Definir função para medir a qualidade de cada abordagem
Definimos uma função score_dataset() para comparar diferentes abordagens para lidar com valores ausentes. Esta função relata o erro absoluto médio (MAE) de um modelo de floresta aleatória.

```python
from sklearn.ensemble import RandomForestRegressor
from sklearn.metrics import mean_absolute_error

# Function for comparing different approaches
def score_dataset(X_train, X_valid, y_train, y_valid):
    model = RandomForestRegressor(n_estimators=10, random_state=0)
    model.fit(X_train, y_train)
    preds = model.predict(X_valid)
    return mean_absolute_error(y_valid, preds)
```

Pontuação da Abordagem 1 (Retirar Colunas com Valores Omissos)
Como estamos trabalhando com conjuntos de treinamento e validação, temos o cuidado de descartar as mesmas colunas em ambos os DataFrames.

```python
# Get names of columns with missing values
cols_with_missing = [col for col in X_train.columns
                     if X_train[col].isnull().any()]

# Drop columns in training and validation data
reduced_X_train = X_train.drop(cols_with_missing, axis=1)
reduced_X_valid = X_valid.drop(cols_with_missing, axis=1)

print("MAE from Approach 1 (Drop columns with missing values):")
print(score_dataset(reduced_X_train, reduced_X_valid, y_train, y_valid))
```

<aside>
💡 MAE from Approach 1 (Drop columns with missing values):
183550.22137772635 ### saida

</aside>

Pontuação da Abordagem 2 (Imputação)
Em seguida, usamos SimpleImputer para substituir valores ausentes pelo valor médio ao longo de cada coluna.

Embora seja simples, o preenchimento do valor médio geralmente funciona muito bem (mas isso varia de acordo com o conjunto de dados). Embora os estatísticos tenham experimentado maneiras mais complexas de determinar os valores imputados (como a imputação de regressão, por exemplo), as estratégias complexas normalmente não oferecem nenhum benefício adicional quando você conecta os resultados a modelos sofisticados de aprendizado de máquina.

```python
from sklearn.impute import SimpleImputer

# Imputation
my_imputer = SimpleImputer()
imputed_X_train = pd.DataFrame(my_imputer.fit_transform(X_train))
imputed_X_valid = pd.DataFrame(my_imputer.transform(X_valid))

# Imputation removed column names; put them back
imputed_X_train.columns = X_train.columns
imputed_X_valid.columns = X_valid.columns

print("MAE from Approach 2 (Imputation):")
print(score_dataset(imputed_X_train, imputed_X_valid, y_train, y_valid))
```

```
MAE from Approach 2 (Imputation):
178166.46269899711
```

Vemos que a Abordagem 2 tem MAE menor que a Abordagem 1, então a Abordagem 2 teve um desempenho melhor neste conjunto de dados.

**Pontuação da Abordagem 3** (Uma Extensão à Imputação)
Em seguida, imputamos os valores ausentes, além de acompanhar quais valores foram imputados.

```python
# Make copy to avoid changing original data (when imputing)
X_train_plus = X_train.copy()
X_valid_plus = X_valid.copy()

# Make new columns indicating what will be imputed
for col in cols_with_missing:
    X_train_plus[col + '_was_missing'] = X_train_plus[col].isnull()
    X_valid_plus[col + '_was_missing'] = X_valid_plus[col].isnull()

# Imputation
my_imputer = SimpleImputer()
imputed_X_train_plus = pd.DataFrame(my_imputer.fit_transform(X_train_plus))
imputed_X_valid_plus = pd.DataFrame(my_imputer.transform(X_valid_plus))

# Imputation removed column names; put them back
imputed_X_train_plus.columns = X_train_plus.columns
imputed_X_valid_plus.columns = X_valid_plus.columns

print("MAE from Approach 3 (An Extension to Imputation):")
print(score_dataset(imputed_X_train_plus, imputed_X_valid_plus, y_train, y_valid))
```

```python
MAE from Approach 3 (An Extension to Imputation):
178927.503183954
```

Como podemos ver, a Abordagem 3 teve um desempenho ligeiramente pior que a Abordagem 2.

Então, por que a imputação teve um desempenho melhor do que descartar as colunas?
Os dados de treinamento têm 10.864 linhas e 12 colunas, onde três colunas contêm dados ausentes. Para cada coluna, menos da metade das entradas estão faltando. Assim, descartar as colunas remove muitas informações úteis e, portanto, faz sentido que a imputação tenha um desempenho melhor.

```python
# Shape of training data (num_rows, num_columns)
print(X_train.shape)

# Number of missing values in each column of training data
missing_val_count_by_column = (X_train.isnull().sum())
print(missing_val_count_by_column[missing_val_count_by_column > 0])
```

```python
(10864, 12)
Car               49
BuildingArea    5156
YearBuilt       4307
dtype: int64
```

# **Part 3**

## ****Categorical Variables****

### Introdução

Uma variável categórica aceita apenas um número limitado de valores.

Considere uma pesquisa que pergunta com que frequência você toma café da manhã e oferece quatro opções: "Nunca", "Raramente", "Na maioria dos dias" ou "Todos os dias". Nesse caso, os dados são categóricos, porque as respostas se enquadram em um conjunto fixo de categorias.
Se as pessoas respondessem a uma pesquisa sobre qual marca de carro elas possuíam, as respostas cairiam em categorias como "Honda", "Toyota" e "Ford". Nesse caso, os dados também são categóricos.
Você receberá um erro se tentar conectar essas variáveis à maioria dos modelos de aprendizado de máquina em Python sem pré-processá-las primeiro. Neste tutorial, vamos comparar três abordagens que você pode usar para preparar seus dados categóricos.

### Três abordagens

1. **Eliminar Variáveis Categóricas**
A abordagem mais fácil para lidar com variáveis categóricas é simplesmente removê-las do conjunto de dados. Essa abordagem só funcionará bem se as colunas não contiverem informações úteis.
2. **Codificação Ordinal**
A codificação ordinal atribui cada valor exclusivo a um número inteiro diferente.

![3](https://user-images.githubusercontent.com/59926475/175112203-025fb8d7-c02b-4900-aa5b-4a9c0a0414ab.png)

**Essa abordagem pressupõe uma ordenação das categorias: "Nunca" (0) < "Raramente" (1) < "Na maioria dos dias" (2) < "Todos os dias" (3).**

**Essa suposição faz sentido neste exemplo, porque há uma classificação indiscutível para as categorias**. Nem todas as variáveis categóricas têm uma ordenação clara nos valores, mas nos referimos àquelas que têm como variáveis ordinais. Para modelos baseados em árvore (como árvores de decisão e florestas aleatórias), você pode esperar que a codificação ordinal funcione bem com variáveis ordinais.

**3.Codificação One-Hot**
A codificação one-hot cria novas colunas indicando a presença (ou ausência) de cada valor possível nos dados originais. Para entender isso, vamos trabalhar com um exemplo.

![2222](https://user-images.githubusercontent.com/59926475/175116510-caf38d19-8e77-4153-a3c6-4c3a176fd960.png)

**No conjunto de dados original, "Cor" é uma variável categórica com três categorias: "Vermelho", "Amarelo" e "Verde". A codificação one-hot correspondente contém uma coluna para cada valor possível e uma linha para cada linha no conjunto de dados original. Onde quer que o valor original fosse "Vermelho", colocamos um 1 na coluna "Vermelho"; se o valor original era "Amarelo", colocamos 1 na coluna "Amarelo" e assim por diante**.

Em contraste com a codificação ordinal, a codificação one-hot não assume uma ordenação das categorias. Assim, você pode esperar que essa abordagem funcione particularmente bem se não houver uma ordenação clara nos dados categóricos (por exemplo, "Vermelho" não é nem mais nem menos que "Amarelo"). Referimo-nos a variáveis categóricas sem uma classificação intrínseca como variáveis nominais.

**A codificação one-hot geralmente não funciona bem se a variável categórica assumir um grande número de valores (ou seja, você geralmente não a usará para variáveis com mais de 15 valores diferentes)**.

## Exemplo

Como no tutorial anterior, trabalharemos com o conjunto de dados Melbourne Housing.

Não vamos nos concentrar na etapa de carregamento de dados. Em vez disso, você pode imaginar que está em um ponto em que já tem os dados de treinamento e validação em X_train, X_valid, y_train e y_valid.

```python
import pandas as pd
from sklearn.model_selection import train_test_split

# Read the data
data = pd.read_csv('../input/melbourne-housing-snapshot/melb_data.csv')

# Separate target from predictors
y = data.Price
X = data.drop(['Price'], axis=1)

# Divide data into training and validation subsets
X_train_full, X_valid_full, y_train, y_valid = train_test_split(X, y, train_size=0.8, test_size=0.2,
                                                                random_state=0)

# Drop columns with missing values (simplest approach)
cols_with_missing = [col for col in X_train_full.columns if X_train_full[col].isnull().any()] 
X_train_full.drop(cols_with_missing, axis=1, inplace=True)
X_valid_full.drop(cols_with_missing, axis=1, inplace=True)

# "Cardinality" means the number of unique values in a column
# Select categorical columns with relatively low cardinality (convenient but arbitrary)
low_cardinality_cols = [cname for cname in X_train_full.columns if X_train_full[cname].nunique() < 10 and 
                        X_train_full[cname].dtype == "object"]

# Select numerical columns
numerical_cols = [cname for cname in X_train_full.columns if X_train_full[cname].dtype in ['int64', 'float64']]

# Keep selected columns only
my_cols = low_cardinality_cols + numerical_cols
X_train = X_train_full[my_cols].copy()
X_valid = X_valid_full[my_cols].copy()
```

/opt/conda/lib/python3.7/site-packages/pandas/core/frame.py:4913: SettingWithCopyWarning:
Um valor está tentando ser definido em uma cópia de uma fatia de um DataFrame
Consulte as advertências na documentação: [https://pandas.pydata.org/pandas-docs/stable/user_guide/indexing.html#returning-a-view-versus-a-copy](https://pandas.pydata.org/pandas-docs/stable/user_guide/indexing.html#returning-a-view-versus-a-copy)
erros=erros,

Damos uma olhada nos dados de treinamento com o método head() abaixo.

```python
X_train.head()
```

[Untitled](https://www.notion.so/9671f2f7a2534afc803fd26b8e742404)

Em seguida, obtemos uma lista de todas as variáveis categóricas nos dados de treinamento.

**Fazemos isso verificando o tipo de dados (ou dtype) de cada coluna. O objeto dtype indica que uma coluna tem texto** (há outras coisas que teoricamente poderiam ser, mas isso não é importante para nossos propósitos). Para este conjunto de dados, as colunas com texto indicam variáveis categóricas

```python
# Get list of categorical variables
s = (X_train.dtypes == 'object')
object_cols = list(s[s].index) # verifico se existe alguma string no dataframe

print("Categorical variables:")
print(object_cols)
```

```
Categorical variables:
['Type', 'Method', 'Regionname']
```

Definir função para medir a qualidade de cada abordagem
Definimos uma função score_dataset() para comparar as três abordagens diferentes para lidar com variáveis categóricas. Esta função relata o erro absoluto médio (MAE) de um modelo de floresta aleatória. Em geral, queremos que o MAE seja o mais baixo possível!

```python
from sklearn.ensemble import RandomForestRegressor
from sklearn.metrics import mean_absolute_error

# Function for comparing different approaches
def score_dataset(X_train, X_valid, y_train, y_valid):
    model = RandomForestRegressor(n_estimators=100, random_state=0)
    model.fit(X_train, y_train)
    preds = model.predict(X_valid)
    return mean_absolute_error(y_valid, preds)
```

## **Pontuação da Abordagem 1** (Descartar Variáveis Categóricas)

**Eliminamos as colunas do objeto com o método select_dtypes().**

```python
drop_X_train = X_train.select_dtypes(exclude=['object'])
drop_X_valid = X_valid.select_dtypes(exclude=['object'])

print("MAE from Approach 1 (Drop categorical variables):")
print(score_dataset(drop_X_train, drop_X_valid, y_train, y_valid))
```

```
MAE from Approach 1 (Drop categorical variables):
175703.48185157913
```

## Pontuação da Abordagem(Approach) 2 (Codificação Ordinal)

Scikit-learn tem uma classe OrdinalEncoder que pode ser usada para obter codificações ordinais. **Fazemos um loop sobre as variáveis categóricas e aplicamos o codificador ordinal separadamente a cada coluna.**

```python
from sklearn.preprocessing import OrdinalEncoder

# Make copy to avoid changing original data 
label_X_train = X_train.copy()
label_X_valid = X_valid.copy()

# Apply ordinal encoder to each column with categorical data
ordinal_encoder = OrdinalEncoder()
label_X_train[object_cols] = ordinal_encoder.fit_transform(X_train[object_cols])
label_X_valid[object_cols] = ordinal_encoder.transform(X_valid[object_cols])

print("MAE from Approach 2 (Ordinal Encoding):") 
print(score_dataset(label_X_train, label_X_valid, y_train, y_valid))
```

```
MAE from Approach 2 (Ordinal Encoding):
165936.40548390493
```

**Na célula de código acima, para cada coluna, atribuímos aleatoriamente cada valor exclusivo a um inteiro diferente**. Essa é uma abordagem comum que é mais simples do que fornecer rótulos personalizados; no entanto, podemos esperar um aumento adicional no desempenho se fornecermos rótulos mais bem informados para todas as variáveis ordinais.

## Pontuação da Abordagem 3 (Codificação One-Hot)

**Usamos a classe OneHotEncoder do scikit-learn para obter codificações one-hot. Há vários parâmetros que podem ser usados para personalizar seu comportamento.**

**Definimos handle_unknown='ignore' para evitar erros quando os dados de validação contêm classes que não são representadas nos dados de treinamento e configurar sparse=False garante que as colunas codificadas sejam retornadas como uma matriz numpy (em vez de uma matriz esparsa).**
**Para usar o codificador, fornecemos apenas as colunas categóricas que queremos que sejam codificadas em um hot-hot.** **Por exemplo, para codificar os dados de treinamento, fornecemos X_train[object_cols]. (object_cols na célula de código abaixo é uma lista dos nomes das colunas com dados categóricos e, portanto, X_train[object_cols] contém todos os dados categóricos no conjunto de treinamento.)**

```python
from sklearn.preprocessing import OneHotEncoder

# Apply one-hot encoder to each column with categorical data
OH_encoder = OneHotEncoder(handle_unknown='ignore', sparse=False)
OH_cols_train = pd.DataFrame(OH_encoder.fit_transform(X_train[object_cols]))
OH_cols_valid = pd.DataFrame(OH_encoder.transform(X_valid[object_cols]))

# One-hot encoding removed index; put it back
OH_cols_train.index = X_train.index
OH_cols_valid.index = X_valid.index

# Remove categorical columns (will replace with one-hot encoding)
num_X_train = X_train.drop(object_cols, axis=1)
num_X_valid = X_valid.drop(object_cols, axis=1)

# Add one-hot encoded columns to numerical features
OH_X_train = pd.concat([num_X_train, OH_cols_train], axis=1)
OH_X_valid = pd.concat([num_X_valid, OH_cols_valid], axis=1)

print("MAE from Approach 3 (One-Hot Encoding):") 
print(score_dataset(OH_X_train, OH_X_valid, y_train, y_valid))
```

```
MAE from Approach 3 (One-Hot Encoding):
166089.4893009678
```

Nesse caso, descartar as colunas categóricas (Abordagem 1) teve o pior desempenho, pois teve a maior pontuação no MAE. Quanto às outras duas abordagens, uma vez que as pontuações do MAE retornadas são tão próximas em valor, não parece haver nenhum benefício significativo de uma sobre a outra.

**Em geral, a codificação one-hot (abordagem 3) normalmente terá o melhor desempenho, e a eliminação das colunas categóricas (abordagem 1) normalmente apresenta o pior desempenho, mas varia caso a cas**o.

# **Part 4**

# Pipelines

Neste tutorial, você aprenderá a usar pipelines para limpar seu código de modelagem.

## Introdução

**Os pipelines são uma maneira simples de manter seu código de modelagem e pré-processamento de dados organizado.** Especificamente, um **pipeline agrupa etapas de pré-processamento e modelagem para que você possa usar todo o pacote configurável como se fosse uma única etapa.**

**Muitos cientistas de dados hackeiam modelos sem pipelines, mas os pipelines têm alguns benefícios importantes.** Esses incluem:

**Código mais limpo**: a contabilização de dados em cada etapa do pré-processamento pode ficar confusa. Com um pipeline, você não precisará acompanhar manualmente seus dados de treinamento e validação em cada etapa.

**Menos Bugs:** Há menos oportunidades de aplicar incorretamente uma etapa ou esquecer uma etapa de pré-processamento.

**Mais fácil de produzir**: pode ser surpreendentemente difícil fazer a transição de um modelo de um protótipo para algo implantável em escala. Não entraremos em muitas preocupações relacionadas aqui, mas os pipelines podem ajudar.

**Mais opções para validação de modelo**: você verá um exemplo no próximo tutorial, que aborda a validação cruzada.

## Exemplo

Como no tutorial anterior, trabalharemos com o conjunto de dados Melbourne Housing.

Não vamos nos concentrar na etapa de carregamento de dados. Em vez disso, você pode imaginar que está em um ponto em que já tem os dados de treinamento e validação em X_train, X_valid, y_train e y_valid.

```python
import pandas as pd
from sklearn.model_selection import train_test_split

# Read the data
data = pd.read_csv('../input/melbourne-housing-snapshot/melb_data.csv')

# Separate target from predictors
y = data.Price
X = data.drop(['Price'], axis=1)

# Divide data into training and validation subsets
X_train_full, X_valid_full, y_train, y_valid = train_test_split(X, y, train_size=0.8, test_size=0.2,
                                                                random_state=0)

# "Cardinality" means the number of unique values in a column
# Select categorical columns with relatively low cardinality (convenient but arbitrary)
categorical_cols = [cname for cname in X_train_full.columns if X_train_full[cname].nunique() < 10 and 
                        X_train_full[cname].dtype == "object"]

# Select numerical columns
numerical_cols = [cname for cname in X_train_full.columns if X_train_full[cname].dtype in ['int64', 'float64']]

# Keep selected columns only
my_cols = categorical_cols + numerical_cols
X_train = X_train_full[my_cols].copy()
X_valid = X_valid_full[my_cols].copy()
```

Damos uma olhada nos dados de treinamento com o método head() abaixo. Observe que os dados contêm dados categóricos e colunas com valores ausentes. Com um pipeline, é fácil lidar com ambos!

```python
X_train.head()
```

[Untitled](https://www.notion.so/ea10e3d5ded44da084cd7e6d4a3b7c7b)

**Construímos o pipeline completo em três etapas.**

## ****Step 1: Define Preprocessing Steps[¶](https://www.kaggle.com/code/alexisbcook/pipelines#Step-1:-Define-Preprocessing-Steps)**

**Semelhante a como um pipeline agrupa as etapas de pré-processamento e modelagem, usamos a classe ColumnTransformer para agrupar diferentes etapas de pré-processamento**. O código abaixo: **imputa valores ausentes em dados numéricos, e imputa valores ausentes e aplica uma codificação one-hot a dados categóricos.**

```python
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline
from sklearn.impute import SimpleImputer
from sklearn.preprocessing import OneHotEncoder

# Preprocessing for numerical data
numerical_transformer = SimpleImputer(strategy='constant')

# Preprocessing for categorical data
categorical_transformer = Pipeline(steps=[
    ('imputer', SimpleImputer(strategy ='most_frequent')),
    ('onehot', OneHotEncoder(handle_unknown  ='ignore'))
])

# Bundle preprocessing for numerical and categorical data
preprocessor = ColumnTransformer(
    transformers=[
        ('num', numerical_transformer, numerical_cols),
        ('cat', categorical_transformer, categorical_cols)
    ])
```

# Etapa 2:definir o modelo

**Em seguida, definimos um modelo de floresta aleatória com a classe RandomForestRegressor familiar.**

```python
from sklearn.ensemble import RandomForestRegressor

model = RandomForestRegressor(n_estimators=100, random_state=0)
```

## **Etapa 3: criar e avaliar o pipeline**

Por fim, usamos a classe Pipeline para definir um pipeline que agrupa as etapas de pré-processamento e modelagem. **Existem algumas coisas importantes a serem observadas:**
**Com o pipeline, pré-processamos os dados de treinamento e ajustamos o modelo em uma única linha de código.** (**Em contraste, sem um pipeline, temos que fazer imputação, codificação one-hot e treinamento de modelo em etapas separadas. Isso se torna especialmente confuso se tivermos que lidar com variáveis numéricas e categóricas!**)
**Com o pipeline, fornecemos os recursos não processados em X_valid ao comando predict(), e o pipeline pré-processa automaticamente os recursos antes de gerar previsões.** (No entanto, sem **um pipeline, temos que nos lembrar de pré-processar os dados de validação antes de fazer previsões.)**

```python
from sklearn.metrics import mean_absolute_error

# Bundle preprocessing and modeling code in a pipeline
my_pipeline = Pipeline(steps=[('preprocessor', preprocessor),('model', model)])

# Preprocessing of training data, fit model 
my_pipeline.fit(X_train, y_train)

# Preprocessing of validation data, get predictions
preds = my_pipeline.predict(X_valid)

# Evaluate the model
score = mean_absolute_error(y_valid, preds)
print('MAE:', score)
```

`MAE: 160679.18917034855`  valor do score 

# Part 5

## Croos-Validation

### Introdução

O aprendizado de máquina é um processo iterativo.

Você enfrentará escolhas sobre quais variáveis preditivas usar, quais tipos de modelos usar, quais argumentos fornecer a esses modelos etc. Até agora, você fez essas escolhas de maneira orientada por dados, medindo a qualidade do modelo com uma validação ( ou holdout) definido.

Mas há algumas desvantagens nessa abordagem. Para ver isso, imagine que você tenha um conjunto de dados com 5.000 linhas. **Normalmente, você manterá cerca de 20% dos dados como um conjunto de dados de validação ou 1.000 linhas. Mas isso deixa alguma chance aleatória na determinação das pontuações do modelo. Ou seja, um modelo pode funcionar bem em um conjunto de 1.000 linhas, mesmo que seja impreciso em 1.000 linhas diferentes.**

Em um extremo, você pode imaginar ter apenas 1 linha de dados no conjunto de validação. Se você comparar modelos alternativos, qual deles faz as melhores previsões em um único ponto de dados será principalmente uma questão de sorte!

Em geral, quanto maior o conjunto de validação, menos aleatoriedade (também conhecido como "ruído") existe em nossa medida de qualidade do modelo e mais confiável ela será. Infelizmente, só podemos obter um grande conjunto de validação removendo linhas de nossos dados de treinamento, e conjuntos de dados de treinamento menores significam modelos piores!

## O que é validação cruzada?

Na validação cruzada, executamos nosso processo de modelagem em diferentes subconjuntos de dados para obter várias medidas de qualidade do modelo.

Por exemplo, poderíamos começar dividindo os dados em 5 partes, cada uma com 20% do conjunto de dados completo. Nesse caso, dizemos que dividimos os dados em 5 "dobras".

![5](https://user-images.githubusercontent.com/59926475/175116790-02c0b655-6ef1-4dd1-a2f1-e8638e371c51.png)
    
Em seguida, executamos um experimento para cada dobra:

**No Experimento 1, usamos a primeira dobra como um conjunto de validação (ou validação) e todo o resto como dados de treinamento. Isso nos dá uma medida da qualidade do modelo com base em um conjunto de 20% de validação.
No Experimento 2, mantemos os dados da segunda dobra (e usamos tudo, exceto a segunda dobra, para treinar o modelo). O conjunto de validação é então usado para obter uma segunda estimativa da qualidade do modelo.
Repetimos esse processo, usando cada dobra uma vez como o conjunto de retenção. Juntando isso, 100% dos dados são usados como validação em algum momento e acabamos com uma medida de qualidade do modelo baseada em todas as linhas do conjunto de dados (mesmo que não usemos todas as linhas simultaneamente) .**

## Quando você deve usar a validação cruzada?

A validação cruzada fornece uma medida mais precisa da qualidade do modelo, o que é especialmente importante se você estiver tomando muitas decisões de modelagem. No entanto, pode demorar mais para ser executado, pois estima vários modelos (um para cada dobra).

Então, dadas essas compensações, quando você deve usar cada abordagem?

**Para conjuntos de dados pequenos, onde a carga computacional extra não é um grande problema, você deve executar a validação cruzada.
Para conjuntos de dados maiores, um único conjunto de validação é suficiente. Seu código será executado mais rápido e você pode ter dados suficientes para que haja pouca necessidade de reutilizar alguns deles para validação.
Não há um limite simples para o que constitui um conjunto de dados grande versus pequeno. Mas se o seu modelo demorar alguns minutos ou menos para ser executado, provavelmente vale a pena mudar para a validação cruzada.**

Como alternativa, você pode executar a validação cruzada e ver se as pontuações de cada experiência parecem próximas. Se cada experimento produzir os mesmos resultados, um único conjunto de validação provavelmente será suficiente.

Exemplo
Trabalharemos com os mesmos dados do tutorial anterior. Carregamos os dados de entrada em X e os dados de saída em y.

```python
import pandas as pd

# Read the data
data = pd.read_csv('../input/melbourne-housing-snapshot/melb_data.csv')

# Select subset of predictors
cols_to_use = ['Rooms', 'Distance', 'Landsize', 'BuildingArea', 'YearBuilt']
X = data[cols_to_use]

# Select target
y = data.Price
```

Em seguida, definimos um **pipeline que usa um imputador para preencher os valores ausentes e um modelo de floresta aleatória para fazer previsões.**

Embora seja possível fazer validação cruzada sem pipelines, é bastante difícil! Usar um pipeline tornará o código notavelmente direto.

```python
from sklearn.ensemble import RandomForestRegressor
from sklearn.pipeline import Pipeline
from sklearn.impute import SimpleImputer

my_pipeline = Pipeline(steps=[('preprocessor', SimpleImputer()),
                        ('model', RandomForestRegressor(n_estimators=50,random_state=0))
                        ])
```

Obtemos as pontuações de validação cruzada com a função cross_val_score() do scikit-learn. Definimos o número de dobras com o parâmetro cv.

```python
from sklearn.model_selection import cross_val_score

# Multiply by -1 since sklearn calculates *negative* MAE
scores = -1 * cross_val_score(my_pipeline, X, y,cv=5,scoring='neg_mean_absolute_error')

print("MAE scores:\n", scores)
```

```
MAE scores:
 [301628.7893587  303164.4782723  287298.331666   236061.84754543
 260383.45111427]
```

O parâmetro de pontuação escolhe uma medida de qualidade do modelo para relatar: neste caso, escolhemos o erro absoluto médio negativo (MAE). A documentação do scikit-learn mostra uma lista de opções.

É um pouco surpreendente que especifiquemos MAE negativo. O Scikit-learn tem uma convenção onde todas as métricas são definidas para que um número alto seja melhor. O uso de negativos aqui permite que eles sejam consistentes com essa convenção, embora o MAE negativo seja quase inédito em outros lugares.

Normalmente, queremos uma única medida de qualidade do modelo para comparar modelos alternativos. Então, pegamos a média entre os experimentos.

```python
print("Average MAE score (across experiments):")
print(scores.mean())
```

```
Average MAE score (across experiments):
277707.3795913405
```

## Conclusão

O uso da validação cruzada produz uma medida muito melhor da qualidade do modelo, com o benefício adicional de limpar nosso código: observe que não precisamos mais acompanhar conjuntos separados de treinamento e validação. Portanto, especialmente para conjuntos de dados pequenos, é uma boa melhoria!

# Part 6

# XGBoost

**Neste tutorial, você aprenderá como construir e otimizar modelos com aumento de gradiente**. Esse método domina muitas competições do Kaggle e alcança resultados de última geração em uma variedade de conjuntos de dados.

Introdução
Durante grande parte deste curso, você fez previsões com o método de floresta aleatória, que alcança melhor desempenho do que uma única árvore de decisão simplesmente pela média das previsões de muitas árvores de decisão.

**Referimo-nos ao método da floresta aleatória como um "método de conjunto". Por definição, os métodos de conjunto combinam as previsões de vários modelos (por exemplo, várias árvores, no caso de florestas aleatórias).**

## Aumento de gradiente

**O aumento de gradiente é um método que passa por ciclos para adicionar modelos de forma iterativa em um conjunto.**

**Ele começa inicializando o conjunto com um único modelo, cujas previsões podem ser bastante ingênuas.** (Mesmo que suas previsões sejam extremamente imprecisas, adições subsequentes ao conjunto resolverão esses erros.)

Então, iniciamos o ciclo:

**Primeiro, usamos o conjunto atual para gerar previsões para cada observação no conjunto de dados. Para fazer uma previsão, adicionamos as previsões de todos os modelos no conjunto.
Essas previsões são usadas para calcular uma função de perda (como erro quadrático médio, por exemplo).**
**Em seguida, usamos a função de perda para ajustar um novo modelo que será adicionado ao ensemble. Especificamente, determinamos os parâmetros do modelo para que a adição desse novo modelo ao conjunto reduza a perda.** (Nota: O "gradiente" em "aumento de gradiente" refere-se ao fato de que usaremos gradiente descendente na função de perda para determinar os parâmetros neste novo modelo.)
Por fim, adicionamos o novo modelo ao ensemble e ...... repetir!

![6](https://user-images.githubusercontent.com/59926475/175121827-d78be637-55eb-4c38-90ef-b6a3a5cab647.png)![7](https://user-images.githubusercontent.com/59926475/175112217-7aa06d44-2749-47aa-81b8-54215bbb6058.png)
### Exemplo

Começamos carregando os dados de treinamento e validação em X_train, X_valid, y_train e y_valid

```python
import pandas as pd
from sklearn.model_selection import train_test_split

# Read the data
data = pd.read_csv('../input/melbourne-housing-snapshot/melb_data.csv')

# Select subset of predictors
cols_to_use = ['Rooms', 'Distance', 'Landsize', 'BuildingArea', 'YearBuilt']
X = data[cols_to_use]

# Select target
y = data.Price

# Separate data into training and validation sets
X_train, X_valid, y_train, y_valid = train_test_split(X, y)
```

Neste exemplo, você trabalhará com a biblioteca XGBoost. XGBoost significa aumento de gradiente extremo, que é uma implementação de aumento de gradiente com vários recursos adicionais focados em desempenho e velocidade. (O Scikit-learn tem outra versão de aumento de gradiente, mas o XGBoost tem algumas vantagens técnicas.)

Na próxima célula de código, importamos a API scikit-learn para XGBoost (xgboost.XGBRegressor). Isso nos permite construir e ajustar um modelo exatamente como faríamos no scikit-learn. Como você verá na saída, a classe XGBRegressor tem muitos parâmetros ajustáveis -- você aprenderá sobre eles em breve!

```python
from xgboost import XGBRegressor

my_model = XGBRegressor()
my_model.fit(X_train, y_train)
```

```
XGBRegressor(base_score=0.5, booster='gbtree', colsample_bylevel=1,
             colsample_bynode=1, colsample_bytree=1, enable_categorical=False,
             gamma=0, gpu_id=-1, importance_type=None,
             interaction_constraints='', learning_rate=0.300000012,
             max_delta_step=0, max_depth=6, min_child_weight=1, missing=nan,
             monotone_constraints='()', n_estimators=100, n_jobs=4,
             num_parallel_tree=1, predictor='auto', random_state=0, reg_alpha=0,
             reg_lambda=1, scale_pos_weight=1, subsample=1, tree_method='exact',
             validate_parameters=1, verbosity=None)
```

Também fazemos previsões e avaliamos o modelo.

```python
from sklearn.metrics import mean_absolute_error

predictions = my_model.predict(X_valid)
print("Mean Absolute Error: " + str(mean_absolute_error(predictions, y_valid)))
```

`Mean Absolute Error: 239435.01260125183`

## Ajuste de parâmetro

**O XGBoost possui alguns parâmetros que podem afetar drasticamente a precisão e a velocidade do treinamento. Os primeiros parâmetros que você deve entender são:**

### n_estimadores

**n_estimators especifica quantas vezes passar pelo ciclo de modelagem descrito acima. É igual ao número de modelos que incluímos no conjunto.**

**Um valor muito baixo causa subajuste, o que leva a previsões imprecisas nos dados de treinamento e nos dados de teste.
Um valor muito alto causa overfitting, o que causa previsões precisas em dados de treinamento, mas previsões imprecisas em dados de teste (que é o que nos importa).
Os valores típicos variam de 100-1000, embora isso dependa muito do parâmetro learning_rate discutido abaixo.**

Aqui está o código para definir o número de modelos no conjunto:

```python
my_model = XGBRegressor(n_estimators=500)
my_model.fit(X_train, y_train)
```

```
XGBRegressor(base_score=0.5, booster='gbtree', colsample_bylevel=1,
             colsample_bynode=1, colsample_bytree=1, enable_categorical=False,
             gamma=0, gpu_id=-1, importance_type=None,
             interaction_constraints='', learning_rate=0.300000012,
             max_delta_step=0, max_depth=6, min_child_weight=1, missing=nan,
             monotone_constraints='()', n_estimators=500, n_jobs=4,
             num_parallel_tree=1, predictor='auto', random_state=0, reg_alpha=0,
             reg_lambda=1, scale_pos_weight=1, subsample=1, tree_method='exact',
             validate_parameters=1, verbosity=None)
early_stopping_rounds
```

**early_stopping_rounds oferece uma maneira de encontrar automaticamente o valor ideal para n_estimators**. **A interrupção antecipada faz com que o modelo pare de iterar quando a pontuação de validação para de melhorar, mesmo que não estejamos na parada difícil para n_estimators**. **É inteligente definir um valor alto para n_estimators e, em seguida, usar early_stopping_rounds para encontrar o momento ideal para interromper a iteração.**

C**omo a chance aleatória às vezes causa uma única rodada em que as pontuações de validação não melhoram, você precisa especificar um número para quantas rodadas de deterioração direta permitir antes de parar**. **Definir early_stopping_rounds=5** é uma escolha razoável. Nesse caso, paramos após 5 rodadas consecutivas de deterioração das pontuações de validação.

**Ao usar early_stopping_rounds, você também precisa separar alguns dados para calcular as pontuações de validação** - isso é feito definindo o parâmetro eval_set.

Podemos modificar o exemplo acima para incluir a parada antecipada:

```python
my_model = XGBRegressor(n_estimators=500)
my_model.fit(X_train, y_train, 
             early_stopping_rounds=5, 
             eval_set=[(X_valid, y_valid)],
             verbose=False)
```

```
XGBRegressor(base_score=0.5, booster='gbtree', colsample_bylevel=1,
             colsample_bynode=1, colsample_bytree=1, enable_categorical=False,

             gamma=0, gpu_id=-1, importance_type=None,
             interaction_constraints='', learning_rate=0.300000012,
             max_delta_step=0, max_depth=6, min_child_weight=1, missing=nan,
             monotone_constraints='()', n_estimators=500, n_jobs=4,

             num_parallel_tree=1, predictor='auto', random_state=0, reg_alpha=0,
             reg_lambda=1, scale_pos_weight=1, subsample=1, tree_method='exact',
             validate_parameters=1, verbosity=None)
```

Se mais tarde você quiser ajustar um modelo com todos os seus dados, defina n_estimators para qualquer valor que você achou ideal quando executado com interrupção antecipada.

### taxa de Aprendizagem

**Em vez de obter previsões simplesmente somando as previsões de cada modelo de componente, podemos multiplicar as previsões de cada modelo por um pequeno número (conhecido como taxa de aprendizado) antes de adicioná-las.**

**Isso significa que cada árvore que adicionamos ao conjunto nos ajuda menos. Assim, podemos definir um valor mais alto para n_estimators sem overfitting.** Se usarmos a parada antecipada, o número apropriado de árvores será determinado automaticamente.

**Em geral, uma pequena taxa de aprendizado e um grande número de estimadores produzirão modelos XGBoost mais precisos, embora também leve mais tempo para treinar o modelo, pois faz mais iterações ao longo do ciclo. Como padrão, o XGBoost define learning_rate=0.1**.

Modificar o exemplo acima para alterar a taxa de aprendizado gera o seguinte código:

```python
my_model = XGBRegressor(n_estimators=1000, learning_rate=0.05)
my_model.fit(X_train, y_train, 
             early_stopping_rounds=5, 
             eval_set=[(X_valid, y_valid)], 
             verbose=False)
```

```
XGBRegressor(base_score=0.5, booster='gbtree', colsample_bylevel=1,
             colsample_bynode=1, colsample_bytree=1, enable_categorical=False,

             gamma=0, gpu_id=-1, importance_type=None,
             interaction_constraints='', learning_rate=0.05, max_delta_step=0,

             max_depth=6, min_child_weight=1, missing=nan,
             monotone_constraints='()', n_estimators=1000, n_jobs=4,
             num_parallel_tree=1, predictor='auto', random_state=0, reg_alpha=0,

             reg_lambda=1, scale_pos_weight=1, subsample=1, tree_method='exact',

             validate_parameters=1, verbosity=None)
```

n_trabalhos
**Em conjuntos de dados maiores em que o tempo de execução é uma consideração, você pode usar o paralelismo para criar seus modelos mais rapidamente.** **É comum definir o parâmetro n_jobs igual ao número de núcleos em sua máquina**. Em conjuntos de dados menores, isso não ajudará.

O modelo resultante não será melhor, então a micro-otimização para o tempo de adaptação normalmente não passa de uma distração. **Mas é útil em grandes conjuntos de dados onde você gastaria muito tempo esperando durante o comando fit.**

Aqui está o exemplo modificado:

```python
my_model = XGBRegressor(n_estimators=1000, learning_rate=0.05, n_jobs=4)
my_model.fit(X_train, y_train, 
             early_stopping_rounds=5, 
             eval_set=[(X_valid, y_valid)], 
             verbose=False)
```

```
XGBRegressor(base_score=0.5, booster='gbtree', colsample_bylevel=1,
             colsample_bynode=1, colsample_bytree=1, enable_categorical=False,

             gamma=0, gpu_id=-1, importance_type=None,
             interaction_constraints='', learning_rate=0.05, max_delta_step=0,

             max_depth=6, min_child_weight=1, missing=nan,
             monotone_constraints='()', n_estimators=1000, n_jobs=4,
             num_parallel_tree=1, predictor='auto', random_state=0, reg_alpha=0,

             reg_lambda=1, scale_pos_weight=1, subsample=1, tree_method='exact',
             validate_parameters=1, verbosity=None)
```

### Conclusão

O XGBoost é uma biblioteca de software líder para trabalhar com dados tabulares padrão (o tipo de dados que você armazena no Pandas DataFrames, em oposição a tipos de dados mais exóticos, como imagens e vídeos). Com o ajuste cuidadoso dos parâmetros, você pode treinar modelos altamente precisos.

# Part 7

# Data Leakage

Neste tutorial, você aprenderá o que é vazamento de dados e como evitá-lo. Se você não souber como evitá-lo, o vazamento surgirá com frequência e arruinará seus modelos de maneiras sutis e perigosas. Portanto, esse é um dos conceitos mais importantes para a prática de cientistas de dados.

## Introdução

**O vazamento de dados (ou vazamento) ocorre quando seus dados de treinamento contêm informações sobre o destino, mas dados semelhantes não estarão disponíveis quando o modelo for usado para previsão**. Isso leva a um alto desempenho no conjunto de treinamento (e possivelmente até nos dados de validação), mas o modelo terá um desempenho ruim na produção.

**Em outras palavras, o vazamento faz com que um modelo pareça preciso até que você comece a tomar decisões com o modelo, e então o modelo se torna muito impreciso.**

**Existem dois tipos principais de vazamento: vazamento alvo e contaminação de teste de treino.**

## Vazamento alvo

**O vazamento de destino ocorre quando seus preditores incluem dados que não estarão disponíveis no momento em que você fizer previsões.** É importante pensar no vazamento de destino em termos de tempo ou ordem cronológica em que os dados se tornam disponíveis, não apenas se um recurso ajuda a fazer boas previsões.

Um exemplo será útil. Imagine que você queira prever quem ficará doente com pneumonia. As primeiras linhas de seus dados brutos são assim:

[Untitled](https://www.notion.so/f9d517de95f7413e9b38c7b1d32b2a46)

As pessoas tomam medicamentos antibióticos depois de contrair pneumonia para se recuperar. Os dados brutos mostram uma forte relação entre essas colunas, mas take_antibiotic_medicine é frequentemente alterado depois que o valor de got_pneumonia é determinado. Este é o vazamento alvo.

O modelo veria que qualquer pessoa que tivesse um valor False para take_antibiotic_medicine não tinha pneumonia. Como os dados de validação vêm da mesma fonte que os dados de treinamento, o padrão se repetirá na validação e o modelo terá ótimas pontuações de validação (ou validação cruzada).

Mas o modelo será muito impreciso quando implantado posteriormente no mundo real, porque mesmo os pacientes que terão pneumonia ainda não terão recebido antibióticos quando precisarmos fazer previsões sobre sua saúde futura.

Para evitar esse tipo de vazamento de dados, qualquer variável atualizada (ou criada) após a realização do valor de destino deve ser excluída.

![7](https://user-images.githubusercontent.com/59926475/175122135-1802e7c1-db47-44a0-96d1-1cf90f0d1028.png)

### Contaminação de teste de teste

**Um tipo diferente de vazamento ocorre quando você não tem o cuidado de distinguir os dados de treinamento dos dados de validação.**

**Lembre-se de que a validação deve ser uma medida de como o modelo se comporta em dados que ele não considerou antes**. Você pode corromper esse processo de maneiras sutis se os dados de validação afetarem o comportamento de pré-processamento. **Isso às vezes é chamado de contaminação de teste de treino.**

Por exemplo, imagine que você executa o pré-processamento (como ajustar um imputer para valores ausentes) antes de chamar train_test_split(). **O resultado final? Seu modelo pode obter boas pontuações de validação, dando a você grande confiança nele, mas apresentar um desempenho ruim quando você o implanta para tomar decisões.**

Afinal, você incorporou dados da validação ou dados de teste em como você faz previsões, portanto, pode se sair bem nesses dados específicos, mesmo que não possa generalizar para novos dados. **Esse problema se torna ainda mais sutil (e mais perigoso) quando você faz engenharia de recursos mais complexa.**

Se sua validação for baseada em uma simples divisão de teste de treino**, exclua os dados de validação de qualquer tipo de ajuste, incluindo o ajuste de etapas de pré-processamento. Isso é mais fácil se você usar pipelines scikit-learn. Ao usar validação cruzada, é ainda mais importante que você faça seu pré-processamento dentro do pipeline!**

```python
import pandas as pd

# Read the data
data = pd.read_csv('../input/aer-credit-card-data/AER_credit_card_data.csv', 
                   true_values = ['yes'], false_values = ['no'])

# Select target
y = data.card

# Select predictors
X = data.drop(['card'], axis=1)

print("Number of rows in the dataset:", X.shape[0])
X.head()
```

`Number of rows in the dataset: 1319`

![8](https://user-images.githubusercontent.com/59926475/175122206-5beb42fc-15b3-4cc8-9741-a94e3b3cf68c.png)![3](https://user-images.githubusercontent.com/59926475/175113289-6a2c14f1-d9df-4ea1-a2fb-4bcb5f4f3bce.png)
Como este é um conjunto de dados pequeno, usaremos validação cruzada para garantir medidas precisas da qualidade do modelo.

```python
from sklearn.pipeline import make_pipeline
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import cross_val_score

# Since there is no preprocessing, we don't need a pipeline (used anyway as best practice!)
my_pipeline = make_pipeline(RandomForestClassifier(n_estimators=100))
cv_scores = cross_val_score(my_pipeline, X, y, 
                            cv=5,
                            scoring='accuracy')

print("Cross-validation accuracy: %f" % cv_scores.mean()
```

`Cross-validation accuracy: 0.980292`

Com a experiência, você descobrirá que é muito raro encontrar modelos que sejam precisos 98% das vezes. Isso acontece, mas é incomum o suficiente para inspecionarmos os dados mais de perto em busca de vazamento de destino.

- aqui está um resumo dos dados, que você também pode encontrar na guia de dados:
- cartão: 1 se o pedido de cartão de crédito for aceito, 0 se não
- relatórios: Número de grandes relatórios depreciativos
- idade: idade n anos mais duodécimos de um ano
- renda: Renda anual (dividida por 10.000)
- share: Proporção das despesas mensais com cartão de crédito em relação à renda anual
- despesas: despesas médias mensais com cartão de crédito
- proprietário: 1 se possui casa, 0 se aluga
- selfempl: 1 se autônomo, 0 se não dependentes: 1 + número de dependentes
- meses: Meses morando no endereço atual
- majorcards: número dos principais cartões de crédito mantidos
- active: número de contas de crédito ativas

```python
expenditures_cardholders = X.expenditure[y]
expenditures_noncardholders = X.expenditure[~y]

print('Fraction of those who did not receive a card and had no expenditures: %.2f' \
      %((expenditures_noncardholders == 0).mean()))
print('Fraction of those who received a card and had no expenditures: %.2f' \
      %(( expenditures_cardholders == 0).mean()))
```

```
Fraction of those who did not receive a card and had no expenditures: 1.00
Fraction of those who received a card and had no expenditures: 0.02
```

Conforme demonstrado acima, todos que não receberam cartão não tiveram gastos, enquanto apenas 2% dos que receberam cartão não tiveram gastos. Não é surpreendente que nosso modelo pareça ter uma alta precisão. Mas este também parece ser um caso de fuga de metas, onde os gastos provavelmente significam gastos no cartão que solicitaram.

Como a parcela é parcialmente determinada pelas despesas, ela também deve ser excluída. As variáveis active e majorcards são um pouco menos claras, mas pela descrição, parecem preocupantes. Na maioria das situações, é melhor prevenir do que remediar se você não puder rastrear as pessoas que criaram os dados para saber mais.

Executaríamos um modelo sem vazamento de destino da seguinte forma:

```python
# Drop leaky predictors from datasetpotential_leaks = ['expenditure', 'share', 'active', 'majorcards']
X2 =X.drop(potential_leaks, axis=1)

# Evaluate the model with leaky predictors removedcv_scores =cross_val_score(my_pipeline,X2,y,
                            cv=5,
                            scoring='accuracy')

print("Cross-val accuracy:%f" %cv_scores.mean())
```

`Cross-val accuracy: 0.838510`

Essa precisão é um pouco menor, o que pode ser decepcionante. No entanto, podemos esperar que ele esteja certo em cerca de 80% das vezes quando usado em novos aplicativos, enquanto o modelo com vazamento provavelmente faria muito pior do que isso (apesar de sua pontuação aparente mais alta na validação cruzada)

**O vazamento de dados pode ser um erro multimilionário em muitos aplicativos de ciência de dados. A separação cuidadosa dos dados de treinamento e validação pode evitar a contaminação do teste de trem, e os pipelines podem ajudar a implementar essa separação. Da mesma forma, uma combinação de cautela, bom senso e exploração de dados pode ajudar a identificar o vazamento de destino.**
