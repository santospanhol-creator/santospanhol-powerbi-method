# Modelo, relaciones, medidas y columnas

Referencia de la skill `powerbi-metodo`. Los nombres de tablas, columnas y medidas son ejemplos genéricos (`ventas`, `producción`, `incidencias`); sustitúyalos por los del modelo en el que trabaje.

## 1. Arquitectura: informes finos sobre un modelo compartido

- Los ficheros de informe publicados son solo informe, conectados en vivo a un modelo semántico publicado en el servicio. El modelo se construye una vez y varios informes lo reutilizan. El modelo incrustado en el fichero es solo caché.
- **Informe como aplicación**: una página `Inicio` visible; el resto ocultas en modo vista. Navegación por bookmarks y botones de acción, más páginas-menú por área. No se usa la navegación nativa página a página.
- **Plantilla común**: tema compartido y logos como recursos registrados; los informes nuevos se clonan de la plantilla.
- **Proyectos `.pbip`**: los informes nuevos se guardan como proyecto (informe en PBIR, un JSON por visual; modelo en TMDL, un `.tmdl` por tabla) y se construyen y corrigen por código (ver `edicion-y-verificacion-por-codigo.md`).

## 2. Tablas del modelo: nombres y formas

Cada tabla del modelo es una vista del almacén:

- `D_…` dimensiones, `F_…` hechos, vistas de negocio ya agregadas, `Dim_Fecha` calendario, `P_…` tablas desconectadas (`P_UltimaActualizacion` = sello de refresco; `P_ValoresMedida` = selector de medida).
- **La tabla del modelo se nombra por su papel aunque la vista SQL se llame de otra forma**: la vista `vw_IncidenciasActual` entra al modelo como `F_Incidencias`. Las columnas, en cambio, no se renombran.
- **Patrón «foto»** para datos que se cargan a cortes: tabla SQL en la que solo se inserta (cada carga es una foto con `FechaCorte`; recargar la misma foto no duplica porque las vistas se quedan con la última). En el modelo, dos tablas: `…_Actual` (última foto, estado de hoy) y `…_Hist` (todas las fotos, evolución).
- **Versionado por periodo** desde SQL: vistas con el año en el nombre (`…2025`, `…2026`); un periodo nuevo clona la vista, no sobrescribe.
- **Catálogos desde SQL, nunca tecleados en el modelo.** Una dimensión pequeña escrita como `DATATABLE` se quedó sin el elemento nuevo el día que se añadió en origen, y sin avisar: no tenía relaciones, así que los totales cuadraban, pero la segmentación no lo ofrecía. Se sustituye por una tabla que lee de la vista del catálogo, conservando nombre de tabla y columnas para no romper visuales.

## 3. Relaciones (TMDL)

```
relationship <Hecho>_<Dim>
    fromColumn: <Hecho>.<CLAVE>
    toColumn:   <Dim>.<CLAVE>
```

- **Muchos a uno, un solo sentido, activas** (son los valores por defecto de TMDL, por eso no se declaran).
- **Clave compuesta resuelta en SQL**: si el grano de la dimensión es entidad + periodo, la vista expone una sola columna de texto concatenada y la relación va por ella.
- La tabla de foto actual y la histórica se relacionan las dos con la misma dimensión; la fecha de negocio del hecho va a `Dim_Fecha[Date]`.
- Un comentario que diga qué une y con qué clave. Ojo: en TMDL el comentario `///` delante de `relationship` rompió la carga del modelo en las versiones de Desktop probadas; compruebe que su versión lo admite antes de generalizarlo.
- Al sustituir una tabla por código, **las relaciones se apuntan antes y se recrean después**, conservando cardinalidad, dirección de filtro, filtro de seguridad e `IsActive`: quitar la tabla las borra, y el servidor rechaza la transacción entera si alguna queda apuntando al vacío.

## 4. Medidas DAX

Forma de cada medida en TMDL:

```
/// Qué cuenta, sobre qué población, y de dónde sale el umbral si lo hay.
measure 'Nombre legible (unidad)' = <DAX>
    formatString: #,##0
    displayFolder: <Área> | <Área> evolución | KPI Tiempos | KPI Extremos
```

- `displayFolder` por área, con carpetas aparte para evolución, tiempos y extremos. Descripción `///` sobre toda medida que no sea evidente.
- `formatString` siempre en la medida: `#,##0` conteos, `0.0` minutos u horas, `0.0%;-0.0%;0.0%` porcentajes, `+#,##0;-#,##0;0` variaciones con signo, `dd/mm/yyyy hh:mm` sellos de fecha.
- Medidas base: `COUNTROWS(hecho)`, `DISTINCTCOUNT(hecho[Entidad])`, `SUM`.
- **Filtro con predicado booleano**, no `FILTER` sobre la tabla: `CALCULATE(SUM(H[Importe]), H[Tipo] = 1)`. `FILTER(tabla, flag = 1)` solo dentro de los iteradores `…X`.
- **Las poblaciones las define SQL**: la vista expone banderas (`ES_<POBLACION>`, `CANCELADO`, `ANOMALIA_<X>`, `EXCESO_<UMBRAL>`, `BANDA_<X>`) y las medidas solo las cuentan o filtran. Los **umbrales viven en una sola definición** (la vista, o el script que también genera cualquier informe en otro formato), nunca duplicados en DAX: dos informes que cuentan cosas distintas con el mismo nombre es el fallo que se evita.
- **Distribuciones con cola larga**: mediana (`MEDIANX`), media, P90 (`PERCENTILEX.INC(…, 0.9)`) y peor caso (`MAX`) **juntas**; la distancia entre mediana y media ES el dato. Y los **extremos contados, no promediados**: `'Entidades con más de <Umbral>'`, `'… con más de 2× <Umbral>'`, su `%` sobre la población y **la magnitud que arrastran**, no solo cuántas son.
- **Cobertura al lado de cada cifra**: `'X con dato'` y `'% X con dato'`.
- **Unidades humanas**: `DIVIDE(minutos, 60)` cuando la magnitud se piensa en horas.
- `DIVIDE` siempre. `BLANK` cuando no hay dato. **Excepción documentada**: una medida `'X (color)' = COALESCE(COUNTROWS(…), 0)` solo para formato condicional.
- **Sello de datos en texto** para una tarjeta: `"Datos del " & FORMAT(MAX(H[FechaCorte]), "dd/mm/yyyy hh:mm")`.
- **Misma magnitud, misma base en toda la página**: una revisión encontró una tarjeta de importe sin excluir cancelados y un porcentaje del mismo importe que sí los excluía; quien dividía las dos tarjetas no obtenía el porcentaje mostrado. Toda medida que comparta magnitud comparte filtro de población.
- Una medida no puede llamarse como una columna de ninguna tabla del modelo; renombre la columna o dé a la medida un nombre distinto.

**Inteligencia temporal.** Cuando el eje de análisis es el periodo de negocio o la foto, y no el calendario, se usa el patrón «foto anterior» en lugar de `TOTALYTD` o `SAMEPERIODLASTYEAR`:

```
measure 'Variación vs foto anterior' =
    VAR _corte    = MAX ( H[FechaCorte] )
    VAR _anterior = CALCULATE ( MAX ( H[FechaCorte] ), REMOVEFILTERS ( H[FechaCorte] ), H[FechaCorte] < _corte )
    VAR _prev     = IF ( NOT ISBLANK ( _anterior ),
                         CALCULATE ( COUNTROWS ( H ), REMOVEFILTERS ( H[FechaCorte] ), H[FechaCorte] = _anterior ) )
    RETURN IF ( NOT ISBLANK ( _prev ), COUNTROWS ( H ) - _prev )
```

Con `FechaCorte` en el eje da una cifra por foto y su variación. **Aviso**: en un script de la vista TMDL de Desktop, una medida no puede llamar a otra creada en el mismo script; se repite la lógica con un comentario que lo explica, o se crean en dos pasadas.

