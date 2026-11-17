# 03 — Frontend y UX de un CRM: cómo se construyen de verdad las interfaces

**Alcance:** patrones de interfaz de CRMs comerciales y de código abierto, elección de librerías y stack mínimo viable para un equipo de una persona.
**Fecha:** 16 de noviembre de 2026.
**Convención de evidencias:** `[verificado + URL]` = comprobado en documentación, repositorio o ficha de paquete consultada durante esta investigación. `[inferido]` = deducción razonada a partir de evidencia parcial o de práctica extendida, sin fuente directa citada.

---

## 0. Resumen ejecutivo

1. Un CRM no es "una app con tablas": es un **sistema de registro con vistas guardadas**. La list view no es una pantalla más, es el producto. Todo lo demás cuelga de ella.
2. Los cinco productos serios (Salesforce, HubSpot, Pipedrive, Attio, EspoCRM) convergen en las mismas cinco pantallas y en el mismo orden de prioridad de construcción.
3. La diferencia entre un CRM que se siente rápido y uno que se siente lento **no es React**: es si la tabla pagina en el servidor con keyset, virtualiza filas y trata los filtros como estado persistido, no como `useState` local.
4. La UI dirigida por metadatos es real y todos la tienen, pero **nadie la implementa como motor genérico desde el día uno**: empiezan con un registro de campos (field registry) y layouts en JSON, y crecen hacia metadatos en base de datos.
5. Casi todos son SPA. No es moda: es que el estado de navegación (vista, filtros, columnas, selección, panel lateral abierto) tiene que sobrevivir a la navegación sin recargar, y el CRM pasa horas abierto en una pestaña.

---

## 1. Las cinco pantallas que todo CRM tiene

### 1.1 List view (lista / índice de registros)

**Qué resuelve:** "¿qué tengo pendiente y cómo lo segmento?" Es la pantalla de entrada al producto y la que más tiempo ocupa en pantalla.

**Qué información va primero (jerarquía real):**

| Orden | Elemento | Por qué |
|---|---|---|
| 1 | Nombre del registro (columna primaria, enlace) | Es el ancla cognitiva; siempre visible, siempre congelada al hacer scroll horizontal |
| 2 | Etapa / estado | Es el eje de decisión comercial |
| 3 | Importe o valor | Prioriza sin abrir nada |
| 4 | Propietario / asignado | Determina si "me toca" |
| 5 | Fecha relevante (cierre, última actividad) | Ordena la urgencia |
| 6 | Resto de columnas configurables | Secundarias, ocultables |

**Acciones a un clic:** cambiar de vista guardada, ordenar por columna, abrir filtros, editar celdas en línea, seleccionar filas y ejecutar acción masiva, crear registro, exportar la vista.

**Qué se deja fuera a propósito:** el detalle completo del registro (va en panel lateral), los campos calculados costosos, los historiales y las notas. Regla que se repite: **la fila es un resumen escaneable, no un formulario aplanado**. Salesforce y HubSpot limitan la densidad de columnas por ese motivo.

### 1.2 Tablero kanban por etapa

**Qué resuelve:** "¿cómo fluye el pipeline?" Complementa la lista; no la sustituye.

**Qué va primero:** nombre del registro + importe en la tarjeta; **el total agregado por columna** en la cabecera (suma de importes y conteo). Ese agregado es lo que convierte un kanban decorativo en una herramienta de previsión.

**Acciones a un clic:** arrastrar entre etapas (con actualización optimista y deshacer), filtrar por propietario, colapsar columnas, crear tarjeta en la columna.

**Qué se deja fuera:** edición de campos secundarios en la tarjeta (solo 3–4 campos), y en muchos productos, el reordenamiento manual dentro de la columna (se ordena por el criterio de la vista).

> HubSpot unifica esto: una *saved view* puede tener **tipo de vista** tabla o tablero (y gantt), compartiendo los mismos filtros y orden `[verificado + https://knowledge.hubspot.com/records/create-and-manage-saved-views]`. Ese es el patrón correcto: **la vista es el objeto, el layout es un atributo de la vista**.

### 1.3 Ficha de detalle (record view)

**Qué resuelve:** "dame todo el contexto de esta cuenta/oportunidad y déjame actuar sin salir".

**Qué va primero:** cabecera con nombre, etapa, importe y propietario (siempre visible al hacer scroll); luego un bloque de campos clave en dos columnas; luego asociaciones (contactos, negocios, tickets); a la derecha o debajo, la actividad reciente.

**Acciones a un clic:** editar en línea (no "modo edición" global), registrar llamada/nota/tarea, cambiar etapa, enviar correo, avanzar a la siguiente registro de la lista de origen.

**Qué se deja fuera:** la lista completa de campos de sistema (created by, ids), el histórico entero (se pagina), y las acciones destructivas (van a un menú "⋯" con confirmación).

**Patrón clave:** el detalle debe abrirse **en panel lateral (side peek) sobre la lista**, no como página completa, para no perder la posición ni el contexto. Notion lo hace explícito: *side peek*, *center peek* o página completa, configurable por vista, con side peek por defecto en tabla/tablero/lista/timeline `[verificado + https://www.notion.com/help/views-filters-and-sorts]`.

### 1.4 Línea de tiempo de actividad (timeline / stream)

**Qué resuelve:** "¿qué ha pasado con esto y qué dijimos?" Es memoria institucional, y es el principal motivo por el que la gente deja de usar el CRM (si no registra el contexto, no sirve).

**Qué va primero:** eventos ordenados por fecha descendente, agrupados por día, con **icono por tipo** (correo, llamada, reunión, nota, cambio de etapa, cambio de campo). Cada evento: autor + hora + resumen de una línea; el detalle se expande.

**Acciones a un clic:** escribir nota (editor con markdown), adjuntar archivo, filtrar por tipo de evento, mencionar a un compañero, fijar.

**Qué se deja fuera:** los cambios de campo triviales se agrupan ("actualizó 4 campos") para no inundar; los eventos masivos de sistema se ocultan por defecto.

> EspoCRM llama a esto *Stream* y tiene su propio `streamDefs` en los metadatos `[verificado + https://docs.espocrm.com/development/metadata/]`.

### 1.5 Panel de métricas (dashboard)

**Qué resuelve:** "¿cómo vamos?" Dirección y gestión, no operación diaria.

**Qué va primero:** 3–5 KPIs en la parte superior (pipeline total, ganado del mes, tasa de conversión, actividades vencidas), luego gráficos de tendencia y luego rankings por propietario/etapa.

**Acciones a un clic:** cambiar periodo, filtrar por equipo, hacer clic en un segmento para **profundizar hacia la lista filtrada** (drill-down). Si un gráfico no lleva a una lista, es decoración.

**Qué se deja fuera:** tablas crudas y detalle por registro. Los dashboards son componentes dispuestos en una rejilla arrastrable (Twenty usa `react-grid-layout` `[verificado + https://raw.githubusercontent.com/twentyhq/twenty/main/packages/twenty-front/package.json]`; EspoCRM tiene "Dashlets" y `gridstack` `[verificado + https://github.com/espocrm/espocrm]`).

### 1.6 Tabla comparativa y orden de construcción

| Pantalla | Pregunta que responde | Densidad | Frecuencia de uso | Dificultad técnica | Prioridad |
|---|---|---|---|---|---|
| List view | ¿Qué tengo y cómo lo filtro? | Alta | Cada sesión, decenas de veces | Alta (grid + server-side + saved views) | **1** |
| Ficha de detalle | ¿Qué sé de este registro? | Media | Constante | Media | **2** |
| Timeline | ¿Qué ha pasado? | Media | Constante | Media-baja (pero requiere modelo de eventos) | **3** |
| Kanban | ¿Cómo fluye el pipeline? | Baja | Diaria en ventas | Media (drag & drop + agregados) | **4** |
| Métricas | ¿Cómo vamos? | Baja | Semanal | Media (agregaciones SQL + gráficos) | **5** |

**Justificación del orden:** la list view define el modelo de datos (columnas = campos tipados), la ficha define el motor de formularios, la timeline obliga a un modelo de eventos append-only, el kanban reutiliza list view + un campo de agrupación, y las métricas se construyen sobre las consultas que ya existen. Si empiezas por el dashboard, construyes un producto de demostración.

---

## 2. La list view como pieza central

### 2.1 Anatomía (de arriba abajo)

