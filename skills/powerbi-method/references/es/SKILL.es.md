---
name: powerbi-metodo-es
description: Method for building and reviewing Power BI reports - thin reports over one shared semantic model, star schema over the warehouse, DAX measure conventions, report design, verification against SQL, publishing and gateway refresh. Use when creating, extending or auditing a Power BI report or dataset. / Método para construir y revisar informes de Power BI - informes finos sobre un modelo semántico compartido, estrella sobre el DW, convenciones DAX, diseño, cuadre contra SQL, publicación y refresco.
---

# Método Power BI

Método genérico para construir un informe de Power BI sobre un almacén de datos relacional (SQL Server o similar), con un modelo semántico compartido y verificación contra la fuente. No depende de ningún cliente ni de ningún sector. Si el usuario escribe en inglés, responda en inglés: los términos del método (star schema, thin report, bookmark, gateway) son los de Power BI.

Detalle y porqués, en dos ficheros de referencia que se leen cuando el paso lo pide:

- `modelo-medidas-y-columnas.md`: forma del modelo, relaciones, medidas DAX, columnas y Power Query.
- `edicion-y-verificacion-por-codigo.md`: edición de un proyecto `.pbip` (TMDL/PBIR) por script, y cómo se da un informe por bueno.

## Principios

1. **Un modelo semántico una vez, informes muchos.** El modelo se construye y publica una sola vez; los informes son ficheros finos conectados en vivo a él.
2. **El almacén manda.** El modelo parte de vistas del almacén; lo que se puede resolver en SQL no se transforma en Power Query.
3. **Estrella, no tabla plana.** Hechos y dimensiones, relaciones uno a muchos en un solo sentido.
4. **Medidas, no columnas calculadas.** La lógica de negocio vive en medidas del modelo.
5. **Solo lectura.** Power BI sirve para VER. Escribir o accionar se hace en otra herramienta.
6. **Sin cuadre contra SQL no se publica.**

## Paso 1. Preguntas de negocio y origen

- Con quien consume el informe: lista cerrada de preguntas que debe responder (unas 10 como máximo) y desde qué dispositivo se lee (móvil o escritorio).
- Con quien responde del almacén: qué vistas alimentan cada pregunta, el grano de cada una y la consulta SQL de control de cada total clave.
- Si falta una métrica en el almacén, se pide la vista; no se parchea en Power Query.

## Paso 2. Modelo en estrella sobre el almacén

Detalle en `modelo-medidas-y-columnas.md`.

- Una tabla del modelo = una vista del almacén (1:1). Prefijos que se trasladan desde SQL: `D_` dimensiones, `F_` hechos, vistas de negocio ya agregadas, `Dim_Fecha` calendario.
- Una dimensión central de negocio y una dimensión de fecha compartida por todos los hechos. Calendario propio marcado como tabla de fechas; sin la fecha automática de Power BI.
- Grano documentado por tabla en una frase («una fila por X y por Y»). Relaciones uno a muchos, de la dimensión al hecho, un solo sentido. Bidireccional solo con justificación escrita. Evitar muchos a muchos: si aparece, falta una dimensión puente.
- **Versionado por periodo desde SQL**: si el almacén clona vistas por año en el nombre, el modelo apunta a la vista del periodo; un periodo nuevo se añade, no sobrescribe el anterior (rompería los históricos).
- **Las columnas son el contrato con SQL**: se conservan los nombres del almacén. No se renombra en Power Query; el formato visible se resuelve en el informe.
- Import por defecto. DirectQuery solo si el dato debe ser en tiempo real y la fuente lo aguanta.
- Rendimiento: solo las columnas que se usan, tipos correctos, claves y columnas técnicas ocultas, sin texto largo en los hechos.
- **Clave compuesta resuelta en la vista** (una sola columna), nunca relaciones por dos columnas ni claves fabricadas en Power Query.
- **Datos que se cargan a cortes → patrón «foto»**: tabla SQL en la que solo se inserta, con `FechaCorte`, y en el modelo una tabla `…_Actual` (estado de hoy) y otra `…_Hist` (evolución).
- **Catálogos y dimensiones pequeñas desde una vista SQL, nunca tecleados en el modelo** (`DATATABLE`): se desincronizan sin avisar.
- **Origen M de un solo paso** (`Sql.Database` y la vista), sin transformaciones; servidor y base de datos desde configuración, nunca literales en el repositorio.
- **Filtro de línea de negocio a nivel de página, nunca de informe.** Al añadir una segunda línea de negocio se revisa todo lo que asumía la primera: vista, maestro, filtros globales y relaciones.
- Tabla desconectada `P_UltimaActualizacion` con el sello del último refresco, visible en la portada. Otras tablas de apoyo desconectadas también con prefijo `P_`.
- Entregable: un documento del modelo con tablas, grano, relaciones y tabla de medidas.

