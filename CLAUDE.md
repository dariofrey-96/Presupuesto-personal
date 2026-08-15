# CLAUDE.md

Contexto persistente para este repositorio.

## Usuario

- Email: dariofrey@hendercross.com
- **Sin background técnico** — no lee código. Explicar todo en criollo, sin jerga ni nombres de funciones salvo que haga falta. Al terminar un cambio, mostrarlo funcionando (abrir la app en el navegador) en vez de solo describir el diff: la validación no puede venir de que el usuario lea el código.

## Contexto general: Presupuesto Personal + Plan de Retiro

Dos apps hermanas (HTML/JS, hosteadas en GitHub Pages), mismo sistema de diseño (cards, modo claro/oscuro adaptativo, mobile-first, bottom nav, sidebar de parámetros):

- **Presupuesto Personal** — repo `Presupuesto-personal` (este repo), en pesos argentinos.
- **Plan de Retiro** — repo `Plan-retiro-dashboard`, en dólares (USD) por defecto. Proyección de retiro + cartera de inversión con precios en vivo e historial automático vía GitHub Actions.

**Visión a futuro**: una sola app unificada con cuentas de usuario (login), que junte presupuesto e inversiones, pensada para gente que no sabe de finanzas ni es organizada. Decisión del usuario: **es la meta final pero no es urgente** — por ahora se sigue mejorando cada app standalone. No construir login/backend salvo pedido explícito, pero evitar decisiones que compliquen esa unión más adelante.

## Este repo (Presupuesto Personal)

App de una sola página en pesos argentinos. Todo vive en `index.html` (~2200 líneas, HTML + CSS + JS en el mismo archivo). El repo solo tiene `README.md` e `index.html` — sin build, sin dependencias, sin backend, sin GitHub Actions.

### Pestañas

1. **Presupuesto** — sliders por categoría (salario, alquiler, servicios, supermercado, nafta, prepaga, etc.) para armar el presupuesto mensual planificado.
2. **Gastos del mes** — carga manual de gastos reales (fecha, categoría, monto) y comparación contra lo presupuestado.
3. **Histórico** — meses ya "cerrados" con `closeMonth()`, que guarda una foto del mes (ingreso, gasto real, superávit, desglose por categoría).
4. **Proyección** — evolución del salario y gastos a varios años (nominal y real), con crecimiento salarial e inflación ARS configurables.

### Dónde se guardan los datos

Todo en `localStorage` del navegador — **no hay servidor ni sincronización automática** (a diferencia del repo de Plan de Retiro, que sí sincroniza a GitHub):

- `presupuesto_params_v1` — sliders y toggles del presupuesto.
- `presupuesto_gastos_v1` — gastos cargados, agrupados por mes (`{"2026-08": [...]}`).
- `presupuesto_snapshots_v1` — cierres de mes del histórico.
- `presupuesto_tc_v1` — tipo de cambio ARS/USD cargado a mano.
- `presupuesto_theme` — tema claro/oscuro.
- `presupuesto_alertas_cfg_v1` — config de alertas de categoría (`{activas, umbral}`).
- `presupuesto_alertas_estado_v1` — qué alertas ya se mostraron y cuáles se silenciaron, por mes y categoría (`{"2026-08": {"juntadas": {nivel, mute}}}`).
- `planRetiro_params_v1` — **clave compartida con la app de Plan de Retiro.**

### Integración con Plan de Retiro (importante)

Las dos apps están publicadas bajo el mismo dominio (`dariofrey-96.github.io`), así que **comparten el `localStorage` del navegador**. `syncAhorroARetiro()` toma el superávit real del mes, lo convierte a USD con el tipo de cambio cargado a mano, y lo escribe en `planRetiro_params_v1.ahorroMensual` — o sea, el ahorro mensual del plan de retiro se alimenta del presupuesto real. Al tocar esa clave, tener en cuenta que la otra app la lee.

### Categorías

Hay 22 categorías fijas en el array `GASTO_CATS` (id + label). Los ids coinciden con los ids de los sliders del presupuesto, que es como se compara "presupuestado vs. real" por categoría.

### Backup

Export/import manual a archivo JSON (`exportBackup()` / `importBackup()`), que abarca las claves de params, gastos, snapshots, tipo de cambio y alertas (`BACKUP_KEYS`). Es el único mecanismo de respaldo — si el usuario limpia el navegador sin exportar, pierde los datos.

### Alertas de categoría (implementado 2026-08-13)