```
┌─ Tabs de vistas guardadas: [Mis negocios abiertos] [Cerrados Q4] [+ ...] ─┐
├─ Barra: buscador dentro de la vista │ chips de filtro activos │ ⚙ columnas ┤
├─ Cabecera de tabla: checkbox │ Nombre (fija) │ Etapa │ Importe │ ...       │
├─ Filas virtualizadas (32–40 px)                                            │
├─ Barra flotante de selección: "12 seleccionados · Etapa · Propietario · ⋯" │
└─ Pie: "1–50 de ≈ 4.812" [‹][›]  │ densidad ▾ │ exportar │                 ┘
```

### 2.2 Columnas configurables

- Se eligen de un diálogo con **checkboxes + arrastrar para reordenar**, y se aplican en bloque: HubSpot literalmente documenta "hover sobre el borde derecho → icono editar columnas → seleccionar checkboxes → arrastrar → Apply" `[verificado + https://knowledge.hubspot.com/records/create-and-manage-saved-views]`.
- **Anchos persistentes** por vista y por usuario, no globales.
- **Congelar** la primera columna (y opcionalmente hasta una columna N): Notion expone "Freeze up to column" `[verificado + https://www.notion.com/help/views-filters-and-sorts]`.
- Regla práctica: si el usuario puede configurar columnas, la fila **no puede tener altura variable** — la virtualización se rompe. Altura fija por densidad (compacta 32 px / cómoda 40 px).
- Salesforce resuelve esto de forma declarativa: en `lightning-datatable` las columnas son un array de objetos `{ label, fieldName, type, editable }`, donde `type` decide el renderizador (`text`, `phone`, `email`, `currency`, `date`, `url`, `action`…) y admite **tipos de dato personalizados** `[verificado + https://raw.githubusercontent.com/trailheadapps/lwc-recipes/main/force-app/main/default/lwc/datatableInlineEditWithUiApi/datatableInlineEditWithUiApi.js y .../datatableCustomDataType/]`.

### 2.3 Filtros guardados (saved views) — el patrón que casi nadie copia bien

Lo que hace HubSpot, y que conviene replicar casi literalmente `[verificado + https://knowledge.hubspot.com/records/create-and-manage-saved-views]`:

| Pieza | Comportamiento |
|---|---|
| Tipo de vista | Una vista guardada define filtros + orden + columnas + **tipo de visualización** (tabla, tablero, gantt) |
| Visibilidad | `Privada` (solo yo + super admins), `Equipo`, `Todos`, `Personalizada` |
| Roles de la vista | `Owner` (edita y **publica** cambios globales), `Collaborator` (edita su copia, no publica; aparecen puntos azules indicando divergencia), `Member` (solo lectura; puede tocar paginación, altura de fila y rayado) |
| Publicación | Los cambios del owner generan un aviso "Update available" a los demás |
| Tabs | Máximo ~5 pestañas fijadas; arrastrar para reordenar; la más a la izquierda es la vista por defecto |
| Clonar | Duplicar una vista existente como punto de partida |

Modelo de datos mínimo:

```sql
view(id, tenant_id, object_type, name, is_default,
     config_json,      -- {columns:[{field,width}], sort:[{field,dir}], filters:{...}, layout:'table'|'board', group_by}
     visibility,       -- 'private' | 'team' | 'everyone' | 'custom'
     owner_id, version, published_at)
view_share(view_id, principal_type, principal_id, role)  -- owner | collaborator | member
```

Dos decisiones que ahorran meses de soporte:
1. **Separar "mi ajuste local" de "la configuración de la vista"**: paginación, densidad y rayado son preferencias de usuario; filtros y columnas son configuración de la vista. HubSpot ya las distingue por rol.
2. **Versionar la vista** (`version`, `published_at`) para poder avisar de cambios sin sobrescribir a quien está trabajando.

### 2.4 Orden (sort)

- Orden **múltiple** (campo 1, desempate por campo 2, y siempre desempate final por `id` para que la paginación sea determinista — sin esto, las filas se duplican o desaparecen entre páginas).
- El orden debe ser **de servidor** en cuanto la tabla pasa de ~2.000 filas. Ordenar en cliente exige tener todas las filas, que es exactamente lo que no quieres.
- Orden por defecto sensato por tipo de objeto: negocios → `fecha de cierre asc` (lo que se va a vencer primero); contactos → `última actividad desc`.

### 2.5 Paginación vs scroll infinito

| Criterio | Paginación numerada | Scroll infinito | **Recomendado: "cargar más" con cursor** |
|---|---|---|---|
| Coste en servidor | `OFFSET` alto degrada linealmente | Igual si es offset | `WHERE (sort,id) > (…)` con índice → coste constante |
| Recuperar posición | Sí | No (se pierde al volver) | Sí, si guardas cursor de la página |
| Saber cuánto falta | Sí | No | Parcial ("≈ 4.800") |
| Accesibilidad / teclado | Buena | Mala (contenido que aparece solo) | Buena |
| Impresión / export | Sí | No | Sí |

**Recomendación:** botón "cargar más" o scroll infinito **con** paginación real por cursor y un contador aproximado. El scroll infinito puro es aceptable en un feed, no en un registro de negocio: si el usuario hace clic en un registro y vuelve, debe aterrizar donde estaba.

### 2.6 Selección múltiple y acciones en lote

- Checkbox por fila + checkbox de cabecera con estado **indeterminado** (no binario).
- **Tres estados de selección, no dos:** *nada* → *N filas visibles* → *"todos los N que coinciden con el filtro"*. El tercer estado es obligatorio: nadie selecciona 50.000 checkboxes. Se implementa enviando al servidor **la definición del filtro** (o un `selection_id` firmado), no una lista de ids.
- Barra de acciones flotante que aparece solo con selección activa, con 3–5 acciones y un menú "⋯" para el resto. Acciones típicas: cambiar propietario, cambiar etapa, añadir a campaña/lista, exportar, borrar (con confirmación que exige escribir el número de registros si son >100).
- **Ejecución asíncrona con job y progreso** cuando el lote supera ~500 registros: nunca bloquees la UI esperando 5.000 updates.
- Salesforce ilustra el extremo opuesto y sirve de advertencia: la edición en lote de `lightning-datatable` dispara **una petición por registro** en paralelo con `Promise.all` `[verificado + https://raw.githubusercontent.com/trailheadapps/lwc-recipes/main/force-app/main/default/lwc/datatableInlineEditWithUiApi/datatableInlineEditWithUiApi.js]`. Funciona con 20 filas, no con 2.000. Un endpoint `PATCH /records/bulk` es obligatorio en un CRM propio.

### 2.7 Edición en línea

Patrón canónico, tomado de la implementación de Salesforce LWC:

1. La tabla mantiene `draftValues`: un array de cambios pendientes por fila.
2. El usuario edita varias celdas → no se envía nada.
3. Al guardar, se convierten los drafts en payloads, se envían, se limpian los drafts y se refresca la fuente de datos (`refreshApex`) `[verificado + misma URL]`.
4. Errores: por fila, no globales. Una celda inválida no debe tumbar las otras 19.

Reglas de UX:
- Tipos con renderizador propio: fecha (date picker), importe (con máscara y moneda del tenant), etapa (select con color), referencia (buscador de registros), usuario (avatar + autocompletar).
- **Nunca guardar en `onBlur` sin señal visual y sin deshacer.** Es la causa número uno de corrupción de datos percibida.
- Actualización optimista: pinta el valor, marca la celda como "guardando" (punto o borde), y si falla revierte con un toast accionable ("Reintentar").

### 2.8 Cómo se comporta con 50.000 filas

Presupuesto concreto, en orden de impacto:

| # | Técnica | Efecto |
|---|---|---|
| 1 | **Paginación keyset en servidor** (50 por página) | Solo viajan 50 filas; el DOM nunca ve 50.000 |
| 2 | **Virtualización de filas** (solo ~25–40 filas vivas en DOM) | DOM constante; 50k o 500k es lo mismo |
| 3 | **Índices compuestos** `(tenant_id, deleted_at, <campo de orden>, id)` | El orden y el filtro no escanean la tabla |
| 4 | **Contador aproximado** (`pg_class.reltuples`, o contador cacheado) en vez de `COUNT(*)` | Evita el `Seq Scan` completo en cada carga |
| 5 | **Virtualización de columnas** (si hay >15) | Menos nodos en el hot path |
| 6 | **Debounce 250–350 ms** en filtros de texto + `AbortController` para cancelar peticiones obsoletas | Evita carreras y carga inútil |
| 7 | **Estabilidad de referencia** (`React.memo` por fila, `getRowId` estable, no recrear el array de columnas) | Evita re-render de toda la tabla al editar una celda |
| 8 | **Altura de fila fija** | La virtualización con alturas dinámicas exige medición y produce saltos |

