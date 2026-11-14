# Edición por código y verificación

Referencia de la skill `powerbi-metodo`. Es válida para un proyecto `.pbip` (informe en PBIR, modelo en TMDL) en Windows con Power BI Desktop. Microsoft cambia los esquemas con las versiones de Desktop: fije versiones y pruebe cada actualización. Lo marcado como «observado» es lo comprobado en la práctica, no una especificación.

## 1. Formatos

`.pbip` = carpeta `<Nombre>.Report/definition/` (PBIR: `pages/pages.json`, `pages/<id>/page.json`, `pages/<id>/visuals/<id>/visual.json`) + `<Nombre>.SemanticModel/definition/tables/<tabla>.tmdl`. Todo es texto: se edita con scripts pequeños, uno por cambio, que se versionan en git. **El `.pbix` o `.pbip` vive en la carpeta de trabajo del cliente, no en git.**

## 2. Reglas de edición (cada una costó horas)

1. **Con el `.pbip` abierto en Desktop no se edita su TMDL**: al cerrar, Desktop vuelca su modelo en memoria encima y se lleva lo escrito (pasó con decenas de columnas y medidas). Se cierra primero.
2. Columnas y medidas se insertan **antes del bloque `partition`** (TMDL lo quiere al final de la tabla; detrás, el fichero no abre). `lineageTag` nuevo (UUID) por objeto.
3. **Idempotentes**: comprobar si el objeto ya existe antes de añadirlo; copia `.bak_<motivo>` antes de escribir; y después **contar en el fichero** lo escrito. El «escrito» del script no cuenta.
4. Al añadir a una lista de formato (`objects.values` de una tabla) **se añade, no se sustituye**: asignarla entera borró el formato de letra sin dar error.
5. **No clicar en el lienzo en modo edición para automatizar**: crea visuales vacíos o abre editores. La página activa se cambia en `pages.json` (`activePageName`) y se pulsa «Apply external changes».
6. **Las páginas con bookmarks no se clonan ni se rehacen encima**: los bookmarks apuntan a ids de visual, y el clon muestra todas las capas a la vez. Se crea una página nueva al lado y la antigua queda oculta hasta decidir.
7. Un **molde** (función generadora) para páginas hermanas: misma disposición, para que lo que se aprende en una valga en la otra. Pero las cifras son de cada negocio: cada línea mide lo que le duele.
8. **Filtro de una línea de negocio a nivel de página, nunca de informe**: un filtro global pensado para la primera línea dejó entera en blanco, sin error, la página de la segunda.
9. Observado: un cambio hecho por TOM sobre el modelo abierto no existe en disco hasta guardar; si se cierra Desktop sin guardar, se pierde. Compruebe el `.tmdl` o el `visual.json` del disco.
10. Observado: una matriz de cinco niveles se pintaba solo hasta el primer bloque; cuatro niveles como máximo y el detalle en otra página. No escriba estados de despliegue (`expansionStates`) a mano: la matriz arranca plegada y el +/− funciona solo.
11. Observado: si el fichero de un script PowerShell no lleva BOM, Windows PowerShell 5.1 lo lee en ANSI y un carácter como «·» llega mal a una medida. Use BOM, o pase la expresión desde un fichero UTF-8; al leer la salida de PowerShell desde Python, fuerce UTF-8.
12. Observado: con dos Power BI abiertos, «la primera ventana de Power BI» puede ser la de otra persona. Elija la ventana por su título y compruebe cuál está delante justo antes de cualquier acción sobre ella.

**Scripts TMDL para pegar** (cuando el modelo es un `.pbix` sin guardar como `.pbip`): un script `createOrReplace` más `ref table <tabla>` con las medidas, y otro con las relaciones; se pegan en la vista TMDL de Desktop.

## 3. El modelo abierto, sin servidor MCP

