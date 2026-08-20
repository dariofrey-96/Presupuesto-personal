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

### Objetivo comercial (declarado 2026-08-16)

El usuario **quiere monetizar esto**: que otras personas se lo descarguen y pagar por ello. Consultó con otra IA sobre qué debería tener una app así y trajo una lista (conciliación bancaria, Open Finance, MFA/FIDO2, Zero Trust, AES-256, HSM, DORA). Postura acordada:

- **La lista de seguridad describe cómo proteger un servidor, y esta app no tiene servidor.** Hoy es más privada que cualquier fintech porque no hay nada que hackear. Seguir esa lista implicaría construir el backend primero — o sea, crear el riesgo y después defenderlo. Todo eso pasa a ser obligatorio **solo si** se convierte en producto multiusuario (ahí además aplica la ley argentina de datos personales 25.326).
- **Los riesgos reales de hoy son otros**: ~~si limpia el navegador sin exportar el backup pierde todo~~ (resuelto el 2026-08-20 con la copia automática en GitHub), y cualquiera con acceso a su compu ve los datos.
- **Open Finance / conexión al banco no es viable en Argentina** para un proyecto personal. El lector de PDF no es un plan B: es *la* solución local.
- **Diferencial real frente a Fintonic/YNAB/Mobills**: entender cuotas argentinas ("8 de 10" reubicada al mes del resumen), pagos por trimestre, inflación y dualidad peso/dólar.
- **Consejo dado**: la app ya está publicada y es gratis de mantener. Validar con 10-20 personas reales antes de invertir en backend; el orden inverso es el error caro más común. Aclarado que esto no es asesoramiento de negocios y que no se puede predecir el éxito comercial.

### Diseño del presupuesto en dos momentos (acordado 2026-08-16, pendiente de implementar)

El usuario detectó un agujero real: **"sugerir montos según lo que gastás" no sirve el primer día**, porque no hay historial. Preguntó quién define las categorías y los montos para alguien que recién se descarga la app. Diseño acordado:

1. **Definir** (lo pone la persona, guiada): onboarding corto al abrir sin datos — cuánto entra por mes → qué categorías usa (marca de la lista) → cuánto en cada una. Solo se muestran las categorías marcadas; el resto no existe para esa persona.
2. **Ajustar** (lo propone la app, con datos): después de 1-2 meses cargados, sugerir subir/bajar el presupuesto de un rubro según el gasto real. **Nunca cambia solo, siempre pregunta.**
3. **Atajo de arranque**: importar el último resumen de tarjeta durante el onboarding y armar el presupuesto con lo gastado de verdad, para no partir de una pantalla en blanco.

## Este repo (Presupuesto Personal)

App de una sola página en pesos argentinos. Todo vive en `index.html` (~2200 líneas, HTML + CSS + JS en el mismo archivo). El repo solo tiene `README.md` e `index.html` — sin build, sin dependencias, sin backend, sin GitHub Actions.

### Navegación por mes en "Gastos del mes" (implementado 2026-08-15)

**Bug que lo motivó**: `renderGastos()` y todo lo relacionado usaban `monthKey()` (hoy) fijo, así que la pestaña solo mostraba el mes corriente. Al importar el resumen de tarjeta en agosto, los gastos caían en julio y quedaban **cargados pero invisibles** — el usuario creyó que no se había importado nada.

- Variable `mesVisible` (arranca en `monthKey()`), flechitas `‹ ›` en el encabezado y botón "Volver a hoy" que aparece solo fuera del mes actual. El nombre del mes se pinta en dorado cuando no es el actual.
- Se cambió `monthKey()` → `mesVisible` en: `renderGastos`, `removeGasto`, `closeMonth`, `computeSuperavitRealActual`, `gastosPorCatMes`, `get/setAlertaEstado`, `renderAlertasBanner`, `comprometidoFuturo`, `estadoFijos`, `agregarFijoAlMes` y el título de la tarjeta de fijos.
- **Siguen usando `monthKey()` a propósito** (hablan de hoy, no del mes mirado): `ritmoDelMes()` de la pantalla de inicio, el fallback de `parsearLineas`, y `mesesRecientes()` en la detección de recurrentes.
- `renderInicio()` guarda y restaura `mesVisible` alrededor de `pintarInicio()`: la pantalla de inicio siempre habla de hoy aunque en Gastos estés mirando julio.
- Al terminar de importar, `confirmarImport()` **salta al mes donde cayeron los gastos** y lo dice en el mensaje. Si quedaron repartidos en varios meses, los lista.