Números de referencia de un benchmark público de 6 grids sobre 50.000 filas, con configuración agresiva (sin paginación, sin animaciones, virtualización activa): `react-data-grid` reportó **72,3 KB gzip, 367,6 ms hasta listo, 17,3 MB de heap, ~119 FPS en scroll continuo**, y quedó primero en la mayoría de categorías; AG Grid y MUI X DataGrid quedaron a 55–57 FPS en scroll horizontal `[verificado (fuente secundaria, no auditoría propia) + https://itnext.io/i-benchmarked-6-react-data-grids-on-50-000-rows-the-real-winner-was-the-tradeoff-d447fbf5ecde]`. Conclusión transferible, más allá de los números: **todos los grids serios virtualizan; la diferencia está en cuánto DOM mantienen vivo y en el coste de la capa de datos**.

Advertencia de arquitectura: un grid "acotado" (con altura de contenedor definida) virtualiza; si le quitas los límites de alto/ancho, muchos grids renderizan todo el eje `[verificado + https://www.infragistics.com/blogs/best-react-data-grid-for-large-datasets-performance-guide]`. El layout con `height: 100%` y `overflow` contenido es parte del diseño de rendimiento, no un detalle de CSS.

### 2.9 Librerías: comparativa con versión, licencia y madurez (noviembre 2026)

Versiones y licencias consultadas en el registro de npm `[verificado + https://registry.npmjs.org/<paquete>/latest]`.

| Librería | Versión | Licencia | Modelo | Quién la usa | Veredicto |
|---|---|---|---|---|---|
| **@tanstack/react-table** | 9.3.1 | MIT | **Headless** (lógica de tabla, sin DOM ni estilos) | Muy extendida; base de Material React Table y de la mayoría de "data grids propios" | La mejor opción para un CRM a medida: control total del markup (necesario para densidad y accesibilidad) y coste cero. **No virtualiza ni pagina sola**: se combina con `@tanstack/react-virtual` 3.14.26 (MIT) |
| **react-data-grid** (adazzle) | 7.0.0-beta.64 | MIT | Grid "completo" con edición tipo hoja de cálculo | **Twenty usa `7.0.0-beta.59`** `[verificado + package.json de twenty-front]` | Excelente rendimiento y el mejor perfil de bundle del benchmark, pero **sigue en beta tras años**: asumes riesgo de API. Aun así, es la ruta más corta a "edición inline + teclado tipo Excel" |
| **ag-grid-community** | 36.3.0 | MIT (Enterprise comercial) | Grid completo, muy maduro | Estándar empresarial | Trae casi todo, pero lo que un CRM necesita para 50k filas (agrupación, pivot, master-detail, transacciones del modelo server-side, set filter, tool panel de columnas) **está en Enterprise** `[verificado + https://www.ag-grid.com/license-pricing/]`. Sin licencia te quedas justo en la frontera |
| **@mui/x-data-grid** | 9.18.0 | MIT (Pro/Premium comercial) | Grid completo con estilos MUI | Ecosistema MUI | Filtros múltiples, resize y pinning de columnas son **Pro**; agrupación y export a Excel, **Premium** `[verificado + https://mui.com/x/introduction/licensing/]`. Además te ata a Material Design, que es lo contrario de la densidad visual de un CRM |
| **handsontable** | 18.1.2 | **Propietaria** (`SEE LICENSE IN LICENSE.txt`) | Hoja de cálculo | Edición tipo Excel en banca/ops | Solo tiene sentido si el producto *es* una hoja de cálculo. Licencia no libre: descartada por defecto |
| **Glide Data Grid** | — | MIT | Canvas (no DOM) | Nicho, alto rendimiento | Rendimiento brutal, pero accesibilidad y selección de texto sufren al no ser DOM `[inferido]` |

**Por qué cada uno elige lo que elige:**
- Los que priorizan **identidad visual y densidad** (Twenty, Attio) construyen o usan grids de bajo nivel, porque un grid "con estilos incluidos" impone su sistema de diseño.
- Los que priorizan **time-to-market** eligen grid completo (AG Grid, MUI X) y aceptan la licencia o la menor personalización.
- Nadie serio usa una `<table>` con `map()` por encima de ~200 filas.

---

## 3. Formularios y UI dirigida por metadatos

### 3.1 Qué significa "metadata-driven" en la práctica

Cuatro capas distintas que conviene no confundir:

| Capa | Qué define | Dónde vive | Ejemplo real |
|---|---|---|---|
| **Esquema de datos** | Entidades y campos: tipo, obligatoriedad, opciones, relaciones | BD / JSON de configuración | EspoCRM `entityDefs` `[verificado + https://docs.espocrm.com/development/metadata/]` |
| **Layout** | Qué campos aparecen, en qué orden, en qué panel, en qué formulario (crear/editar/detalle) | BD por rol/equipo | HubSpot: columnas y tipo de vista por vista guardada; Salesforce: page layouts y Dynamic Forms `[inferido para Salesforce]` |
| **Permisos** | Quién ve/edita qué campo y qué registro | BD, evaluado en servidor | EspoCRM `aclDefs` + `entityAcl` ("access restriction for specific fields and links") `[verificado + misma URL]` |
| **Lógica** | Visibilidad condicional, obligatoriedad condicional, valores por defecto, validaciones cruzadas | Configuración declarativa | EspoCRM *Dynamic Logic* `[verificado + https://docs.espocrm.com/administration/dynamic-logic/]` |

EspoCRM es el ejemplo más limpio y documentado de motor de metadatos: los JSON se **fusionan recursivamente** desde `application/Espo/Resources/metadata`, módulos y `custom/`, se cachean, y admiten el token `__APPEND__` para añadir valores a arrays existentes en lugar de sobrescribirlos `[verificado + https://docs.espocrm.com/development/metadata/]`. El frontend accede al mismo objeto de metadatos que el backend (`this.getMetadata().get('entityDefs.Account.fields')`), lo que garantiza que formulario y API no se desincronicen.

Salesforce hace lo mismo por otra vía: `lightning-record-form` recibe `objectApiName`, `recordId` y una lista de campos, y **genera el formulario a partir del layout del objeto**; el modo "dynamic" toma el layout asignado al usuario `[verificado + https://raw.githubusercontent.com/trailheadapps/lwc-recipes/main/force-app/main/default/lwc/recordFormDynamicContact/recordFormDynamicContact.js]`. Es decir: el desarrollador no escribe inputs, escribe *qué layout aplicar*.

### 3.2 Cómo se generan formularios desde el esquema (patrón recomendado)

Un **registro de campos** (field registry) que mapea tipo → componente, más un renderizador de layout:

```ts
// 1) Registro: un tipo de campo = un componente. Añadir un tipo no toca el formulario.
const FIELD_RENDERERS = {
  text:     TextField,        // input controlado
  textarea: TextAreaField,
  number:   NumberField,      // con Intl.NumberFormat
  currency: CurrencyField,    // moneda del tenant
  select:   SelectField,      // opciones desde metadatos
  multiselect: MultiSelectField,
  date:     DateField,        // date-fns + zona del usuario
  datetime: DateTimeField,
  user:     UserPicker,       // búsqueda sobre usuarios
  relation: RecordPicker,     // búsqueda sobre otro objeto
  bool:     ToggleField,
} satisfies Record<FieldType, ComponentType<FieldProps>>;

// 2) Layout: array de secciones, cada una con campos. Viene del servidor.
type Layout = { sections: { title: string; columns: 1 | 2; fields: string[] }[] };

// 3) Render: el formulario no conoce los campos, solo los tipos.
function RecordForm({ layout, schema, values, errors, caps }: Props) {
  return layout.sections.map((s) => (
    <Section key={s.title} title={s.title} cols={s.columns}>
      {s.fields
        .filter((f) => caps.visibleFields.includes(f))   // permisos: se filtran, no se deshabilitan
        .map((f) => {
          const def = schema.fields[f];
          const Field = FIELD_RENDERERS[def.type];
          return <Field key={f} name={f} def={def} readOnly={!caps.editableFields.includes(f)} />;
        })}
    </Section>
  ));
}
```

La validación se define **una vez** y se comparte con el servidor (mismo esquema Zod):

```ts
export const dealSchema = z.object({
  name:      z.string().min(1, 'Obligatorio').max(120),
  amount:    z.number().nonnegative().nullable(),
  stage:     z.enum(STAGES),
  closeDate: z.coerce.date().nullable(),
}).refine(d => !(d.stage === 'won' && d.amount == null),
          { message: 'Un negocio ganado necesita importe', path: ['amount'] });
```