Desktop levanta un Analysis Services local: el puerto sale del proceso `msmdsrv` (`-s <carpeta>` → `msmdsrv.port.txt`, UTF-16 con nulos). Con las librerías que trae SSMS (se localizan en el equipo; no se redistribuyen):

- **ADOMD** para leer: `EVALUATE SUMMARIZECOLUMNS(...)`, o evaluar todas las medidas.
- **TOM** para escribir: sustituir la partición de una tabla por M, recrear relaciones, `RequestRefresh(DataOnly)` (no `Full`: recalcula todas las columnas calculadas y una rota hace fracasar la transacción entera). Con `--ver` (en seco) primero, copia del fichero antes y comprobación por DAX después.

Existen también servidores MCP de terceros y el de Microsoft para Power BI; esta skill no depende de ninguno y es compatible con ellos: el método dice qué orden dar, la herramienta la ejecuta.

**Vista previa antes de construir**: un HTML local generado con exactamente las mismas filas que se cargan en la base de datos, para acordar la página antes de montarla en Power BI.

## 4. Verificación: cómo se da un informe por bueno

- **Tres fuentes, la misma cifra**: el entregable (por ejemplo, un Excel), la vista SQL que consume Power BI y el modelo abierto, categoría a categoría. Que dos cuadren no basta (un fallo real: el resumen traía casos de una categoría nueva y la base de datos ninguno, porque la carga no leía la hoja nueva).
- **Y el primer tramo**: lo cargado contra el sistema de origen (que cada registro con id exista allí y traiga las mismas magnitudes). Si los tres se equivocan igual, cuadran y están mal.
- **Coherencia de todas las medidas** contra el modelo: ninguna en blanco ni en error, y reglas internas (mediana ≤ P90 ≤ peor caso; `>X` ≥ `>2X` ≥ `>4X`; % entre 0 y 1; subconjunto ≤ total).
- **Contar las comprobaciones EJECUTADAS, no solo las fallidas**: el DAX vuelve como texto; una comprobación de tipo mal escrita hizo que no se ejecutara ninguna y el resumen dijo «0 fallidas». Un verificador que concluye «no encuentro nada» debe demostrar antes que sabe encontrar algo (siembre un fallo conocido y compruebe que lo detecta).
- **Antes de llamar «fallo» a una medida, lea su DAX**: la población que usa puede ser la buena y el supuesto, el suyo.
- **Distinga «está abierto otro modelo» de «este modelo está roto»** en el aviso: un aviso que asusta sin motivo se deja de leer.
- **Integridad del PBIR** antes de abrir: orden de páginas, páginas ocultas, número de visuales contando ficheros `visual.json` (no carpetas: la sincronización de una carpeta compartida deja carpetas vacías), ids repetidos, id interno igual al nombre de la carpeta, cada botón de la portada lleva a una página que existe, todo JSON válido.
- **Los saldos no se mueven**: repartir por actividad o por otra clave solo redistribuye; la suma por clave y periodo tiene que ser la de la fuente oficial. Se comprueba en cada regeneración, con una tolerancia explícita.
- **Nada de blancos en los filtros**: un blanco esconde importes al filtrar. Etiquetas explícitas («(fuera del maestro)», «(vacío en el maestro)»).
- **Comparar lo comparable**: real de ocho meses contra presupuesto de los mismos ocho meses; puntos porcentuales y no % para una tasa.
- La verificación entra en la **rutina de generación** (el mismo proceso que genera y carga los datos), y la carga a la base de datos se hace a la vez que se entregan los informes, para que lo que se ve en Power BI sea lo mismo que se ha enviado.
- **Nunca se pisa lo que el usuario edita a mano**: antes de regenerar un informe que tiene abierto, guarde, haga copia, genere la versión anterior en otra carpeta, compare y pase al generador lo que él haya cambiado.
- Cada cambio deja un documento breve (qué se pidió, qué se encontró, por qué, qué se aprende) y su commit con el porqué.