## 5. Columnas

- **Conservan el contrato SQL** (idioma, mayúsculas, sin renombrar).
- `summarizeBy: none` en texto, fechas y banderas de texto; `sum` en numéricas y en banderas 0/1 (`int64`), que así se cuentan sumando.
- **Modelo Import**: una columna nueva de la vista no entra sola. Hay que declararla en el `.tmdl` (o refrescar el esquema) y refrescar; si no, la medida que la usa da error y la página sale en blanco.
- **Toda columna de la vista lleva alias.** Una expresión sin nombre (por ejemplo, un `ISNULL(...)` sin alias) impide que el modelo cargue.
- **Excepción al no renombrar**: una columna de enlace se renombra en el modelo al texto que debe salir de cabecera (con `sourceColumn` intacto). Los visuales y el DAX la referencian por su nombre de modelo, nunca por `sourceColumn`: eso rompe el visual con «problema con uno o más campos».
- **Enlaces clicables**: columna con `dataCategory: WebUrl` y `urlIcon: true` (icono en lugar de la URL entera). Las propiedades de formato inventadas rompen el visual sin error claro: use la vía documentada.
- **Columnas calculadas**: existen en modelos antiguos y son frágiles (tres quedaron rotas apuntando a una tabla ya eliminada y obligaron a refrescar con `DataOnly` en lugar de `Full`). Refuerzan la regla: medidas, o columna en la vista SQL.

## 6. Power Query / M

Un solo paso, sin transformaciones:

```
let
    Origen = Sql.Database("<servidor>", "<bd>"),
    dbo_<vista> = Origen{[Schema="dbo", Item="<vista>"]}[Data]
in
    dbo_<vista>
```

Toda limpieza, unión y cálculo de población va en la vista. Servidor y base de datos salen de la configuración, nunca literales en el repositorio; las credenciales las pone Desktop o el gateway. Cambiar el origen de una tabla es cambiar el `Item` y el nombre del paso, nada más.

## 7. Diseño y tipos de visual

- Mezcla habitual: segmentación (paneles de filtro), botones de acción (navegación), tarjeta, KPI e indicador (cumplimiento frente a objetivo), tabla y matriz (detalle). Geoespacial con mapa de formas o mapa Azure sobre GeoJSON incrustado. Visuales personalizados (matriz editable con escritura, Deneb o Vega, segmentaciones avanzadas, Power Apps embebido) solo en el informe operativo; el resto, solo lectura.
- **Divulgación progresiva en tres niveles**: cabecera y filtros → cinco tarjetas macro → dos gráficos de contexto → gráfico y tabla «lo que hay que mirar» → drillthrough a una página de detalle oculta (`pageBinding: Drillthrough`, filtros de drill por las dimensiones, botón «< VOLVER»).
- **Extremos primero**: arriba, los contados («N casos de más de X»); en medio, las **bandas** pinchables (cortes en los umbrales de negocio, no en números redondos; la peor banda arriba); abajo, **los peores casos ya ordenados** de mayor a menor. El orden es parte de la información.
- **Tarjeta = cifra + rótulo + pie con la población** («sobre N de M»). Sin el pie, no se lee.
- **Sin fila de total bajo medianas**: se lee como una suma y no lo es.
- **Detalle de un proceso por fases**: barra apilada, un registro por fila, una fase por segmento; el tiempo de espera en tonos cálidos y el tiempo activo en verdes, para ver dónde se fue el tiempo sin leer números. En la tabla de detalle, las fechas y horas reales de cada fase, no solo las duraciones.
- **Colores sacados del informe existente, no inventados.** Con una sola serie, el `dataPoint` va sin selector (con selector de `metadata` cae al azul del tema). Los colores de un tema que no se resuelven en un visual generado por código se escriben como literales `'#RRGGBB'`.
- La paleta y la tipografía son las de la plantilla de cada cliente.
