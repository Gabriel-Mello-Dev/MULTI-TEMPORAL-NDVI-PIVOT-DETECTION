# MULTI-TEMPORAL NDVI PIVOT DETECTION

Detecção de possíveis pivôs centrais utilizando **Google Earth Engine**, **Sentinel 2**, **NDVI**, **GLCM** e análise geométrica.

Projeto desenvolvido durante uma pesquisa de **PIBIC Júnior da UNESP Ourinhos**.
<br>

<p align="center"> <img src="QrcodeGoogleEarthEngine.png" alt="QR Code do Google Earth Engine" width="180"> </p>

# Como funciona?

A ideia é simples:

```text
        IMAGENS DE SATÉLITE
                 │
                 ▼
              NDVI
                 │
                 ▼
             TEXTURA
                 │
                 ▼
          JANEIRO / ABRIL / JULHO
                 │
                 ▼
        DETECÇÃO DE MUDANÇAS
                 │
                 ▼
              BORDAS
                 │
                 ▼
             POLÍGONOS
                 │
                 ▼
         FORMATO + TAMANHO
                 │
                 ▼
        POSSÍVEIS PIVÔS
```

O algoritmo não procura simplesmente por círculos.

Ele procura regiões que apresentem uma combinação de características compatíveis com áreas irrigadas por pivô central.

---

# 1. Área de estudo

Primeiro, definimos onde o algoritmo vai trabalhar.

```text
                    ÁREA DE ESTUDO

                         ●
                    ponto central
                         │
                  ┌──────┴──────┐
                  │              │
                  │     AOI      │
                  │              │
                  └──────────────┘

                    raio: 4,5 km
```

No código:

```javascript
var center = ee.Geometry.Point([
  -49.469044930377024,
  -23.06794929792613
]);

var AOI = center.buffer(4500);
```

`AOI` significa **Area of Interest**.

Tudo que o algoritmo faz fica limitado a essa região.

---

# 2. Imagens Sentinel 2

O primeiro dado utilizado é o Sentinel 2.

```text
Sentinel 2
    │
    ▼
┌───────────────┐
│ Imagem        │
│ de satélite   │
└───────────────┘
```

O código seleciona imagens:

```javascript
ee.ImageCollection(
  'COPERNICUS/S2_SR_HARMONIZED'
)
```

Depois são aplicados filtros:

```text
Sentinel 2
    │
    ├── Área
    │
    ├── Data
    │
    └── Nuvens
    │
    ▼
Imagens utilizáveis
```

---

# 3. Remoção de nuvens

Imagens com nuvens podem atrapalhar a análise.

Por isso:

```text
ANTES

┌──────────────────┐
│ █████ ☁☁ ██████ │
│ ████████████████ │
│ ███ ☁ ██████████ │
└──────────────────┘

        ↓

DEPOIS

┌──────────────────┐
│ █████    ███████ │
│ ████████████████ │
│ ███   ██████████ │
└──────────────────┘
```

A banda `QA60` é utilizada para criar a máscara.

```javascript
var qa = img.select('QA60');

var mask = qa.bitwiseAnd(1 << 10).eq(0)
  .and(
    qa.bitwiseAnd(1 << 11).eq(0)
  );
```

---

# 4. Composição da imagem

Existem várias imagens dentro de cada período.

O algoritmo junta essas imagens usando a mediana:

```text
Imagem 1 ─┐
Imagem 2 ─┤
Imagem 3 ─┼──► MEDIANA ──► Imagem final
Imagem 4 ─┤
Imagem 5 ─┘
```

No código:

```javascript
.median()
```

Isso gera uma imagem representativa de cada período.

---

# 5. NDVI

Agora o algoritmo transforma a imagem em uma informação sobre vegetação.

```text
Sentinel 2
    │
    ▼
B8 + B4
    │
    ▼
  NDVI
```

A fórmula é:

```text
        B8 − B4
NDVI = ─────────
        B8 + B4
```

No Sentinel 2:

```text
B8 = Infravermelho próximo

B4 = Vermelho
```

No código:

```javascript
var ndvi = s2
  .normalizedDifference(['B8', 'B4'])
  .rename('NDVI');
```

---

# 6. Por que não usar somente NDVI?

Porque diferentes tipos de vegetação podem apresentar valores semelhantes.

```text
             NDVI alto
                 │
        ┌────────┴────────┐
        │                 │
     Floresta         Agricultura
```

Então o algoritmo adiciona outra informação:

```text
NDVI
 +
TEXTURA
```

---

# 7. Textura GLCM

A textura analisa como os pixels estão organizados espacialmente.

```text
NDVI

███ ███
██ █ ██
███ ███
██ █ ██
```

O objetivo é analisar padrões, e não apenas o valor individual de cada pixel.

O código utiliza:

```javascript
.glcmTexture({
  size: 2
})
```

E seleciona:

```javascript
.select('NDVI_contrast');
```