### Rediseño / simplificación (etapa 1 hecha 2026-08-16, etapa 2 pendiente)

El usuario dijo que la app tenía "demasiado relleno" y que se perdía. Se decidió simplificar **antes** de seguir agregando features, porque cada cosa nueva se apoyaba en la estructura vieja.

**De 6 botones a 2 + Ajustes.** Orden: **Resumen** (primera y por defecto) → **Gastos** → ⚙ Ajustes.

- **Se sacó Proyección** entera ("me parece al pedo"), **Esta semana** (no lo convencía) y la pestaña **Presupuesto**.
- **Se sacaron de todas las vistas**: la tira de KPIs (`#kpi-scroll`) y la barra "Distribución del ingreso" (`#dist-card`), que eran de Presupuesto y aparecían encima de todo. **Están ocultas con `display:none`, no borradas**, porque `recalc()` todavía escribe en ellas.
- **Resumen** (nuevo, `renderResumen`): chips de período (Este mes / Mes pasado / Últimos 3 / Últimos 12), tarjeta "A dónde va tu ingreso", gastado del período con comparación contra el anterior, gráfico de evolución de 12 meses, y "En qué se te va" con dona + lista. Se calcula **con los gastos cargados**, no con snapshots: ya no hace falta "cerrar" el mes.
- El gráfico y la dona copian el estilo del Plan de Retiro (línea sin puntos, `tension: 0.3`, dona `cutout: 72%`), con paleta OKLCH propia porque la de allá es para fondo oscuro.
- **Los tres fondos van separados** dentro de la distribución del ingreso (emergencia / vacaciones / inversiones), con subtotal arriba y sangría abajo. Pedido explícito del usuario.

**Lección de proceso**: se rediseñó dos veces el elemento equivocado porque había dos cosas llamadas "distribución del ingreso" en la misma pantalla (la tira vieja arriba y la tarjeta nueva abajo). **Antes de rediseñar, confirmar con el usuario cuál es el elemento** (captura o nombre exacto).

**Etapa 2 pendiente**: borrar de verdad el código muerto (vistas viejas, `chartProy`, `chartHist`, `renderInicio`/`pintarInicio`, `renderHistorico`, `closeMonth`, los sliders de proyección y las partes de `recalc()` que alimentan lo oculto) y rehacer el presupuesto según el diseño de dos momentos de arriba.

**Referencia visual para el front (que va al final)**: el usuario trajo un mockup de dashboard que le gustó — menú lateral oscuro en escritorio, tarjetas con mucho aire, un solo color de acento fuerte, bloque de saldo con gráfico grande arriba y transacciones recientes abajo.

### Pestañas

1. **Presupuesto** — sliders por categoría (salario, alquiler, servicios, supermercado, nafta, prepaga, etc.) para armar el presupuesto mensual planificado.
2. **Gastos del mes** — carga manual de gastos reales (fecha, categoría, monto) y comparación contra lo presupuestado.
3. **Histórico** — meses ya "cerrados" con `closeMonth()`, que guarda una foto del mes (ingreso, gasto real, superávit, desglose por categoría).
4. **Proyección** — evolución del salario y gastos a varios años (nominal y real), con crecimiento salarial e inflación ARS configurables.

### Dónde se guardan los datos

Todo en `localStorage` del navegador, **más una copia en GitHub** desde el 2026-08-20 (ver "Sincronización con GitHub" abajo):

