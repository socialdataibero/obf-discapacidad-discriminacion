# Discriminación y género

Tablero de Observable Framework sobre la situación de las mujeres en México,
con corte transversal de discapacidad. Construido con cuatro encuestas del
INEGI: ENADIS, ENIGH, Censo 2020 (cuestionario ampliado) y ENDIREH.

Social Data Ibero · Universidad Iberoamericana.

## Correr el sitio

```bash
npm install
npm run dev      # servidor local en http://127.0.0.1:3000
npm run build    # sitio estático en dist/
```

El sitio lee un solo archivo de datos ya calculado, `src/data/indicadores.csv`,
así que `npm run dev` funciona sin tener los microdatos en disco.

## Las tres comparaciones

Todo el tablero gira sobre tres pares fijos, definidos en
`src/components/comparacion.js`:

1. Mujeres frente a hombres
2. Mujeres con frente a mujeres sin discapacidad
3. Mujeres frente a hombres con discapacidad

No todas las fuentes sostienen las tres. ENDIREH entrevista solo a mujeres de
15 años o más, así que únicamente admite la segunda; el objeto `FUENTES`
declara qué puede cada encuesta y la interfaz oculta lo que no aplica en vez de
dibujar gráficas vacías.

## Reconstruir los datos

Los microdatos son públicos pero pesan varios gigabytes y **no se versionan**.
Para regenerar `src/data/indicadores.csv` hay que tenerlos en disco y correr
los data loaders de `src/data/dataloader/`.

| Fuente | Ruta esperada | Variable de entorno |
|---|---|---|
| ENADIS 2017 y 2022 | `…/AnalisisSueltos/Enadis/Enadis{año}/Bases/` | `ENADIS_DIR` |
| ENIGH 2020-2024 | `…/AnalisisSueltos/Obindi/enigh/Bases{año}/` | `ENIGH_DIR` |
| Censo 2020 ampliado | `…/JCF/data/raw/censo2020/Personas00.CSV` | `CENSO_PERSONAS` |
| ENDIREH 2021 | `src/data/raw/endireh/2021/` | `ENDIREH_DIR` |

Descarga de ENDIREH (la única que este proyecto trajo desde cero):

```bash
curl -L -o endireh_2021.zip \
  https://www.inegi.org.mx/contenidos/programas/endireh/2021/datosabiertos/conjunto_de_datos_endireh_2021_csv.zip
```

Generación:

```bash
python src/data/dataloader/enadis.csv.py  > /tmp/enadis.csv
python src/data/dataloader/enigh.csv.py   > /tmp/enigh.csv
python src/data/dataloader/censo.csv.py   > /tmp/censo.csv   # ~3.3 GB, usa DuckDB
python src/data/dataloader/endireh.csv.py > /tmp/endireh.csv
```

Cada salida lleva una columna `encuesta`; el archivo final es la concatenación
de las cuatro. Todos los loaders emiten el mismo esquema largo:

```
tema, indicador, anio, sexo, disc, entidad, rango_edad,
num, den, casos, fuente, universo
```

`num` y `den` van expandidos por el factor de la encuesta; `casos` es el
número de registros **sin expandir**, que es con lo que se juzga si la cifra
aguanta. El porcentaje se calcula en el navegador, después de agregar, para no
promediar tasas nunca.

## Composición por edad

La discapacidad se concentra en las edades mayores, y eso produce paradojas de
Simpson en cualquier indicador que también dependa de la edad.

El caso vivo está en violencia sexual (ENDIREH 2021). En el agregado parece
menor entre mujeres con discapacidad (19.0% contra 22.8%); al abrir por rango de
edad se invierte en todos los grupos menores de 60 años:

| Edad | Con discapacidad | Sin discapacidad |
|---|---|---|
| 18-29 | 61.6% | 39.1% |
| 30-44 | 34.6% | 24.0% |
| 45-59 | 19.0% | 13.2% |
| 60+ | 5.3% | 5.6% |

Por eso la página de violencia abre desglosada por edad y el filtro de rango de
edad está presente en todas las páginas.

## Muestra insuficiente

Las celdas con menos de 30 casos sin expandir se dibujan con textura de rayas y
un asterisco, y la gráfica muestra un aviso con cuántas barras están afectadas.
Se dibujan en vez de ocultarse porque un hueco en una gráfica de barras se lee
como un cero.

Con los datos actuales, ninguna combinación alcanzable desde la interfaz (1,302
celdas evaluadas) cae por debajo del umbral: las 94 celdas frágiles que existen
en los datos están todas en ENADIS por entidad, que es justamente lo que la
interfaz no ofrece.

## Estructura

```
src/
├── components/
│   ├── comparacion.js   # los 3 pares, la paleta y qué puede cada fuente
│   ├── agregar.js       # agregación ponderada y regla de fragilidad
│   ├── graficas.js      # librería de Plot, avisos, KPIs, tablas de respaldo
│   ├── filtros.js       # panel de filtros y preparación de series
│   └── tablero.js       # renderer de las páginas temáticas + catálogo de texto
├── data/
│   ├── indicadores.csv  # datos ya calculados que consume el sitio
│   └── dataloader/      # scripts de reconstrucción desde microdatos
├── temas/               # páginas-cascarón (3 líneas cada una)
├── metodologia/
└── index.md
```