Resultado:

```text
NDVI
 │
 ▼
GLCM
 │
 ▼
CONTRASTE
 │
 ▼
PADRÃO ESPACIAL
```

---

# 8. Três períodos

A análise acontece em três momentos diferentes de 2024.

```text
        2024

┌──────────┬──────────┬──────────┐
│ Janeiro  │  Abril   │  Julho   │
└──────────┴──────────┴──────────┘
     │          │          │
     ▼          ▼          ▼
    T1         T2         T3
```

No código:

```javascript
var p1 = ndviTexture(
  '2024-01-01',
  '2024-01-30'
);

var p2 = ndviTexture(
  '2024-04-01',
  '2024-04-30'
);

var p3 = ndviTexture(
  '2024-07-01',
  '2024-07-31'
);
```

Essa é a parte **MULTI-TEMPORAL** do projeto.

---

# 9. Comparando os períodos

Agora o algoritmo pergunta:

```text
A textura mudou entre Janeiro e Abril?

A textura mudou entre Abril e Julho?
```

Visualmente:

```text
Janeiro          Abril           Julho

████████         ████████        ████████
████████    →    ███░░███   →    ██░░░███
████████         ██░░░███        █░░░░░██
████████         ████████        ██░░░███
```

O código:

```javascript
var change = t1.subtract(t2).abs()
  .add(
    t2.subtract(t3).abs()
  )
  .gt(100);
```

Simplificando:

```text
Diferença 1
     +
Diferença 2
     │
     ▼
  Mudança
     │
     ▼
  > 100?
     │
   ┌─┴─┐
  SIM  NÃO
   │    │
   ▼    ▼
  SIM   0
```

O resultado é uma camada de **detecção de mudanças**.

---

# 10. Procurando bordas

Depois da detecção de mudanças, o algoritmo procura bordas.

```text
Change Detection
       │
       ▼
      Canny
       │
       ▼
     Bordas
```

No código:

```javascript
var edges = ee.Algorithms.CannyEdgeDetector({
  image: change,
  threshold: 0.7,
  sigma: 1
});
```

A ideia é transformar:

```text
REGIÃO

████████████
████░░░░████
███░░░░░░███
██░░░░░░░░██
██░░░░░░░░██
███░░░░░░███
████░░░░████
████████████
```

em algo próximo de:

```text
   ███████
 ██       ██
█           █
█           █
 ██       ██
   ███████
```

---

# 11. Fechando regiões

As bordas podem apresentar pequenos espaços.

Por isso são aplicadas operações morfológicas:

```text
Borda quebrada

████   ███
   ████
███    ███

       ↓

Operações morfológicas

       ↓

Borda mais conectada

████████████
██        ██
██        ██
████████████
```

No código:

```javascript
.focal_max(5)
.focal_min(3)
```

---

# 12. Transformando em polígonos

Depois disso, as regiões são convertidas de raster para vetor.

```text
RASTER

████████
██░░░░██
█░░░░░░█
██░░░░██
████████

       ↓

VECTOR

┌─────────┐
│         │
│   área  │
│         │
└─────────┘
```

No código:

```javascript
var vectors = inverted.reduceToVectors({
  geometry: AOI,
  scale: 10,
  geometryType: 'polygon',
  eightConnected: true,
  maxPixels: 1e9
});
```

Agora cada região pode ser analisada individualmente.

---

# 13. Analisando a forma

Para cada polígono, o algoritmo calcula:

```text
             POLÍGONO
                │
        ┌───────┼───────┐
        │       │       │
        ▼       ▼       ▼
      Área  Circularidade  RectRatio
```

### Área

```javascript
var area = geom.area({
  maxError: 1
});
```

Serve para eliminar regiões muito pequenas ou muito grandes.

### Circularidade

```text
       4π × Área
C = ───────────────
      Perímetro²
```

Quanto mais próxima de um círculo, maior tende a ser a circularidade.

### RectRatio

Compara a área do objeto com seu retângulo envolvente.

```text
┌──────────────────┐
│                  │
│      ██████      │
│    ██████████    │
│      ██████      │
│                  │
└──────────────────┘
```

Isso fornece outra característica para analisar o formato.

---

# 14. Filtro final

Depois de calcular as características, os polígonos passam pelos filtros:

```text
                 POLÍGONOS
                     │
                     ▼
              Área > 50.000
                     │
                     ▼
            Área < 2.000.000
                     │
                     ▼
          Circularidade > 0,2
                     │
                     ▼
             RectRatio < 0,85
                     │
                     ▼
            POSSÍVEIS PIVÔS
```

Código:

```javascript
var pivots = metrics
  .filter(ee.Filter.gt('area', 50000))
  .filter(ee.Filter.lt('area', 2000000))
  .filter(ee.Filter.gt('circularity', 0.2))
  .filter(ee.Filter.lt('rectRatio', 0.85));
```

Importante:

**Pivôs detectados pelo algoritmo são candidatos.**

A classificação automática não significa que todo resultado seja necessariamente um pivô real.

---

# 15. Resultado

Por fim, os polígonos selecionados são transformados novamente em uma imagem:

```javascript
var pivotMask = ee.Image()
  .byte()
  .paint(pivots, 1)
  .selfMask();
```

E exibidos no mapa:

```javascript
Map.addLayer(
  pivotMask,
  {
    palette: 'red'
  },
  'Pivos detectados'
);
```

Resultado conceitual:

```text
        IMAGEM DE SATÉLITE

┌──────────────────────────┐
│                          │
│       ◯                  │
│                          │
│              ◯           │
│                          │
│   ◯                      │
│                          │
└──────────────────────────┘

               ↓

        DETECÇÃO

┌──────────────────────────┐
│                          │
│       🔴                 │
│                          │
│              🔴          │
│                          │
│   🔴                     │
│                          │
└──────────────────────────┘
```

---

# 16. Algoritmo em uma linha

```text
Sentinel 2
   ↓
NDVI
   ↓
GLCM
   ↓
Janeiro + Abril + Julho
   ↓
Mudanças
   ↓
Canny
   ↓
Morfologia
   ↓
Polígonos
   ↓
Geometria
   ↓
POSSÍVEIS PIVÔS
```

---

# 17. Por que "Multi Temporal"?

O nome vem da utilização de diferentes períodos para analisar a mesma região.

```text
                 MESMA ÁREA

        ┌─────────┬─────────┬─────────┐
        │ Janeiro │  Abril  │  Julho  │
        └─────────┴─────────┴─────────┘
             │         │         │
             ▼         ▼         ▼
            NDVI      NDVI      NDVI
             │         │         │
             ▼         ▼         ▼
           GLCM      GLCM      GLCM
             └─────────┼─────────┘
                       ▼
                 COMPARAÇÃO
                       │
                       ▼
                  DETECÇÃO
```

Em vez de analisar uma fotografia isolada, o algoritmo analisa uma sequência de observações.

---

# 18. Estrutura do código

```text
MULTI-TEMPORAL NDVI PIVOT DETECTION
│
├── Área de estudo
│
├── Função NDVI + textura
│
├── Período 1
│   └── Janeiro
│
├── Período 2
│   └── Abril
│
├── Período 3
│   └── Julho
│
├── Texturas
│
├── Change Detection
│
├── Canny
│
├── Operações morfológicas
│
├── Vetorização
│
├── Métricas
│   ├── Área
│   ├── Circularidade
│   └── RectRatio
│
├── Filtros
│
└── Pivôs detectados
```

---

# 19. Resultados da pesquisa

Na área analisada durante a pesquisa:

```text
Método desenvolvido

59 pivôs detectados
```

Comparação utilizada na pesquisa:

```text
Método desenvolvido     59
MapBiomas                41
```

A comparação foi utilizada para analisar as diferenças entre as abordagens de detecção.

---

# 20. Limitações

Durante o desenvolvimento, foram encontradas limitações relacionadas ao processamento de formas circulares diretamente no Google Earth Engine.

```text
Detecção direta de círculos
          │
          ▼
Alto custo computacional
          │
          ▼
Limite de memória
          │
          ▼
Erro de processamento
```

A estratégia foi então modificada para:

```text
NDVI
 +
GLCM
 +
Análise temporal
 +
Bordas
 +
Geometria
```

Essa abordagem permitiu continuar o desenvolvimento dentro das limitações encontradas.

---

# 21. Tecnologias utilizadas

```text
Google Earth Engine
        │
        ├── JavaScript
        ├── Sentinel 2
        ├── NDVI
        ├── GLCM
        ├── Canny Edge Detector
        ├── Operações morfológicas
        └── Vetorização
```

Também foram utilizados dados do **MapBiomas** para comparação dos resultados e a plataforma **Roboflow** como suporte à criação de um dataset de pivôs centrais.

---

# 22. Contexto científico

O projeto está relacionado ao uso de geotecnologias para análise de áreas agrícolas e ao monitoramento de recursos hídricos.

A pesquisa está associada ao **ODS 6 da ONU**, relacionado à água potável e saneamento, devido à relação entre irrigação agrícola e utilização de recursos hídricos.

---

# 23. Continuidade

Como possibilidade de continuidade da pesquisa, está prevista a criação de um aplicativo mobile para divulgação dos resultados obtidos pelo algoritmo.

```text
Google Earth Engine
        │
        ▼
Detecção de pivôs
        │
        ▼
Dados geográficos
        │
        ▼
Aplicativo mobile
        │
        ▼
Visualização dos resultados
```

---

## Autor

**Gabriel de Oliveira Mello**

Pesquisa desenvolvida na **UNESP Ourinhos** durante o programa **PIBIC Júnior**.

**Orientadora:** Edinéia Aparecida dos Santos Galvanin