- `presupuesto_params_v1` — sliders y toggles del presupuesto.
- `presupuesto_gastos_v1` — gastos cargados, agrupados por mes (`{"2026-08": [...]}`).
- `presupuesto_snapshots_v1` — cierres de mes del histórico.
- `presupuesto_tc_v1` — tipo de cambio ARS/USD cargado a mano.
- `presupuesto_theme` — tema claro/oscuro.
- `presupuesto_alertas_cfg_v1` — config de alertas de categoría (`{activas, umbral}`).
- `presupuesto_alertas_estado_v1` — qué alertas ya se mostraron y cuáles se silenciaron, por mes y categoría (`{"2026-08": {"juntadas": {nivel, mute}}}`).
- `presupuesto_sync_v1` — `updatedAt` de la versión de GitHub con la que este dispositivo quedó igual.
- `presupuesto_sync_huella_v1` — huella de los datos que llegaron a subirse de verdad.
- `presupuesto_previo_v1` — copia de lo que había antes de adoptar una versión remota.
- `planRetiro_params_v1` — **clave compartida con la app de Plan de Retiro.**
- `planRetiro_gh_token` — **clave compartida**: el token de GitHub que ya usa el Plan de Retiro.

### Integración con Plan de Retiro (importante)

Las dos apps están publicadas bajo el mismo dominio (`dariofrey-96.github.io`), así que **comparten el `localStorage` del navegador**. `syncAhorroARetiro()` toma el superávit real del mes, lo convierte a USD con el tipo de cambio cargado a mano, y lo escribe en `planRetiro_params_v1.ahorroMensual` — o sea, el ahorro mensual del plan de retiro se alimenta del presupuesto real. Al tocar esa clave, tener en cuenta que la otra app la lee.

### Categorías

Hay 22 categorías fijas en el array `GASTO_CATS` (id + label). Los ids coinciden con los ids de los sliders del presupuesto, que es como se compara "presupuestado vs. real" por categoría.

### Backup

Export/import manual a archivo JSON (`exportBackup()` / `importBackup()`), que abarca las claves de params, gastos, snapshots, tipo de cambio y alertas (`BACKUP_KEYS`). **Se deja tal cual a pedido del usuario**: sirve para sacar los datos afuera de GitHub. Desde el 2026-08-20 ya no es el único respaldo.

### Sincronización con GitHub (implementado 2026-08-20)

Último bloque `<script>` del archivo. Resuelve el riesgo real que estaba anotado en el objetivo comercial: si limpiaba el navegador sin exportar, perdía todo.

- **Dónde**: `presupuesto.json` en `dariofrey-96/Plan-retiro-datos`, el **mismo repo privado** donde el Plan de Retiro guarda la cartera, en archivo aparte (no se tocan entre sí).
- **Token**: se guarda en `planRetiro_gh_token`, compartida con el Plan de Retiro. Las dos apps se publican bajo `dariofrey-96.github.io`, así que para el navegador son el mismo sitio y comparten `localStorage`: pegar la llave en una la deja puesta en la otra.
- **La llave se pega EN ESTA APP** (grupo "Copia en GitHub" del panel de Ajustes: `pintarZonaToken()` / `guardarToken()` / `editarToken()`). **Corrección del 2026-08-20**: al principio no había campo acá y se le dijo al usuario que fuera a pegarla a la app de Plan de Retiro. Su respuesta fue la correcta: *"no entiendo por qué está acá metido el tema del plan retiro... yo quiero abrir la app de presupuesto personal en cualquier dispositivo y ya tener todo lo que fui cargando"*. **Que dos apps compartan una clave por abajo es un detalle de implementación; no puede convertirse en un paso manual en otra app.** El campo es `type="password"`, arranca vacío y nunca muestra la llave guardada; guardar vacío la borra.
- **Botón "👁 Ver la llave"** (`verLlave()` / `copiarLlave()`), visible sólo si ya hay una puesta. Sin esto no había forma de pasar la llave de un dispositivo a otro: las dos apps la guardan oculta, así que para el celular había que crear un token nuevo en GitHub. Se muestra sólo cuando el usuario lo pide, en un campo de sólo lectura, con aviso de que es como una contraseña.
- **Formato**: `{app, updatedAt, data}` donde `data` es exactamente lo que arma `exportBackup()` a partir de `BACKUP_KEYS`. Un solo formato para las dos cosas.
- **Cuándo sube**: `programarSync()` se engancha a `saveParams`, `saveGastosAll`, `saveTC`, `saveAlertasCfg`, `saveRecurrentes` y `saveReglasImport` reemplazando la referencia global (no se tocó ninguna función original). Retardo de 2s para no hacer un commit por tecla, y no sube si la huella de los datos no cambió. `saveAlertasEstado` queda **afuera a propósito**: es ruido de pantalla, viaja pegado al próximo cambio real.
- **Al abrir** (`arrancarSync()`): primero `restaurarPresupuestoSiEstaVacio()`, si no `traerCambiosDeOtroDispositivo()`, y al final sube si quedaron cambios sin subir. Una sola de las tres hace algo.
- **Cartel de estado**: puntito de color en el header (verde/rojo/gris, con la etiqueta visible sólo en escritorio) y línea completa + botón "↻ Sincronizar ahora" en el grupo "Copia en GitHub" del panel de Ajustes. El botón manual primero baja lo de otro dispositivo y, si no hay nada más nuevo, sube lo de acá.
- **Es "gana el último que sube"**, decidido así a propósito. No se intenta resolver conflictos.