Sección `── ALERTAS DE CATEGORÍA ──` en el script principal, más el modal (después de `sheet-backdrop`), el `#alertas-banner` en la pestaña de gastos y el grupo "Alertas de gasto" en el sidebar. Dos niveles: `NIVEL_AVISO` (umbral configurable, default 80%) y `NIVEL_EXCESO` (100%).

- `chequearAlertaCat(cat)` se llama al final de `addGasto()` y dispara el modal solo si ese rubro cruzó un nivel **nuevo**.
- `renderAlertasBanner()` se llama desde `renderGastos()`; además destraba el estado guardado si el rubro bajó de nivel (borrar un gasto o subir el presupuesto).
- El estado es por mes, así que se resetea solo al cambiar de mes. Solo alerta rubros con presupuesto asignado (`PV_CAT_IDS`, excluye "Otro").

**Retoques que el usuario todavía no evaluó**: si el modal centrado resulta invasivo (alternativa: toast desde arriba), si conviene más de un umbral de aviso, y el tono de los textos.

### Pagos repartidos en varios meses (implementado 2026-08-13)

Nació de un problema real del usuario: paga el gimnasio o el mantenimiento del auto de una sola vez por 3-4 meses, y el mes del pago disparaba una falsa alarma de exceso. Ahora el formulario de gastos tiene un select **"¿Cuántos meses cubre?"** y el pago se reparte en una parte por mes.

- `addGasto()` usa `repartirEnMeses()` (el sobrante del redondeo va al primer mes, la suma siempre da exacto) y `fechaDesplazada()` (clampea el día al último del mes, ej. 31/01 → 28/02).
- Genera **una entrada por mes**, cada una en el bucket de su mes, unidas por `grupo`. Campos extra: `grupo`, `cuota`, `cuotas`, `montoTotal`, `fechaPago`. Los gastos de un solo mes se guardan igual que antes (sin esos campos).
- Decisión de diseño: **se prorratea en todos lados** (KPIs, comparativa, alertas, superávit, snapshots), no solo en la comparativa. La app es mensual de punta a punta y mezclar caja con devengado daría dos "gastado este mes" distintos. **Efecto colateral conocido**: el superávit del mes del pago queda mejor que la caja real, y eso es lo que alimenta `syncAhorroARetiro()`. Avisado al usuario el 2026-08-13; si le molesta, ahí está la palanca.
- `removeGasto()` borra el grupo entero (con `confirm`), porque media parte de un pago repartido no significa nada.
- `renderFuturoNote()` muestra en la pestaña de gastos cuánto ya está cubierto en meses siguientes, para que no parezca que la plata se perdió.

**Opción "Calcularlo según mi presupuesto" (agregada 2026-08-13, corregida el mismo día).** Para gastos irregulares el usuario no sabe cuántos meses cubre el pago, pero sí razona por **bolsa anual** (ej. mantenimiento del auto: "tengo $600.000 al año"). `planBolsa()` devuelve una lista de `{offset, monto}`.

- **Llena el hueco libre de cada mes, no divide en partes iguales.** Primera versión dividía parejo ignorando lo ya gastado, y el usuario encontró el bug: si en el mes del pago ya había un gasto de ese rubro, el prorrateo lo sumaba encima y disparaba una falsa alarma de exceso — justo lo que la función venía a evitar. Ahora, si en agosto ya había $30K de $50K, ahí entran $20K y el resto sigue de largo. Un mes ya lleno se **saltea** entero (los `offset` no son necesariamente consecutivos).
- Antes de cargar muestra un `confirm` con la cuenta completa: mensual, anual, cuánto había ya en el mes, y el desglose mes por mes. El usuario no lee código: la cuenta tiene que ser visible. Con más de 6 meses el detalle se condensa.
- Si no entra ni llenando 12 meses, el sobrante va al último mes y avisa por cuánto se pasó del presupuesto anual.
- Errores cubiertos: categoría sin presupuesto (o con el toggle apagado) y categoría "Otro", que no tiene presupuesto propio.
- **Elegir los meses a mano sigue dividiendo en partes iguales** y sin `confirm` — ahí el usuario ya sabe cuántos meses cubre (caso gimnasio).
- **Efecto esperado**: al llenar el mes justo hasta el tope, ese mes queda en 100% y salta la alerta "Usaste todo el presupuesto". Es correcto (el rubro quedó consumido), pero si resulta ruidoso, la palanca es no chequear alertas en las partes auto-prorrateadas.

