# Método Power BI (Santospanhol) · Power BI Method

[Español](#español) · [English](#english)

---

## Español

Plugin con una skill, `powerbi-method`, que enseña a Claude un método concreto para construir y revisar informes de Power BI sobre un almacén de datos relacional (SQL Server o similar). No es una herramienta que edite sus modelos: es el criterio sobre qué orden dar, en qué momento y cómo comprobar que el resultado es correcto.

> **¿Prefiere que revisemos su modelo?** Santospanhol, S.L. ofrece la **Auditoría de modelo Power BI**: revisamos su modelo y sus informes con este mismo método. Le entregamos un informe con los problemas ordenados por impacto (cifras que no cuadran, medidas frágiles, rendimiento, refresco) y el plan para corregirlos. Escríbanos a santospanhol@gmail.com con el asunto «Auditoría Power BI».

### Qué contiene

- **Informes finos sobre un modelo semántico compartido**: el modelo se publica una vez y los informes se conectan a él.
- **Modelo en estrella sobre vistas SQL**: una tabla del modelo por vista, columnas como contrato con SQL, calendario propio, patrón «foto» para datos a cortes.
- **Convenciones de medidas DAX**: una medida base por concepto, `DIVIDE`, poblaciones y umbrales definidos en SQL, mediana junto a P90 y extremos contados, cobertura junto a cada media.
- **Diseño del informe**: una idea por página, divulgación progresiva, vista de teléfono, navegación por bookmarks.
- **Verificación**: «sin cuadre contra SQL no se publica», evaluación de todas las medidas y recuento de las comprobaciones ejecutadas.
- **Edición por código** de proyectos `.pbip` (TMDL y PBIR), con las trampas que más tiempo cuestan.
- **Publicación y refresco** con gateway.

### Cómo se usa

Instale el plugin y pida a Claude, por ejemplo: «revisa este modelo contra el método Power BI», «diseña el modelo en estrella para estas vistas» o «¿qué comprobaciones hago antes de publicar este informe?». La skill se activa sola cuando la tarea trata de crear, ampliar o revisar un informe o un modelo de Power BI.

```
/plugin install santospanhol-powerbi-method --marketplace santospanhol-creator/santospanhol-powerbi-method
```


Estructura:

```
.claude-plugin/plugin.json
skills/powerbi-method/SKILL.md
skills/powerbi-method/references/model-measures-and-columns.md
skills/powerbi-method/references/editing-and-verification-by-code.md
skills/powerbi-method/references/es/   (versión en español)
```

### Qué hace el plugin en su equipo

Nada más que aportar instrucciones de texto a Claude. No ejecuta código, no lleva scripts ni binarios, no abre conexiones de red, no lee credenciales y no envía datos a ningún sitio. Cuando la skill describe un script o una consulta (por ejemplo, evaluar medidas contra el modelo abierto en Power BI Desktop), es Claude, a petición suya y con sus permisos habituales, quien lo propone y lo ejecuta.

### Alcance y límites

- Pensado para Windows y Power BI Desktop con proyectos `.pbip`; la verificación en local se apoya en el Analysis Services que levanta Desktop.
- El método se ha aplicado sobre un almacén en SQL Server con gateway; otros orígenes no se han probado.
- Es documentación de método, sin garantía (ver la licencia). Microsoft cambia los formatos con cada versión de Desktop: pruebe antes de generalizar.
- Privacidad: el plugin solo contiene texto de instrucciones (una skill). No recoge, guarda ni envía datos personales ni de ningún otro tipo, y no se conecta a ningún servicio externo.
- Soporte: este plugin se publica tal cual. Contacto: santospanhol@gmail.com · [repositorio](https://github.com/santospanhol-creator/santospanhol-powerbi-method).

### Marcas

Power BI, Power Query, DAX y SQL Server son marcas de Microsoft Corporation. Este plugin es independiente y no está afiliado, patrocinado ni respaldado por Microsoft ni por Anthropic.

### Licencia

MIT. Véase `LICENSE`.

---

## English

A plugin with one skill, `powerbi-method`, that teaches Claude a specific method for building and reviewing Power BI reports on a relational data warehouse (SQL Server or similar). It is not a tool that edits your models: it is the judgement about which step to take, when, and how to check that the result is correct. The skill is written in English (Spanish version in `references/es/`); Claude answers in the language you write in.

> **Want us to review your model?** Santospanhol, S.L. offers a **Power BI Model Audit**: we review your model and reports against this same method. You get a written report with the issues ranked by impact (numbers that don't reconcile, fragile measures, performance, refresh) and a plan to fix them. Email santospanhol@gmail.com with the subject "Power BI audit".

### What it covers

- **Thin reports over one shared semantic model**: publish the model once, connect many reports to it.
- **Star schema over SQL views**: one model table per view, columns kept as the contract with SQL, own calendar table, a "snapshot" pattern for data loaded at cut-off dates.
- **DAX measure conventions**: one base measure per concept, `DIVIDE`, populations and thresholds defined in SQL, median next to P90 and counted extremes, coverage next to every average.
- **Report design**: one idea per page, progressive disclosure, phone layout, bookmark navigation.
- **Verification**: "no publish without reconciliation against SQL", every measure evaluated, and a count of the checks actually executed.
- **Editing by code** for `.pbip` projects (TMDL and PBIR), including the traps that cost the most time.
- **Publishing and refresh** through a gateway.

### How to use it

Install the plugin and ask Claude, for example: "review this model against the Power BI method", "design the star schema for these views" or "what should I check before publishing this report?". The skill activates on its own when the task is about creating, extending or reviewing a Power BI report or model.

```
/plugin install santospanhol-powerbi-method --marketplace santospanhol-creator/santospanhol-powerbi-method
```


### What the plugin does on your machine

Nothing beyond giving Claude text instructions. It runs no code, ships no scripts or binaries, opens no network connections, reads no credentials and sends no data anywhere. Where the skill describes a script or a query (for example, evaluating measures against the model open in Power BI Desktop), Claude proposes and runs it at your request and under your usual permissions.

### Scope and limits

- Designed for Windows and Power BI Desktop with `.pbip` projects; local verification relies on the Analysis Services instance that Desktop starts.
- The method has been applied to a SQL Server warehouse with a gateway; other sources are untested.
- This is method documentation, provided without warranty (see the license). Microsoft changes the file formats between Desktop versions: test before generalising.
- Privacy: the plugin contains instruction text only (one skill). It does not collect, store or transmit personal or any other data, and it connects to no external service.
- Support: this plugin is published as is. Contact: santospanhol@gmail.com · [repository](https://github.com/santospanhol-creator/santospanhol-powerbi-method).

### Trademarks

Power BI, Power Query, DAX and SQL Server are trademarks of Microsoft Corporation. This plugin is independent and is not affiliated with, sponsored by or endorsed by Microsoft or Anthropic.

### License

MIT. See `LICENSE`.

---

Santospanhol, S.L. · NIF B19348903