**Trampas que ya nos habían mordido en el repo hermano y están cubiertas acá:**

- **Al adoptar la versión remota NO se vuelve a subir.** Si se subiera, se generaría un `updatedAt` nuevo, el otro dispositivo lo bajaría y lo volvería a subir: se pisan en círculo para siempre.
- **`presupuesto_sync_v1` se anota tanto al subir como al adoptar.** Sin esa marca no hay forma de distinguir "otro dispositivo cambió algo" de "esto lo cambié yo".
- **Antes de pisar lo local se guarda `presupuesto_previo_v1`**, por si había algo editado sin conexión que no llegó a subirse.
- **Un 404 al leer significa "todavía no existe", no es error** y no muestra cartel rojo.
- **Se pide `Accept: application/vnd.github.raw`**: el JSON de la API deja de devolver el archivo arriba de 1 MB.
- **La restauración actúa sólo si no hay absolutamente nada local** (`hayDatosLocales()` mira todas las `BACKUP_KEYS`), si no pisaría datos buenos.
- **Nunca se sube un payload vacío.** Si el dispositivo está vacío y encima falla la lectura, subir "nada" borraría el archivo bueno de GitHub. Por eso `subirPresupuesto()` corta si no hay datos locales.
- **`presupuesto_sync_huella_v1` cubre los cambios que nunca llegaron a subirse** (editados sin conexión, o un backup manual restaurado): al abrir, si la huella de los datos no coincide con la última subida, se sube. Sin eso, un cambio hecho offline quedaba sólo en ese dispositivo para siempre.
- **Un 409** (otro dispositivo escribió entre el GET del sha y el PUT) reintenta solo a los 2s.

**El primer encuentro pregunta (agregado 2026-08-20, mismo día).** Lo detectó el usuario apenas se publicó: él carga todo **desde el celular**, pero sin querer sincronizó desde la computadora, así que `presupuesto.json` quedó con los datos flacos de la compu. Al abrir el celular, `traerCambiosDeOtroDispositivo()` iba a ver una versión remota que nunca había visto (marca en `null`) y **la iba a adoptar, reemplazando meses de carga**. Quedaba recuperable en `presupuesto_previo_v1`, pero él no lee código: no tenía forma de volver.