### 3.3 Validación: cliente vs servidor

| Regla | Cliente | Servidor | Motivo |
|---|---|---|---|
| Formato y longitud | Sí (inmediato) | Sí (autoridad) | UX vs integridad |
| Campo obligatorio | Sí | **Sí, siempre** | El cliente se puede saltar con `curl` |
| Unicidad (email, dominio) | No (requiere consulta) | **Sí** | Solo el servidor conoce el estado global |
| Reglas de negocio (no cerrar sin importe) | Sí (feedback) | **Sí** | Puede cambiar entre carga y guardado |
| Campo oculto por permisos | Sí (no se envía) | **Sí (rechaza si viene)** | Escalada de privilegios |
| Transiciones de etapa válidas | Sí (UI) | **Sí (máquina de estados)** | El kanban arrastra, pero la API decide |

**Regla de oro:** el cliente valida para dar feedback rápido; el servidor valida porque es el único que no miente. Un CRM con validación solo en cliente es un CRM con datos sucios en tres meses.

### 3.4 Layouts por rol

- El layout no debería ser "uno por rol" hardcodeado, sino **layout asignado a un perfil**, con posibilidad de que el usuario reordene sus propios campos (preferencia local).
- Cuando alguien del equipo comercial necesita ver 5 campos distintos de los de soporte, la respuesta correcta es **un layout más**, no un `if (role === 'sales')` en el componente.
- Los campos que el rol no puede ver **no se renderizan**; los que no puede editar se muestran en modo lectura con un tooltip ("sin permiso de edición"). Mostrar un campo deshabilitado sin explicación genera tickets de soporte.

### 3.5 Cuándo NO construir un motor de metadatos

| Situación | Enfoque |
|---|---|
| ≤10 objetos, ≤15 campos cada uno, 1–2 clientes | Configuración **en código** (TypeScript) + registro de campos. Es más rápido y tiene tipos |
| Varios clientes con campos distintos, sin deploy | Metadatos **en BD**, editables por admin, con migraciones versionadas |
| El usuario final crea objetos y campos | Motor completo (el camino de Salesforce/EspoCRM: años de trabajo) |

Un equipo de una persona que empieza por el motor completo no entrega nunca la list view. Empieza por el registro de campos tipado en código y deja la puerta abierta: si el esquema vive en una tabla `field_def` y el frontend lo consume como JSON, migrar después no cuesta nada.

---

## 4. Búsqueda y navegación

### 4.1 Búsqueda global con resultados agrupados

Comportamiento esperado (convergen todos los productos):

- Un único campo, disponible en todas las pantallas (`/` o `⌘K` para enfocar).
- Resultados **agrupados por tipo de objeto**, con el conteo por grupo y "ver todos" por grupo.
- Cada resultado: nombre + un segundo dato de desambiguación (empresa del contacto, etapa del negocio, dominio).
- Los registros **vistos recientemente** aparecen con la caja vacía. Es la mejora de UX más barata y más valorada de todo el buscador.
- Búsqueda por número/importe exacto y por identificador externo, no solo por texto.

### 4.2 Ranking

Un ranking razonable para un CRM (no es un buscador web):

1. **Coincidencia exacta** de nombre/email/dominio (peso máximo).
2. **Prefijo** de la columna primaria (`nombre ILIKE 'acme%'`).
3. **Coincidencia en palabra** dentro de campos indexados (`tsvector` en Postgres, con `ts_rank` y pesos por campo: nombre 1.0, email 0.8, notas 0.2).
4. **Señales de uso**: registros abiertos recientemente por este usuario, registros propios, registros con actividad reciente.
5. Penalizar registros archivados/cerrados.

Implementación pragmática en Postgres:

```sql
-- índice GIN sobre un tsvector generado con pesos por campo
ALTER TABLE contact ADD COLUMN search tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('simple', coalesce(name,'')),  'A') ||
    setweight(to_tsvector('simple', coalesce(email,'')), 'A') ||
    setweight(to_tsvector('simple', coalesce(company_name,'')), 'B') ||
    setweight(to_tsvector('simple', coalesce(notes,'')), 'D')
  ) STORED;
CREATE INDEX contact_search_idx ON contact USING GIN (search);

SELECT id, name, ts_rank(search, websearch_to_tsquery('simple', $1)) AS rank
FROM contact
WHERE tenant_id = $2 AND search @@ websearch_to_tsquery('simple', $1)
ORDER BY rank DESC, updated_at DESC
LIMIT 5;
```

Nota: usa `'simple'` (sin stemming) o un diccionario por idioma; el stemming de español en nombres propios y marcas produce falsos negativos molestos `[inferido]`. Para búsqueda "a mitad de palabra" (ej. "cme" → "Acme"), añade `pg_trgm` con índice GIN y `similarity()`, pero mantenla fuera del camino caliente del typeahead.

### 4.3 Filtros facetados

- Facetas clásicas de un CRM: etapa, propietario, equipo, etiqueta, rango de fechas, rango de importe, origen.
- **Chips** de filtro activos, cada uno removible con una ×, más un botón "limpiar todo".
- **Filtros avanzados** con grupos anidados AND/OR: Notion permite anidar hasta **tres niveles** `[verificado + https://www.notion.com/help/views-filters-and-sorts]`. Ese límite es sensato: más profundidad no la usa nadie y multiplica los bugs del constructor de consultas.
- Contadores por faceta ("Etapa: Propuesta (12)") solo si el coste es aceptable; si no, omítelos antes que mostrar números desactualizados.
- **Toda faceta debe ser una consulta de servidor** sobre columnas indexadas. Un filtro de cliente solo funciona sobre lo cargado, y produce el bug clásico: "filtro por Cerrado y me salen 12, pero hay 400".

### 4.4 Command palette (⌘K)

Qué debe contener, en este orden de prioridad:

| Sección | Ejemplos | Nota |
|---|---|---|
| Navegación | "Ir a Negocios", "Ir a Mi pipeline", "Abrir vista Cerrados Q4" | Incluye las **vistas guardadas** del usuario: es el atajo más usado |
| Acciones | "Crear negocio", "Crear contacto", "Registrar llamada", "Importar CSV" | Ejecutables sin salir de la pantalla actual |
| Registros | Los 5 mejores resultados de la búsqueda global | Reutiliza el endpoint de búsqueda, con debounce |
| Recientes | Últimos 5 registros abiertos | Cero escritura del usuario |

Detalles que hacen la diferencia: navegación con flechas + `Enter`, `Esc` cierra, el foco vuelve al elemento anterior, y el palette **no** se abre mientras el foco está en un input de texto (o se abre solo con `⌘K`, nunca con `/`).

Librería: `cmdk` 1.1.1 (MIT) resuelve accesibilidad, filtrado difuso y teclado en ~4 KB, y es la base de facto del patrón `[verificado + https://registry.npmjs.org/cmdk/latest]`. Para el resto de atajos, `react-hotkeys-hook` 4.x (MIT) es lo que usa Twenty `[verificado + package.json de twenty-front]`.

### 4.5 Atajos de teclado imprescindibles

| Atajo | Acción |
|---|---|
| `⌘K` / `Ctrl+K` | Command palette |
| `G` luego `D`/`C`/`N` | Ir a Negocios / Contactos / Notas (navegación tipo Gmail) |
| `C` | Crear registro en el objeto actual |
| `E` | Editar el registro abierto |
| `↑` `↓` | Mover selección en la lista |
| `Enter` | Abrir el registro seleccionado |
| `X` | Marcar/desmarcar fila |
| `Esc` | Cerrar panel lateral / cancelar edición |
| `?` | Ver todos los atajos |

EspoCRM documenta su propia lista de atajos como característica de producto `[verificado + https://docs.espocrm.com/user-guide/shortcuts/]`: si un CRM considera los atajos una feature, es porque sus usuarios viven dentro de la app ocho horas.

---

## 5. Comportamiento con permisos

### 5.1 El modelo de cuatro niveles

| Nivel | Pregunta | Dónde se aplica |
|---|---|---|
| **Objeto** | ¿Puede ver/crear/editar/borrar este tipo de registro? | Servidor + navegación (ocultar sección) |
| **Registro** | ¿Puede ver *este* registro? (propietario, equipo, territorio, compartido manualmente) | Servidor, en cada consulta (filtro obligatorio por `tenant_id` + reglas de visibilidad) |
| **Campo** | ¿Puede ver/editar este campo? (salario, coste, margen) | Servidor al serializar + UI al renderizar |
| **Acción** | ¿Puede ejecutar esta acción? (borrar, exportar, reasignar en lote, ver auditoría) | Servidor + UI (botón ausente o deshabilitado) |