## Paso 3. Medidas DAX

- Se alojan en la tabla de hechos a la que pertenecen, o en una tabla `_Medidas` si son transversales.
- **Nombres de dominio**, legibles, con la unidad en el nombre en los modelos nuevos (`'Tiempo de espera (min)'`, `'% Registros con dato'`). Porcentajes con `%` en el nombre; neto y bruto con una marca clara y documentada.
- **Versiones y desvío**: cuando el negocio gira en torno al cumplimiento (objetivo, presupuesto, real), una medida por versión y una de desvío por versión.
- Una medida base por concepto (suma, recuento); las demás **se construyen sobre medidas y no repiten lógica**.
- `DIVIDE` siempre en lugar de `/`. `BLANK()` cuando no hay dato, no un cero engañoso.
- Filtro con predicado booleano dentro de `CALCULATE`, no `FILTER` sobre la tabla entera. `VAR` para legibilidad y rendimiento.
- Formato (decimales, unidad, miles) fijado en la medida, no en el visual.
- **Poblaciones y umbrales definidos en SQL** (banderas `ES_…`, `EXCESO_…`, `BANDA_…`), una sola definición compartida con cualquier otro informe que cuente lo mismo; el DAX solo cuenta y filtra.
- **Magnitudes con cola larga (tiempos, esperas, importes)**: mediana, media, P90 y peor caso juntos, y los **extremos contados**, no promediados, junto con la magnitud que arrastran.
- **Cobertura al lado de cada media** (`'% X con dato'`): una media calculada sobre el 30 % de los registros es la de esos registros.
- **Misma magnitud, misma base de población** en toda la página.
- Evolución por cortes con el patrón «foto anterior» (`VAR` + `REMOVEFILTERS` + `<`) cuando el negocio vive por cortes y no por calendario.
- Documentar cada medida: qué calcula, grano, fuente y consulta SQL de control. Sin ese par, no está terminada.

## Paso 4. Páginas y diseño del informe

- **Informe como aplicación**: una sola página visible, `Inicio`; el resto, ocultas en modo vista, y se llega a ellas por navegación. Menús por área.
- **Navegación por bookmarks y botones de acción**, no la navegación nativa página a página. Convención `<Pagina>_<Accion>` para no perderse cuando sean cientos.
- **Una idea por página.** Orden de lectura: KPI y semáforos arriba, comparativas en medio, detalle (tabla o matriz) debajo.
- Visuales base: tarjeta, KPI o indicador para cumplimiento frente a objetivo, segmentación en panel lateral, tabla o matriz para el detalle. Mapas solo si el dato es geográfico.
- Visuales personalizados solo en un informe operativo que los necesite y tras revisión de seguridad; los informes ejecutivos son de solo lectura con visuales nativos.
- **Vista de teléfono** en cada página que se consuma desde la aplicación móvil: KPI y semáforos primero, una columna, sin tablas anchas.
- **Divulgación progresiva**: filtros → cinco tarjetas macro → contexto → «lo que hay que mirar» → detalle por drillthrough, en una página oculta con botón de vuelta. **Extremos primero**, bandas pinchables, peores casos ya ordenados; tarjeta con pie que dice la población («sobre N de M»); sin fila de total bajo medianas.
- Páginas hermanas desde **un mismo molde**, pero cada línea mide lo que le duele.
- Una página con bookmarks no se clona ni se rehace encima: se crea la nueva al lado y la vieja queda oculta.
- Plantilla común: mismo tema, mismo logo (como recurso registrado), misma página `Inicio`. Un informe nuevo se clona de la plantilla, no se empieza en blanco.
- Estilo visual: pocos colores, semáforo consistente (mismo umbral en todo el informe), tipografía y tamaños del tema, nada de decoración que no informe.