- **Si no hay marca Y hay datos locales, se pregunta en vez de adivinar.** El cartel muestra el dato que decide: *"ACÁ tenés: 55 gastos en 5 meses, ingreso de $1.5M / EN GITHUB hay: 1 gasto en 1 mes, ingreso de $500K"*, con la fecha de la versión remota. Cancelar = quedarse con lo del dispositivo y subirlo. **Pasa una sola vez por dispositivo**: después queda la marca y no molesta más.
- **Botón "⟲ Recuperar lo que había antes"** en Ajustes, visible sólo si existe `presupuesto_previo_v1`, y **rotulado con lo que hay adentro** ("55 gastos en 5 meses"). Al usarlo **intercambia**: guarda lo actual como copia previa, así se puede ir y volver. Sube lo recuperado a GitHub.
- **El botón manual ya no puede subir un dispositivo vacío.** Antes `subirPresupuesto(manual=true)` salteaba esa guarda, así que un clic sin querer en una compu en blanco borraba el archivo bueno. Ahora `sincronizarAhora()` con la app vacía **restaura** en vez de subir.
- **Si los datos remotos son idénticos a los locales**, sólo se anota la marca: no pregunta, no guarda copia previa y no repinta. La comparación va por `huellaDeDatos()`, que ordena siempre por `BACKUP_KEYS` — comparar el JSON crudo fallaba cuando las mismas claves venían en otro orden.

**Probado con un GitHub falso en memoria** (se intercepta `fetch` a `api.github.com` y se responde con un archivo simulado, sha incluido). Casos verificados: primer encuentro eligiendo quedarse con el dispositivo (sube lo de acá) y eligiendo traer lo de GitHub (aparece el botón de recuperar, y al usarlo vuelve todo y se sube); que no vuelva a preguntar en la segunda apertura; dispositivo vacío que toca "Sincronizar ahora" y **baja** en vez de borrar lo remoto; datos idénticos con las claves en otro orden; dispositivo vacío que restaura; dispositivo con datos viejos que adopta lo nuevo y guarda la copia previa; **abrir dos veces seguidas no toca nada** (1 GET, ningún PUT, sha remoto intacto); cambio local que sube tras el retardo; guardar sin cambios reales que no genera commit; cambio offline que se sube al abrir de nuevo; 404 (crea el archivo, sin error); 409 (reintenta y termina bien); sin token (no hace ninguna petición, la app sigue andando y el cartel dice "los datos quedan sólo en este dispositivo"); y el backup manual, que sigue exportando el JSON igual que siempre.

**Para volver a probarlo**: no hace falta el token real. Interceptar `fetch` como arriba y llamar a `arrancarSync()` a mano después de armar el estado.

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

**Nota para verificar en el navegador**: si el panel de preview no está compositando, **`requestAnimationFrame` nunca corre**. Consecuencias al medir:

- Las transiciones CSS quedan congeladas y `getComputedStyle` del `body` devuelve el color viejo al cambiar de tema.
- **Los gráficos de Chart.js calculan la geometría pero no se pintan**: `getImageData` da 0% de área dibujada. Una dona se ve especialmente afectada porque todo su barrido está animado. Para medir de verdad: `chart.options.animation = false; chart.update('none'); chart.draw();` y recién ahí leer los píxeles.

Nada de esto es un bug de la app — en un navegador real anda.

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

**Afinado contra el resumen real (2026-08-15).** El usuario pasó su Resumen Visa de Santander (cierre 30/07/26). El lector v1 fallaba en 6 cosas; todas corregidas. Lo que enseñó ese resumen, que vale para cualquier tarjeta argentina:

- **Las cuotas traen la fecha de la compra original**, no la del período. "Perfumerías Juleriaque 8 de 10" figuraba con fecha 21/12/25 y se cargaba en diciembre 2025, cuando esa cuota se cobra ahora. Eran $463K de $577K yendo a meses que ya pasaron. Ahora `RE_CUOTA` las detecta y las reubica en el **mes del resumen**, que se deduce del mes más repetido entre los consumos que NO son cuotas (así no depende del banco).
- **Los consumos en dólares venían como pesos** (Apple U$S 2,99 → $3). Ahora `importesDeLinea()` separa por símbolo: `U$S` vs `$`. Se convierten con el TC de la app; si no hay, se usa el `tc1500,000` que el propio resumen trae en la línea del pago anterior; si tampoco, la fila viene destildada.
- **Líneas de continuación sin fecha**: el segundo cargo de Apple del mismo día no repite la fecha y se perdía. Ahora la fecha se arrastra (`fechaVigente`).
- **Números entre paréntesis son bases de cálculo, no importes**: "Iibb percep-cord 3,00%( 9394,88) $ 281,84" tomaba 9394,88. Se descarta todo lo que esté entre paréntesis.
- **El número de comprobante ensuciaba la descripción** ("Ypf urca 302447"). Se quitan los números de 5+ dígitos.
- **La letra chica del final tiene fechas y montos** y generaba movimientos fantasma. `RE_FIN_MOVIMIENTOS` corta el parseo al llegar a "Términos y condiciones".