EspoCRM modela explícitamente los cuatro: `aclDefs` para scopes y entidades, `entityAcl` para restringir **campos y relaciones concretas**, y roles administrables `[verificado + https://docs.espocrm.com/development/metadata/ y https://docs.espocrm.com/administration/roles-management/]`. Salesforce llama a la capa de campo *field-level security* y la aplica en la API, no en la vista `[inferido: comportamiento documentado de la plataforma, no consultado en esta sesión]`.

### 5.2 Cómo debe esconder la UI (y por qué)

**Patrón correcto: el servidor no envía lo que no puedes ver.**

```json
// GET /api/deals/42  (rol: comercial)
{
  "id": "42",
  "name": "Acme — Renovación",
  "amount": 24000,
  "stage": "proposal",
  "cost": null,
  "_caps": {
    "visibleFields": ["name","amount","stage","closeDate","owner"],
    "editableFields": ["name","amount","stage"],
    "actions": ["edit","log_call","export_row"]
  }
}
```

La UI renderiza a partir de `_caps`, no de una copia local de la matriz de permisos. Ventajas:

- **Una sola fuente de verdad.** Cambiar un permiso no requiere deploy de frontend.
- **No hay filtración por DevTools.** Si el dato no está en la respuesta, no se puede leer en el JSON.
- **Los formularios se generan solos** (sección 3.2: `layout.fields.filter(f => caps.visibleFields.includes(f))`).

**Patrón a evitar:** enviar todo y ocultar con CSS o `disabled`. Cualquiera abre el inspector o llama a la API. Y en un CRM, "ocultar el margen del negocio" es información competitiva, no un detalle cosmético.

### 5.3 Por qué nunca se confía solo en el cliente

1. El cliente es del usuario: puede modificarlo, interceptarlo o llamar a la API directamente.
2. El cliente tiene una foto antigua de los permisos: si un admin revoca un permiso, la pestaña abierta sigue creyendo que lo tiene. Solo el servidor conoce el estado actual.
3. La UI es una **sugerencia de capacidades**; la API es la **garantía**. Toda escritura revalida: permiso de objeto → permiso de registro → permiso de campo (rechaza campos no editables enviados) → validación de negocio.
4. Auditoría: cada cambio guarda actor, timestamp y diff. Sin esto no hay confianza, y sin confianza no hay CRM compartido.

Regla práctica de implementación: **una única función `can(actor, action, resource)` en el backend**, usada por el serializador, por el endpoint de escritura y por el generador de `_caps`. Si la lógica de permisos vive en tres sitios, se desincroniza en un mes.

---

## 6. Actualización en tiempo real

### 6.1 Qué necesita un CRM realmente (mucho menos de lo que se cree)

| Caso de uso | Urgencia | Mecanismo razonable |
|---|---|---|
| "Alguien más está viendo/editando este registro" | Segundos | Presencia por registro, canal ligero |
| Nuevo comentario / mención | Segundos | Push + badge |
| Cambio de etapa hecho por un compañero | Segundos (o al refrescar) | Invalidación de caché por objeto |
| Recuentos del dashboard | Minutos | Refresco periódico o al enfocar la pestaña |
| Edición colaborativa de una nota larga | Tiempo real | CRDT (Yjs) |

La conclusión incómoda: **el 90% del "tiempo real" de un CRM es invalidación de caché, no colaboración**. Un contador que cambia de 12 a 13 no necesita WebSocket.

### 6.2 SSE vs WebSocket

| Criterio | SSE | WebSocket |
|---|---|---|
| Dirección | Servidor → cliente | Bidireccional |
| Complejidad de infraestructura | HTTP normal, funciona con proxies y CDN, reconexión nativa | Requiere soporte de upgrade, balanceo con sticky o pub/sub |
| Autenticación | Cookies/headers normales | Token en el handshake (a menudo en query string, con riesgo de logs) |
| Coste en servidor | Una conexión por pestaña, sin framing propio | Igual, pero más estado |
| Cuándo brilla | Notificaciones, invalidaciones, presencia "de solo lectura" | Edición colaborativa, cursores, juegos |

**Recomendación:** empezar con **SSE**, y añadir WebSocket solo si aparece edición colaborativa de texto. Twenty es exactamente ese caso: su cliente GraphQL usa `graphql-sse` sobre Apollo `[verificado + https://raw.githubusercontent.com/twentyhq/twenty/main/packages/twenty-front/package.json]`, mientras reserva el transporte bidireccional para el editor de notas, donde usa Yjs + `@tiptap/extension-collaboration` + `@tiptap/y-tiptap` `[verificado + mismo archivo]`.

Patrón de evento (SSE), mínimo y suficiente:

```
event: record.updated
data: {"type":"deal","id":"42","version":7,"fields":["stage"],"actor":"u_18","at":"2026-11-16T14:02:11Z"}
```

El cliente no parchea a ciegas: mira si el registro está abierto y sin ediciones pendientes; si lo está y no hay drafts, aplica; si hay drafts, muestra "Actualizado por Ana — ver cambios".

### 6.3 "Alguien más está editando esto"

Tres niveles, de menos a más esfuerzo:

1. **Aviso pasivo (barato y suficiente):** al abrir un registro, el servidor devuelve `lastModifiedBy` y `lastViewedBy[]` con timestamps. Se muestra "Ana vio esto hace 2 min".
2. **Presencia activa:** canal de presencia por registro con TTL de 30 s (heartbeat). El servidor mantiene `record_id → [{user, at}]`. Barato, y resuelve el 95% de los conflictos sociales.
3. **Bloqueo (lock) real:** casi nunca merece la pena en un CRM; produce registros bloqueados por pestañas olvidadas. EspoCRM tiene "record locking" como opción configurable, no como comportamiento por defecto `[verificado + https://docs.espocrm.com/general/record-locking/]`.

### 6.4 Resolución de conflictos

Lo que hace un CRM serio, y que es Notable por su honestidad: **no intenta fusionar automáticamente; detecta y pregunta**.

EspoCRM implementa *Optimistic Concurrency Control* desde la v7: si el usuario guarda un registro ya modificado por otro, aparece un diálogo que permite elegir, **campo por campo**, entre `Current` (lo que intenté escribir), `Actual` (lo que hay en BD) y `Original` (lo que se leyó al abrir). Y el conflicto **no se dispara si los usuarios tocaron campos distintos** `[verificado + https://docs.espocrm.com/user-guide/optimistic-concurrency-control/]`.

Ese es el patrón a copiar:

```
Si version_cliente != version_servidor:
    diff = campos_modificados_por_mi ∩ campos_modificados_por_otros
    si diff == ∅  → merge silencioso (cada uno tocó lo suyo)
    si diff != ∅  → abrir diálogo con tres valores por campo
```

Implementación mínima:

```sql
UPDATE deal SET stage = $1, version = version + 1, updated_at = now()
WHERE id = $2 AND tenant_id = $3 AND version = $4;   -- 0 filas => conflicto (HTTP 409)
```

Detalles que evitan el 80% de los conflictos reales:
- **Guardado por campo, no por formulario entero.** Si dos personas editan campos distintos del mismo registro, no debería haber conflicto.
- **Idempotencia** en las acciones (`Idempotency-Key`) para que un reintento no duplique una nota.
- **CRDT solo para texto libre** (notas, descripciones largas), donde el merge carácter a carácter sí tiene sentido. Aplicar CRDT a un campo `etapa` o `importe` es un error conceptual: no hay merge posible entre "Propuesta" y "Ganado".
- **Última escritura gana** es aceptable para preferencias de UI (densidad, columnas), nunca para datos de negocio.

HubSpot aporta otra variante interesante en el nivel de *configuración*: cuando varios usuarios comparten una vista guardada, distingue owner/collaborator/member y avisa de divergencias con puntos azules y notificaciones de "Update available" `[verificado + https://knowledge.hubspot.com/records/create-and-manage-saved-views]`. Es colaboración en tiempo real aplicada a la configuración, no al dato: menos glamour, muchísimo más usada.

---

## 7. Stack tecnológico real por producto

### 7.1 Tabla comparativa (verificado donde se indica)