## Paso 5. Validar

Antes de publicar, otra persona (o un verificador distinto de quien construyó) comprueba, contra el modelo abierto en Desktop por su puerto local de Analysis Services (ver `edicion-y-verificacion-por-codigo.md`):

1. Cada total clave = consulta SQL de control (mismos filtros) = entregable que se envíe, si lo hay, categoría a categoría. Anotar medida, valor en Power BI, valor en SQL y diferencia.
2. **Todas** las medidas evaluadas: ninguna en blanco ni en error, y reglas de coherencia (mediana ≤ P90 ≤ máximo; `>X` ≥ `>2X`; porcentajes en [0, 1]; subconjunto ≤ total). Se informa cuántas comprobaciones **se ejecutaron**, no solo cuántas fallaron. Antes de llamar fallo a una medida, se lee su DAX.
3. Segmentaciones: seleccionar, limpiar y combinar; los totales cuadran en cada selección.
4. Navegación: cada botón o bookmark lleva a donde dice; no hay páginas huérfanas visibles.
5. Móvil: abrir la vista de teléfono.
6. Primeras filas de cada tabla y matriz comparadas con SQL (es donde antes se ven los fallos de agregación).
7. Si el informe se ha editado por código: integridad del proyecto (visuales contados por `visual.json`, ids únicos, botones a páginas que existen, JSON válido).

## Paso 6. Publicación

- Un workspace por cliente o entorno; el modelo semántico se publica una vez y los informes se publican apuntando a él (conexión en vivo).
- Propietario del modelo: una persona nombrada, nunca «nadie». Accesos mínimos; el consumidor entra como lector.
- Si hay seguridad por filas, se define en el modelo y se prueba con «ver como».
- Sin credenciales incrustadas en el fichero del informe.

## Paso 7. Refresco y gateway

- Origen local: **On-premises data gateway** instalado; la fuente se asigna en el servicio con credenciales guardadas en el gateway, nunca en el informe ni en el repositorio.
- Programación acorde a cuándo cambia el dato en el almacén (suele bastar una o dos veces al día, después de las cargas), con aviso de fallo al propietario.
- Tras el primer refresco programado, comprobar que el sello `P_UltimaActualizacion` avanza y que el total clave sigue cuadrando. Un refresco «verde» no es prueba: se mira el dato.
- Dejar documentado: gateway, fuente, horario, propietario y a quién se avisa si falla.

## Lista de comprobación final

- [ ] Preguntas de negocio acordadas con quien consume el informe
- [ ] Vistas de origen entregadas, con consulta de control por total clave
- [ ] Modelo en estrella; grano y relaciones documentados
- [ ] Calendario propio y fecha automática desactivada
- [ ] Columnas con el contrato del almacén, sin renombrar
- [ ] Medidas documentadas; `DIVIDE`; sin lógica repetida
- [ ] Informe fino conectado a un modelo compartido (no un modelo por informe)
- [ ] `Inicio` visible, resto oculto, navegación por bookmarks
- [ ] KPI y semáforos arriba, detalle debajo, una idea por página
- [ ] Vista de teléfono hecha
- [ ] Totales cuadran con SQL (evidencia anotada) y con el entregable, si lo hay
- [ ] Todas las medidas evaluadas sin blanco ni error; comprobaciones ejecutadas contadas
- [ ] Poblaciones y umbrales definidos en SQL; cobertura junto a cada media
- [ ] Catálogos desde vista SQL (ningún `DATATABLE`); filtros de línea a nivel de página
- [ ] Sin credenciales en el fichero del informe; fuente autenticada en el gateway
- [ ] Propietario del modelo y programación de refresco definidos
- [ ] Sello de última actualización visible
- [ ] Nada enviado al destinatario final sin la aprobación del responsable del informe