**Regla clave que costó encontrar**: el modo de respaldo (adivinar importes sin símbolo de moneda, para bancos que no usan `$`) **solo se aplica en líneas que traen su propia fecha**. Sin esa restricción, la línea de tasas ("En pesos: 77,900 % En dólares: 0,000 %") se colgaba de la fecha arrastrada y entraba como un gasto de $78. Restringirlo por documento (¿usa `$` en alguna parte?) rompía los resúmenes mixtos; por línea con fecha propia, funcionan los tres casos.

**Validación fuerte disponible**: la suma de lo importado tiene que dar exactamente el "Total a pagar" del resumen. Con el de Santander da $576.940 vs $576.940,88 y U$S 10,04 vs U$S 10,04. **Si se toca el parser, repetir esa comparación** — es la prueba que detecta tanto lo que falta como lo que sobra.

Categorías nuevas en el diccionario a partir de comercios reales: peajes (Caminos de las Sierras), perfumerías (Juleriaque), electro (Frávega), universidades ("univ "), e impuestos/percepciones → `cnc`.

**El PDF del usuario no se commitea nunca.** Para probar se copia a `_tmp-resumen.pdf` (está en `.gitignore`), se sirve por el preview, y se borra al terminar.

### Paridad con Plan de Retiro (revisado 2026-08-14)

El usuario preguntó si acá estaban los cambios que se habían hecho en el repo hermano. **No se traspasan solos entre conversaciones**: hay que ir a leer `Plan-retiro-dashboard/index.html`. De esa comparación salió:

- ✅ **Deslizar con el dedo para cambiar de pestaña** — portado el 2026-08-14. `SECCIONES = ['inicio','presupuesto','gastos','historico','proyeccion']`, umbral de 60px, y exige que el gesto sea claramente horizontal (`|dx| > |dy| * 1.5`). `gestoBloqueado()` lo desactiva sobre canvas, inputs, selects, textareas, el sidebar y cualquier contenedor con scroll horizontal (la tira de KPIs y las tablas). `hayAlgoAbierto()` lo desactiva con el sidebar o un modal abierto. Allá usaban `.active`; acá las vistas se muestran con `style.display` y se llama a `setTab()` directo.
- ✅ **Tooltip del gráfico en modo oscuro** — arreglado el 2026-08-14. Los colores del globito estaban fijos en modo claro y `updateChartColors()` solo tocaba ejes y grilla, así que en oscuro quedaba blanco. En el repo hermano esto no pasa porque leen variables CSS con `getCSSVar()`.
- ✅ **Haptics** (`hapticTick()`, `navigator.vibrate(8)` con freno de 35ms) — portado el 2026-08-14 sobre todos los `input[type=range]` del `aside`. **En iOS no vibra**: no hay API de vibración web y no hay forma de engancharla a un arrastre continuo. En el repo hermano resolvieron los botones +/- con un `<input type="checkbox" switch>` nativo invisible encima del botón, pero acá no hay botones +/-, solo sliders — así que esa parte no aplica.
- ✅ **Bottom sheet con swipe-to-dismiss** — portado el 2026-08-14. Se agarra de `.sheet-handle` o `.sheet-title-row` (no de la lista de parámetros, que necesita su scroll). Cierra con arrastre > 18% de la altura de pantalla o con velocidad. Guarda extra que no está en el repo hermano: `getComputedStyle(aside).position !== 'fixed'` corta el gesto en escritorio (≥900px), donde el panel es `static` y no es un cajón.

**Cuidado al verificar gestos con herramientas**: medir justo después de un `resize_window` puede leer un `window.innerHeight` viejo y hacer que el umbral de cierre dé cualquier cosa. Repetir la prueba varias veces antes de creerle a un resultado raro.

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