| Producto | Frontend | API / capa de datos | Grid / tabla | Notas de arquitectura | Fuente |
|---|---|---|---|---|---|
| **Twenty** | **React 19 + Vite**, TypeScript, monorepo Nx. Estado: **Jotai**. Estilos: **Linaria** (CSS-in-JS en build). Router: React Router 7. Formularios: react-hook-form + Zod | **GraphQL** con **Apollo Client 4**; **`graphql-sse`** para suscripciones. Backend NestJS + BullMQ + PostgreSQL + Redis | **`react-data-grid` 7.0.0-beta.59**; kanban con **`@dnd-kit`**; dashboards con `react-grid-layout`; gráficos `@nivo`; editor rico **TipTap + Yjs** | i18n con **Lingui** + Crowdin; búsqueda con `fuse.js`; export/import con PapaParse y xlsx | `[verificado + https://github.com/twentyhq/twenty y https://raw.githubusercontent.com/twentyhq/twenty/main/packages/twenty-front/package.json]` |
| **Attio** | React + TypeScript (código de sus blogs: componentes, hooks, Suspense). App web + React Native | Motor de consultas propio **en Rust** sobre dataset a escala de petabytes; ingesta de eventos >250.000/min; API pública REST documentada | Grid propio; publicaron `@attio/react-data-list` (virtualización declarativa, MIT) | Objetivo explícito: "sentirse app nativa con almacenamiento local", **sincronización offline y colaboración en tiempo real**; ejecutan WASM en el cliente | `[verificado + https://attio.com/engineering/blog/where-s-the-technical-challenge-in-crm-anyway- y https://attio.com/engineering/blog/react-data-list-building-virtualized-uis-declaratively]` |
| **HubSpot** | React + TypeScript sobre su design system **Canvas** (Sass, design tokens en JSON/theo, tests con Jasmine/Enzyme y regresión visual). UI Extensions para apps: React | APIs REST públicas; lógica de extensión en funciones serverless | Grid propio dentro de Canvas | Design system como producto interno con equipo dedicado; el repo público de Canvas quedó archivado (señal de que el sistema evolucionó internamente) | `[verificado + https://adele.uxpin.com/hubspot-canvas y https://product.hubspot.com/blog/how-building-a-design-system-empowers-your-team-to-focus-on-people-not-pixels]` |
| **Pipedrive** | React (detectado en su stack público); design system propio con tokens semánticos (`surface`, `text`, `icon`, `divider`, `primary`, `positive`…) y publicación en Figma Community | Backend mayormente PHP histórico + servicios; APIs REST públicas | Grid propio | Fuerte inversión en tokens semánticos para theming; foco de producto en el kanban como pantalla principal | `[verificado parcial + https://www.priitkaru.com/work/pipedrive-ds]` `[inferido: framework exacto del app web]` |
| **EspoCRM** | **SPA sobre framework propio** ("bullbone"), con **Backbone 1.3.3 + jQuery 3.7 + Underscore + Handlebars**; TypeScript parcial en el núcleo. Build con Grunt + Rollup | REST API en PHP 8.3+; MySQL/MariaDB/PostgreSQL | Grid propio con vistas de lista, kanban y dashlets (`gridstack`) | **El ejemplo más puro de metadata-driven**: JSON fusionado y cacheado (`entityDefs`, `clientDefs`, `aclDefs`, `entityAcl`, `logicDefs`, `streamDefs`); WebSocket vía cliente WAMP (`autobahn`); Control de Concurrencia Optimista | `[verificado + https://github.com/espocrm/espocrm, https://raw.githubusercontent.com/espocrm/espocrm/master/package.json y https://docs.espocrm.com/development/metadata/]` |
| **SuiteCRM 8** | **Angular** (reconstrucción sobre el SuiteCRM 7 legacy en PHP/Sugar) | **Symfony** + API-first | Grid del framework Angular | La v8 es explícitamente "un rebuild moderno con frontend Angular y backend Symfony, API-first" (v8.10.2). La v7 sigue siendo el legado tipo Sugar/Sidecar | `[verificado + https://github.com/SuiteCRM/SuiteCRM-Core]` `[inferido: detalles del frontend de la v7]` |
| **Salesforce** | **LWC** (Lightning Web Components): estándar de Web Components, motor y compilador open source bajo **MIT**; **Aura** como legado. Estilos: **SLDS**, framework CSS + design tokens (BSD-3), desarrollado con Storybook y validado con tests de accesibilidad aXe | API REST/SOAP + UI API + GraphQL (salesforce.com) | **`lightning-datatable`** con edición inline por `draftValues`, tipos de columna declarativos y tipos de dato personalizados | Formularios generados desde el layout del objeto (`lightning-record-form`, modo dynamic); todo el modelo de metadatos (objetos, campos, layouts, permisos) vive en la plataforma | `[verificado + https://github.com/salesforce/lwc, https://github.com/salesforce-ux/design-system y https://raw.githubusercontent.com/trailheadapps/lwc-recipes/main/force-app/main/default/lwc/datatableInlineEditWithUiApi/datatableInlineEditWithUiApi.js]` |

### 7.2 Qué tienen en común (y es lo importante)

1. **SPA con estado de navegación rico.** Ninguno hace SSR de la list view: la vista, los filtros, las columnas y el panel lateral tienen que sobrevivir a la navegación.
2. **Backend como autoridad total.** Todos validan y filtran en servidor; el frontend solo pinta capacidades.
3. **Metadatos en algún sitio.** En BD (Salesforce, EspoCRM, HubSpot) o en código tipado (Twenty, Attio al principio). Pero siempre hay una descripción declarativa de campos y layouts.
4. **Grid propio o de bajo nivel.** Nadie usa el grid "de la framework" con estilos incluidos, porque la densidad visual es el producto.
5. **Un design system con tokens.** SLDS, Canvas, tokens semánticos de Pipedrive, `twenty-ui`. Sin tokens no hay temas ni consistencia.
6. **i18n desde el principio**, con archivos de traducción gestionados externamente (Crowdin, POEditor).

### 7.3 Stack mínimo recomendado para un equipo de una persona

**Principio:** una sola persona no puede mantener un motor de metadatos, un grid propio y un sistema de diseño completo. Debe elegir **una** cosa para construir a medida (la list view, que es el producto) y comprar el resto.

| Capa | Elección | Versión (nov 2026) | Por qué esta y no otra |
|---|---|---|---|
| Build | **Vite** | 7.x | Arranque instantáneo, SPA sin ceremonia, cero configuración de servidor |
| UI | **React + TypeScript** | React 19.2 | Es lo que usa todo el ecosistema de grids y editores; types en los metadatos |
| Rutas | **TanStack Router** (o React Router 7) | 1.170.x | Rutas type-safe y search params como estado real (`?view=abc&sort=-closeDate`), que es exactamente el modelo de una list view |
| Estado de servidor | **TanStack Query** | 5.107.x | Caché, invalidación, reintentos, `keepPreviousData` al paginar (evita el parpadeo). El 90% del "estado" de un CRM es estado de servidor |
| Estado de UI | **Zustand** o **Jotai** | — | Solo para selección de filas, panel abierto, densidad. Un store pequeño, no un Redux |
| Tabla | **TanStack Table + TanStack Virtual** | 9.3.1 / 3.14.26 | Headless: controlas densidad, accesibilidad y estilos. Sin licencia. Si necesitas edición tipo Excel ya, usa `react-data-grid` (asumiendo beta) |
| Formularios | **react-hook-form + Zod** | 7.92 / 4.6 | Schema único compartido con el servidor; RHF evita re-render por campo |
| Componentes base | **Radix UI** (o `shadcn/ui` sobre Radix) | — | Accesibilidad resuelta (focus trap, `aria-*`, teclado) sin imponer estética. Es lo contrario de MUI |
| Kanban | **@dnd-kit** | 6.3.1 | Accesible por teclado y con sensores táctiles; es lo que usa Twenty |
| Command palette | **cmdk** | 1.1.1 | 4 KB, accesible, filtrado incluido |
| Fechas / números | **date-fns** + `Intl` nativo | 4.x | Sin dependencia de locale empaquetada; `Intl.NumberFormat` y `Intl.DateTimeFormat` cubren moneda y fechas i18n |
| i18n | **Lingui** (o i18next) | 5.x | Extracción en build, plurales ICU, compatible con Crowdin |
| Gráficos | **Recharts** o `@nivo` | — | Con 4 gráficos no necesitas una librería de dashboards |
| Tiempo real | **SSE** nativo (`EventSource`) | — | Sin infraestructura extra; se sube a WebSocket solo si hay edición colaborativa |
| Backend | **PostgreSQL + API REST tipada** (Node o Python) | — | Keyset pagination, `tsvector`, índices compuestos. GraphQL solo si vas a tener muchos clientes distintos |
| Auth | Proveedor gestionado (Clerk/Auth0/Keycloak) | — | No construyas autenticación; sí construye autorización |

