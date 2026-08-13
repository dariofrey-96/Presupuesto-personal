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

## Ideas pendientes

Confirmadas por el usuario (2026-08-07), en orden de prioridad. La #1 ya está hecha:

1. ~~**Alertas de categoría**~~ — ✅ hecho el 2026-08-13 (ver sección arriba).
2. **Detección de gastos recurrentes**: si el mismo gasto se repite ~2-3 meses seguidos, sugerir marcarlo como recurrente. Debe **preguntarle al usuario** (no automático) y permitir **editar el monto** después — con la inflación argentina el importe cambia seguido.
3. **Vista "esta semana"** como pantalla de inicio: un resumen rápido al abrir la app en vez de una tabla vacía, para dar un motivo de entrar seguido.

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