### Gastos fijos / recurrentes (implementado 2026-08-13)

Sección `── GASTOS FIJOS / RECURRENTES ──`, más `#fijos-card` y `#sug-card` en la pestaña de gastos. Clave nueva: `presupuesto_recurrentes_v1` = `{ fijos: {clave: {cat, nota, monto, dia}}, descartados: [claves] }`.

- **Clave de identidad**: `cat + '|' + normNota(nota)`. `normNota()` baja a minúsculas, saca acentos y colapsa espacios, así "Cuota Mensual" y "cuota mensual" son el mismo gasto.
- **Detección** (`detectarCandidatosFijos`): mira los últimos 3 meses; sugiere si aparece en ≥2 meses **y exactamente 1 vez por mes** — ese segundo filtro es lo que evita sugerir "supermercado", que se carga varias veces al mes. Excluye los que tienen `grupo` (pagos repartidos), porque esos ya se auto-cargan y sugerirlos los duplicaría.
- **Nunca carga solo**: sugiere, y el usuario decide. Al marcarlo como fijo, cada mes aparece pendiente con el monto del mes anterior **editable** (requisito explícito por la inflación). Al cargarlo, el monto nuevo pisa al viejo para el mes siguiente.
- Decir "No" o quitar un fijo mete la clave en `descartados` para no volver a preguntar.
- La tarjeta se colapsa a una línea cuando ya está todo cargado.
- **Se agregó una cola de alertas** (`alertaCola`) porque "Cargar todos" puede disparar varias a la vez y antes se pisaban entre sí.

### Vista "Esta semana" / pantalla de inicio (implementado 2026-08-13)

Pestaña nueva `inicio`, **primera y por defecto** (`currentTab = 'inicio'`, `view-presupuesto` arranca oculto). Sección `── INICIO / ESTA SEMANA ──`, `renderInicio()`. La semana va **de lunes a domingo** (`lunesDeLaSemana()`); como puede cruzar dos meses, `todosLosGastos()` aplana todos los buckets.

Cuatro bloques: gastado de la semana con comparación contra la semana anterior *a la misma altura* (`totalHastaDia`, si no compararía media semana contra una entera), barritas por día con hoy resaltado, **ritmo del mes** (`ritmoDelMes()`: barra de % de presupuesto usado con una marca en el % de mes transcurrido, más "te quedan $X para Y días = $Z por día"), y "Te está esperando" que resume fijos pendientes + alertas + sugerencias con botones que llevan a Gastos.

- `renderInicio()` se llama desde `setTab('inicio')` y en el INIT. **A propósito NO se llama desde `recalc()`**: recalc corre en cada movimiento de slider y renderInicio lee varias claves de localStorage.
- La barra inferior pasó de 5 a 6 ítems; se acortaron labels ("Presup.", "Histór.", "Proyec.") y se bajó el padding. Verificado: entran a 375px sin cortarse.
- Estados vacíos cubiertos: sin gastos, y sin presupuesto definido (ahí no da veredicto, invita a definirlo).

**Nota para verificar en el navegador**: si el panel de preview no está compositando, las transiciones CSS quedan congeladas y `getComputedStyle` del `body` devuelve el color viejo al cambiar de tema. No es un bug — desactivar `transition` para medir.

### Importar resumen de tarjeta / billetera (implementado 2026-08-14)

Sección `── IMPORTAR RESUMEN ──`, modal `#imp-modal`, botón en el formulario de gastos. Clave nueva: `presupuesto_import_reglas_v1` = `{ "clave de comercio": "catId" }`.

- **pdf.js se carga por CDN y a demanda** (`cargarPdfJs()`), recién al abrir el importador — no penaliza el arranque de la app. Versión 3.11.174 (UMD), worker aparte.
- `lineasDelPdf()` reagrupa los fragmentos posicionados del PDF en líneas juntando los que están a la misma altura (`transform[5]`, tolerancia de 3pt) y ordenando por `x`.
- `parsearLineas()` es **agnóstico del banco**: busca fecha + importe en cada línea. Descarta por `RE_IGNORAR` (totales, saldo, vencimiento, CBU, "su pago") y descarta montos negativos (pagos y devoluciones no son gastos). Si hay varias columnas (pesos y dólares), **se queda con el importe más grande** — en Argentina el de pesos siempre lo es.
- `parsearMonto()` detecta si el separador decimal es coma o punto mirando cuál viene último.
- Categorización: primero las reglas aprendidas, después `DICCIONARIO_CAT` (comercios argentinos), si no cae en "otro".
- **Aprende de las correcciones**: `claveComercio()` toma las 2 primeras palabras sin números, así "FRAVEGA CUOTA 03/06" y "FRAVEGA CUOTA 04/06" son el mismo comercio. Verificado que el segundo resumen ya acierta solo.
- **Nunca importa sin revisión**: pantalla con checkbox, monto editable y categoría editable por fila. Los duplicados (misma fecha + monto + descripción) vienen destildados.
- **Fallback de texto pegado** para resúmenes escaneados o protegidos, que pdf.js no puede leer.