**Lo que NO incluiría al principio:** motor de metadatos en BD, SSR/Next.js, WebSocket, CRDT, dashboard configurable, y un grid comercial de pago. Ninguno es necesario para validar el producto, y todos juntos son la razón por la que los proyectos de una persona no llegan a la semana 12.

**Coste de licencias del stack anterior: 0 €.** Es un criterio, no una casualidad: con una persona, un cambio de licencia de AG Grid Enterprise o MUI X Premium es una conversación que no puedes tener.

---

## 8. Rendimiento

### 8.1 Virtualización de filas

- Solo se montan las filas visibles + un *overscan* de 5–10. Con 40 px de alto y 900 px de viewport, son ~25 filas.
- Requiere **altura fija** y un contenedor con altura definida. Es la razón por la que "filas que crecen al hacer hover" son incompatibles con tablas grandes.
- Virtualizar **columnas** también cuando hay más de ~15: cada celda extra multiplica nodos por fila.
- Cuidado con el foco y la accesibilidad: al virtualizar, el DOM no contiene la fila 4.812, pero el lector de pantalla debe saber que existe (`aria-rowcount` con el total, `aria-rowindex` en cada fila) `[inferido: requisito WAI-ARIA aplicado, no verificado en un producto concreto]`.

### 8.2 Carga diferida

| Qué | Cuándo | Cómo |
|---|---|---|
| Filas | Al hacer scroll / "cargar más" | Keyset, 50 por página |
| Detalle del registro | Al abrir el panel lateral | Petición propia, no incluida en la lista |
| Timeline | Al hacer scroll dentro del panel | Cursor por fecha |
| Asociaciones | Al expandir la sección | Petición por relación |
| Dashboard | Al entrar en la pestaña | `staleTime` largo + refresh al enfocar |
| Búsqueda global | Tras 2 caracteres y 250 ms | Debounce + cancelación |

### 8.3 Estado: React Query / SWR

Un CRM es un problema de **caché de servidor**, no de estado global de cliente. Concretamente:

- Claves de caché por objeto y vista: `['deals', viewId, {cursor, sort, filters}]`. Invalidar por prefijo al recibir un evento SSE: `queryClient.invalidateQueries({queryKey:['deals']})`.
- **Actualización optimista** con `onMutate` + rollback en `onError` para la edición en celda: es lo que hace que la tabla se sienta instantánea.
- `placeholderData: keepPreviousData` al cambiar de página evita el hueco en blanco (el "flash de tabla vacía" es el defecto más visible de un CRM mal hecho).
- `staleTime` agresivo (30–60 s) en listas, corto (0) en el registro abierto.

### 8.4 SSR vs SPA, y por qué casi todos son SPA

| Necesidad del CRM | SSR | SPA |
|---|---|---|
| SEO | Irrelevante (app tras login) | — |
| Primera pintura rápida | Ventaja | Se compensa con bundle pequeño y skeleton |
| Estado persistente al navegar (vista, filtros, scroll, panel) | Difícil | Natural |
| Sesión larga (8 h en la misma pestaña) | No aporta | Es el caso de uso |
| Tiempo real y actualización optimista | Complica el modelo | Directo |
| Despliegue | Requiere servidor Node | Estático en CDN |

Ninguno de los productos estudiados hace SSR de su pantalla principal: EspoCRM se declara SPA explícitamente `[verificado + https://github.com/espocrm/espocrm]`, Twenty se construye con Vite y sirve estático `[verificado + package.json de twenty-front]`. La única razón para considerar SSR sería una vista pública (formulario de captación, portal de cliente), y eso es otra aplicación.

### 8.5 Presupuesto de rendimiento (objetivo medible)

| Métrica | Objetivo | Cómo se mide |
|---|---|---|
| Primera list view interactiva | < 2,0 s en 4G | Lighthouse / WebPageTest |
| Cambio de vista guardada | < 400 ms percibidos | Skeleton inmediato + datos < 400 ms |
| Scroll | 60 FPS sostenidos | DevTools Performance, sin long tasks > 50 ms |
| Nodos DOM en la tabla | < 1.500 | `document.querySelectorAll('[role=row]').length` estable al hacer scroll |
| Edición de celda | < 16 ms hasta el repintado | Optimista, sin esperar red |
| Búsqueda global | < 150 ms hasta el primer resultado | `EXPLAIN ANALYZE` < 30 ms en BD |
| Bundle inicial (gzip) | < 350 KB | `vite build --report`; cargar el editor rico en diferido |

---

## 9. Accesibilidad e internacionalización

### 9.1 Accesibilidad: lo que hace un CRM serio

| Práctica | Evidencia | Por qué importa en un CRM |
|---|---|---|
| **Tests automatizados de aXe en CI** | SLDS los ejecuta: `npm run test:a11y` "run axe tests for accessibility violations" `[verificado + https://github.com/salesforce-ux/design-system]` | Una tabla con 30 columnas y 40 filas es donde más se rompe la semántica; automatizarlo es la única forma de no regresar |
| **Documentación de accesibilidad por componente** | AG Grid mantiene una página de accesibilidad por producto y la incluye en todas las licencias `[verificado + https://www.ag-grid.com/license-pricing/]` | Si el grid no es accesible, el CRM entero no lo es |
| **Navegación completa por teclado** | EspoCRM publica su lista de atajos como feature `[verificado + https://docs.espocrm.com/user-guide/shortcuts/]` | Los comerciales teclean 8 horas; el ratón es el cuello de botella |
| **Contraste y foco visibles en tokens** | SLDS y Pipedrive definen tokens semánticos de color `[verificado + https://www.priitkaru.com/work/pipedrive-ds]` | El estado de etapa se codifica con color: si el color falla, el dato desaparece |

**Imprescindible (no negociable) en una tabla de CRM:**

1. Semántica correcta: `role="grid"` con `row`/`gridcell`/`columnheader`, o `<table>` real. Nunca `div`s con `onClick`.
2. **Roving tabindex**: un solo punto de tabulación en la tabla; las flechas mueven entre celdas.
3. Foco visible en la celda activa, con anillo de 2 px y contraste ≥ 3:1.
4. `aria-sort` en la cabecera ordenada; anuncio del resultado al aplicar filtros (`aria-live="polite"`: "12 resultados").
5. `aria-rowcount` = total real (no el visible) cuando se virtualiza.
6. El color de etapa **nunca** es el único portador de información: añade texto o icono.
7. Modales y paneles laterales con focus trap, `Esc` para cerrar y retorno de foco al disparador.
8. Contraste mínimo 4,5:1 en texto; las tablas densas con gris claro son el fallo más común.

### 9.2 Internacionalización

| Aspecto | Qué hacer | Ejemplo verificado |
|---|---|---|
| **Framework de traducción** | Mensajes extraídos en build, no strings en el JSX | Twenty usa **Lingui** y gestiona traducciones en **Crowdin** `[verificado + README y package.json]`; EspoCRM usa POEditor `[verificado + https://github.com/espocrm/espocrm]` |
| **Traducir los metadatos, no solo la UI** | Las etiquetas de campos, opciones de etapa y nombres de objeto también se traducen, y son datos, no código | EspoCRM: `label`/`options` viven en los JSON de metadatos y se traducen por locale `[verificado + https://docs.espocrm.com/development/metadata/]` |
| **Fechas y zonas horarias** | Guardar en UTC, mostrar en la zona del usuario, y decirlo | Twenty incluye `date-fns-tz` y `temporal-polyfill` `[verificado + package.json]` |
| **Números y moneda** | `Intl.NumberFormat` con la moneda del tenant, no del navegador; separadores por locale | `[inferido: estándar]` |
| **Plurales y género** | ICU MessageFormat (`{count, plural, one {# negocio} other {# negocios}}`) | Lingui/i18next lo soportan nativamente |
| **RTL** | Propiedades lógicas de CSS (`margin-inline-start`, `padding-inline-end`) en vez de `left`/`right` | `[inferido: práctica estándar]`; Notion ofrece árabe y hebreo entre sus 20+ locales `[verificado + https://www.notion.com/help/views-filters-and-sorts]` |
| **Formatos de nombre y dirección** | Orden configurable (nombre/apellidos), y plantillas de dirección por país | EspoCRM tiene `addressFormats` en metadatos `[verificado + https://docs.espocrm.com/development/metadata/]` |
| **Orden alfabético** | `Intl.Collator`, no `<` de strings: "Ángel" debe ordenarse con "Ana" | `[inferido]` |
| **Telefonía** | Validación y formateo por país (`libphonenumber-js`) | Twenty lo incluye `[verificado + package.json]` |

**El error más caro:** tratar la i18n como una fase final. Si los textos están hardcodeados en 200 componentes, traducir a un segundo idioma es un refactor de semanas, no una tarea de traducción.

---

## 10. Diez patrones de UI que vale la pena copiar, y cinco que no

### 10.1 Copiar

| # | Patrón | Por qué funciona | Referencia |
|---|---|---|---|
| 1 | **Saved views como pestañas de primera clase**, con visibilidad privada/equipo/todos, vista por defecto y clonado | Convierte "filtrar cada mañana" en "abrir mi pestaña". Es el corazón del producto | HubSpot `[verificado]` |
| 2 | **Panel lateral (side peek) sobre la lista**, en lugar de página completa | Mantiene posición, scroll y contexto; el usuario vuelve a la lista sin perderse. Configurable por vista | Notion `[verificado]`, patrón extendido en Attio/HubSpot |
| 3 | **Draft values + guardado explícito por lote**, con marca visual de "pendiente" | Permite editar 15 celdas y guardar una vez; el usuario controla cuándo se escribe en el sistema de registro | Salesforce `lightning-datatable` `[verificado]` |
| 4 | **Chips de filtro + filtros avanzados anidados (máx. 3 niveles)** | Hace visible el estado del filtro (nadie recuerda qué filtró) y limita la complejidad del constructor | Notion `[verificado]` |
| 5 | **Columna primaria congelada + densidad configurable** | El nombre siempre visible al desplazar 12 columnas; la densidad deja que cada rol elija entre "ver más" y "leer cómodo" | Notion "Freeze up to column" `[verificado]`; HubSpot deja al rol `Member` cambiar altura de fila `[verificado]` |
| 6 | **Command palette (⌘K) que incluye vistas guardadas y acciones** | Un solo atajo cubre navegación, creación y búsqueda; es el mayor multiplicador de velocidad para usuarios intensivos | `cmdk`; atajos documentados en EspoCRM `[verificado]` |
| 7 | **Kanban con agregados por columna** (suma de importe, conteo) y arrastre con deshacer | Convierte el tablero en previsión, no en decoración; el deshacer elimina el miedo a arrastrar | Patrón común Pipedrive/HubSpot/Pipedrive-style `[inferido en detalle, verificado el tipo de vista "board" en HubSpot]` |
| 8 | **Timeline unificada, agrupada por día, con icono por tipo de evento y cambios de campo agrupados** | Da la memoria del cliente en 5 segundos; agrupar los cambios triviales evita el ruido que hace que nadie la lea | EspoCRM *Stream* con `streamDefs` `[verificado]` |
| 9 | **Resolución de conflictos campo por campo** (Actual / Mío / Original) | Resuelve el caso real (dos personas, campos distintos) sin bloquear ni perder datos; es honesto con el usuario | EspoCRM OCC v7+ `[verificado]` |
| 10 | **Skeletons con el número exacto de filas de la página** en lugar de spinner central | Evita el salto de layout y hace que la carga se perciba más corta; el ojo ya sabe dónde mirar | Práctica de Attio (filas de carga en listas virtualizadas) `[verificado + blog de ReactDataList]` |

*Mención:* **actualización optimista con toast accionable ("Deshacer")** en cada mutación. Va implícita en el patrón 3 y merece la pena por sí sola.

### 10.2 No copiar

| # | Anti-patrón | Por qué no | Qué hacer en su lugar |
|---|---|---|---|
| 1 | **Scroll infinito puro, sin paginación ni retorno de posición** | Al abrir un registro y volver, el usuario aterriza arriba; no hay "fin" y no se puede imprimir ni exportar lo visto. En un registro de negocio es hostil | "Cargar más" con cursor, contador aproximado y restauración de scroll por vista |
| 2 | **Guardar en línea al `onBlur`, sin indicador ni deshacer** | Es la causa nº1 de datos corruptos percibidos: el usuario roza un campo, se guarda, y nadie sabe qué cambió | Drafts + guardado explícito, o guardado optimista con marca "guardando" y "Deshacer" |
| 3 | **Renderizar toda la tabla "y ya optimizamos luego"** | Con 3.000 filas la pestaña se congela; refactorizar después obliga a cambiar el modelo de datos (paginación en servidor) con el producto ya en uso | Virtualización + keyset desde el primer día, aunque al principio solo haya 200 filas |
| 4 | **Modal para todo**, incluido el formulario completo de creación/edición | Rompe la comparación con la lista, tapa el contexto y acumula modales sobre modales | Panel lateral para detalle, página completa solo para creación compleja, modal solo para confirmaciones |
| 5 | **Filtros y búsqueda en cliente sobre el dataset completo** | Solo funcionan sobre lo cargado: "filtro por Ganado y salen 12" cuando hay 400. Y con 50k filas, el `filter()` bloquea el hilo | Filtros como consulta de servidor sobre columnas indexadas; en cliente solo el resaltado de coincidencias |

*Variante del nº5 que merece mención propia:* **auto-guardar los cambios de una vista compartida sin distinguir "mi vista" de "la vista del equipo"**. Un usuario que mueve una columna no debería cambiar la vista de todos. HubSpot lo resuelve con roles owner/collaborator/member y notificación de "Update available" `[verificado]`; sin eso, el equipo deja de compartir vistas.

---

## 11. Fuentes consultadas

**Primarias (repositorios y paquetes)**
- Twenty — repo y `packages/twenty-front/package.json`: https://github.com/twentyhq/twenty
- EspoCRM — repo y `package.json`: https://github.com/espocrm/espocrm
- SuiteCRM 8 — https://github.com/SuiteCRM/SuiteCRM-Core
- Salesforce LWC — https://github.com/salesforce/lwc
- Salesforce Lightning Design System — https://github.com/salesforce-ux/design-system
- LWC Recipes (datatable inline edit, record-form dinámico): https://github.com/trailheadapps/lwc-recipes
- TanStack Table — https://github.com/TanStack/table
- Registro npm (versiones y licencias): `https://registry.npmjs.org/<paquete>/latest`

**Documentación de producto**
- HubSpot — vistas guardadas: https://knowledge.hubspot.com/records/create-and-manage-saved-views
- EspoCRM — metadatos: https://docs.espocrm.com/development/metadata/
- EspoCRM — control de concurrencia optimista: https://docs.espocrm.com/user-guide/optimistic-concurrency-control/
- EspoCRM — atajos de teclado: https://docs.espocrm.com/user-guide/shortcuts/
- Notion — vistas, filtros, orden y grupos: https://www.notion.com/help/views-filters-and-sorts
- MUI X — licencias: https://mui.com/x/introduction/licensing/
- AG Grid — licencias y comparativa Community/Enterprise: https://www.ag-grid.com/license-pricing/

**Blogs de ingeniería y análisis**
- Attio — "Where's the technical challenge in CRM anyway?": https://attio.com/engineering/blog/where-s-the-technical-challenge-in-crm-anyway-
- Attio — "React Data List: building virtualized UIs declaratively": https://attio.com/engineering/blog/react-data-list-building-virtualized-uis-declaratively
- Benchmark de 6 grids sobre 50.000 filas: https://itnext.io/i-benchmarked-6-react-data-grids-on-50-000-rows-the-real-winner-was-the-tradeoff-d447fbf5ecde
- Guía de rendimiento de data grids con datasets grandes: https://www.infragistics.com/blogs/best-react-data-grid-for-large-datasets-performance-guide
- HubSpot Canvas (ficha técnica del design system): https://adele.uxpin.com/hubspot-canvas
- HubSpot — construcción del design system: https://product.hubspot.com/blog/how-building-a-design-system-empowers-your-team-to-focus-on-people-not-pixels
- Pipedrive — caso de estudio del design system y tokens: https://www.priitkaru.com/work/pipedrive-ds

**Nota sobre límites de verificación:** la documentación de Salesforce (`developer.salesforce.com`) y el sitio de SLDS (`lightningdesignsystem.com`) se sirven con renderizado en cliente y bloquearon la consulta directa; las afirmaciones sobre LWC, `lightning-datatable`, `lightning-record-form` y SLDS se verificaron en los repositorios oficiales de GitHub enlazados arriba, no en el sitio de documentación. Las afirmaciones sobre *field-level security* de Salesforce y sobre el frontend de Pipedrive están marcadas como `[inferido]`.