**Pendiente de afinar con un resumen real del usuario** — el lector se probó con formatos sintéticos (Galicia/Santander/Mercado Pago) y con un PDF generado a mano, no con uno suyo. Al 2026-08-14 todavía no lo mandó.

### Paridad con Plan de Retiro (revisado 2026-08-14)

El usuario preguntó si acá estaban los cambios que se habían hecho en el repo hermano. **No se traspasan solos entre conversaciones**: hay que ir a leer `Plan-retiro-dashboard/index.html`. De esa comparación salió:

- ✅ **Deslizar con el dedo para cambiar de pestaña** — portado el 2026-08-14. `SECCIONES = ['inicio','presupuesto','gastos','historico','proyeccion']`, umbral de 60px, y exige que el gesto sea claramente horizontal (`|dx| > |dy| * 1.5`). `gestoBloqueado()` lo desactiva sobre canvas, inputs, selects, textareas, el sidebar y cualquier contenedor con scroll horizontal (la tira de KPIs y las tablas). `hayAlgoAbierto()` lo desactiva con el sidebar o un modal abierto. Allá usaban `.active`; acá las vistas se muestran con `style.display` y se llama a `setTab()` directo.
- ✅ **Tooltip del gráfico en modo oscuro** — arreglado el 2026-08-14. Los colores del globito estaban fijos en modo claro y `updateChartColors()` solo tocaba ejes y grilla, así que en oscuro quedaba blanco. En el repo hermano esto no pasa porque leen variables CSS con `getCSSVar()`.
- ⏳ **Haptics** (vibración al mover sliders, `navigator.vibrate` + switch nativo en iOS) — existe en Plan de Retiro (`hapticTick()`, ~línea 1435), **no está acá**. Pendiente de decisión del usuario.
- ⏳ **Bottom sheet con swipe-to-dismiss** (arrastrar el panel de ajustes hacia abajo para cerrarlo) — existe en Plan de Retiro (~línea 1490), **no está acá**. Pendiente de decisión del usuario.

## Ideas pendientes

Confirmadas por el usuario (2026-08-07). Las tres primeras ya están hechas:

1. ~~**Alertas de categoría**~~ — ✅ hecho el 2026-08-13 (ver sección arriba).
2. ~~**Detección de gastos recurrentes**~~ — ✅ hecho el 2026-08-13 (ver sección arriba).
3. ~~**Vista "esta semana"**~~ — ✅ hecho el 2026-08-13 (ver sección arriba).

Inspiradas en plata.wtf, más a futuro:

4. Importar PDF/resumen de tarjeta con categorización automática.
5. Informes personalizables por período (gastos/ingresos/patrimonio/flujo de caja).
6. Pagos habituales con recordatorio de vencimiento.
7. Concepto de "cuentas" (efectivo/banco/tarjeta/cripto) con transferencias — el cambio más grande de arquitectura, para el final.

Transversales (podrían vivir en cualquiera de las dos apps):

- Racha/hábito (gamificación liviana) por cargar datos seguido.
- Onboarding con sentido: armarle un presupuesto sugerido a partir de sus ingresos, en vez de arrancar en blanco.

**Prioridad general**: funcionalidad primero. El pulido visual (animaciones, transiciones, efectos hover/3D) el usuario lo quiere **casi al final**, como una pasada dedicada.

## Regla de trabajo

- Cambios chicos/aditivos van como funciones nuevas sin tocar el JS original.
- Cambios de lógica de negocio real sí se editan directo, pero siempre probados antes de dar por terminado un cambio. (En el repo hermano se usaba una batería de ~11 archivos de test con jsdom; **este repo no tiene tests todavía** — verificar abriendo la app en el navegador.)
- Commitear y pushear solo cuando el usuario lo pide.
