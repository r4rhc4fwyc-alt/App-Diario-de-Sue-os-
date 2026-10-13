# Plan de implementación — App del Diario de Sueños

Propuesta para aprobación de Roberto, previa a la etapa 0.
Responde a lo que pide la sección 0.5 de la especificación: estructura del
repositorio, esquema de base de datos y stack confirmado.

Estado: **borrador a la espera de aprobación.** No se escribe código de la
aplicación hasta que este documento esté aprobado.

Nota: la especificación todavía no está en el repositorio (vive en el chat).
Su primer lugar será `docs/especificacion.md`, como primer commit de la etapa 0.
Las referencias de este plan (§1, §5.3, §8.1…) apuntan a ella.

---

## 1. Stack confirmado

| Pieza | Decisión | Razón |
|---|---|---|
| Backend | Python 3.12 + FastAPI + uvicorn | El generador del PDF ya es Python (reportlab). Un solo lenguaje para parser de migración, cómputo, exportación y PDF. |
| Base de datos | SQLite en modo WAL, cifrada con SQLCipher | Un usuario, volumen chico, respaldo trivial, sin servidor aparte. Las restricciones (CHECK, triggers, FK) son el lugar donde viven los invariantes. |
| Acceso a datos | `sqlite3` de la biblioteca estándar + SQL escrito a mano. **Sin ORM.** | El esquema es el contrato y el cómputo es SQL. Un ORM esconde justo lo que aquí hay que poder leer y auditar. |
| Frontend | TypeScript + Svelte 5 + Vite, SPA estática servida por el backend | Es lo más liviano que sigue siendo cómodo para formularios y estado. Sin SSR, sin router de servidor, sin build de node en producción (se compila y se sirven archivos estáticos). |
| Sin conexión | Service worker con Workbox (`vite-plugin-pwa`) + IndexedDB (`idb`) | Requisito del día uno. La pantalla de captura se precachea completa. |
| Modelo | SDK oficial `anthropic` para Python, llamado sólo desde el servidor. Nombre del modelo en configuración. | Invariante 2 y §3.5. |
| PDF | reportlab, `layout.py` reutilizado, `datos.py` reescrito contra la base | §8. |
| Validación | `pydantic` en la frontera HTTP; JSON Schema estricto (`additionalProperties: false`) para lo que devuelve el modelo | Invariante 2: el esquema del modelo no tiene dónde poner el crudo. |
| Pruebas | `pytest` + `playwright` (sólo para la PWA sin conexión) | Las pruebas de invariante son parte del entregable, no un extra. |
| Despliegue | Contenedor (Podman o Docker) + unidad systemd, detrás de Tailscale | §3.1. Documentado en `docs/despliegue.md`. |

### 1.1 Alternativas descartadas (y por qué)

- **Postgres:** correcto para muchos usuarios; aquí sólo agrega un proceso que
  mantener y complica el respaldo de un archivo único.
- **SvelteKit / Next:** traen servidor, rutas de servidor y despliegue de Node.
  No hay nada que renderizar en servidor; sobra.
- **ORM (SQLAlchemy):** el invariante 5 exige que el cómputo sea SQL legible, y
  el invariante 1 exige que *no exista* un método para actualizar el crudo. Con
  SQL a mano, la ausencia de ese método es verificable leyendo el repositorio.
- **Web Speech API del navegador** para el dictado: ver §9.3, hay un problema de
  privacidad real.

---

## 2. Estructura del repositorio

```
.
├── README.md
├── Makefile                      # make instalar | pruebas | correr | respaldar | restaurar
├── docs/
│   ├── especificacion.md         # la especificación, versionada
│   ├── plan-implementacion.md    # este documento
│   ├── invariantes.md            # los 11, cada uno con dónde se hace cumplir y qué prueba lo cubre
│   ├── despliegue.md             # runbook del VPS, Tailscale, actualización
│   ├── respaldos.md              # respaldo, restauración, prueba de restauración
│   ├── decisiones-pendientes.md  # P1–P10, con fecha de resolución
│   └── adr/                      # decisiones técnicas con fecha y razón
├── prompts/                      # §7.3 — prompts versionados, en español, fuera del código
│   ├── protocolo-comun.md        # «si fuera mi sueño», Roberto es la autoridad, sin diccionarios
│   ├── ficha/v1.md
│   ├── sospechas-transcripcion/v1.md
│   ├── super-resumen/v1.md
│   ├── lectura-taller/v1.md
│   ├── veredicto-fluidez/v1.md
│   ├── resumen-semanal/v1.md
│   ├── cierre-mensual/v1.md
│   └── carta-foto/v1.md
├── backend/
│   ├── pyproject.toml
│   ├── suenos/
│   │   ├── config.py             # variables de entorno; nada secreto en el código
│   │   ├── dominio/              # tipos, listas cerradas, formato de IDs, reglas de campo (§5)
│   │   │   ├── ids.py
│   │   │   ├── vocabulario.py    # MUNDO, TONO, TÉCNICA, ESTADO, VÍAS, TRANSICIÓN
│   │   │   └── fecha.py          # FECHA = día calendario según HORA 24 h (§5.2)
│   │   ├── db/
│   │   │   ├── conexion.py
│   │   │   ├── esquema.sql
│   │   │   ├── vistas.sql
│   │   │   └── migraciones/      # 001_inicial.sql, 002_…  (versionadas, hacia adelante)
│   │   ├── repositorio/          # un módulo por agregado; dictado.py NO expone escritura
│   │   ├── entrada/
│   │   │   └── registrar_sueno.py  # §3.3 — la única puerta de entrada
│   │   ├── computo/              # §1.5 — sin red, sin anthropic, sólo SQL y Python puro
│   │   │   ├── recurrencias.py   coocurrencias.py   distribuciones.py
│   │   │   ├── tasas.py          calendario.py      panel_taller.py
│   │   ├── llm/                  # el único lugar que importa anthropic
│   │   │   ├── cliente.py        prompts.py         esquemas/*.json
│   │   │   └── ficha.py  sospechas.py  lectura.py  fluidez.py  semanal.py  mensual.py  carta.py
│   │   ├── api/                  # routers FastAPI, uno por pantalla
│   │   ├── export/               # §9 — bitacora.py indice.py entidades.py asignaciones.py taller.py cierres.py
│   │   ├── pdf/                  # layout.py (reutilizado) + datos.py (contra la base)
│   │   ├── migracion/            # parser de los .md actuales + informe (§10)
│   │   ├── respaldo/             # snapshot + age + envío fuera del VPS
│   │   └── auth/
│   └── tests/
│       ├── invariantes/          # ver §12
│       ├── unidad/   integracion/
│       └── datos/                # fixtures SINTÉTICOS; ningún sueño real en el repo (§9.1)
├── frontend/
│   ├── package.json  vite.config.ts  tsconfig.json
│   └── src/
│       ├── paginas/  noche/ captura/ bandeja/ sueno/ taller/ cierres/ explorar/
│       ├── offline/  idb.ts  cola.ts  sw.ts
│       ├── api/      componentes/  estilos/
│       └── dominio/  # copia generada de las listas cerradas, para no duplicar a mano
└── despliegue/
    ├── Containerfile  compose.yml
    ├── systemd/suenos.service  suenos-respaldo.timer
    └── respaldo/respaldar.sh  restaurar.sh
```

Dos reglas de la estructura que no son cosméticas:

1. `computo/` y `llm/` no se importan entre sí. `api/` orquesta: pide el cómputo,
   se lo pasa al modelo, guarda lo que vuelve marcado como hipótesis.
2. `repositorio/dictado.py` tiene `insertar()` y `leer()`. No tiene `actualizar()`
   ni `borrar()`, y una prueba verifica que nunca los tenga.

---

## 3. Esquema de base de datos

SQL completo propuesto. Nombres y valores van acentuados como los escribe el
corpus, porque los mismos valores salen en la exportación `.md` y en el PDF.

Convenciones:

- Fechas de sueño (`fecha`, `hora`): tiempo civil de Chile, **tal como marcaba el
  reloj**. Nunca se convierten de zona. `fecha` es `AAAA-MM-DD`, `hora` es `HH:MM`.
- Marcas de tiempo del sistema (`creado_en`, `sincronizado_en`…): ISO-8601 en UTC.
- Posiciones dentro del crudo: desplazamientos en caracteres Unicode, intervalo
  semiabierto `[inicio, fin)`.
- Patrón de procedencia: donde un texto puede venir del modelo, hay tres
  columnas: el texto, `*_origen` (`escrito por Roberto` | `propuesta del modelo
  aprobada` | `propuesta del modelo editada`) y `*_respuesta_id` hacia
  `respuesta_modelo`. Así una regeneración no reinventa nada (§4.9).

### 3.1 El crudo (invariantes 1 y 2)

```sql
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;

CREATE TABLE dictado (
  uuid_local       TEXT PRIMARY KEY,              -- UUIDv4 generado en el dispositivo
  texto            TEXT NOT NULL,                 -- INMUTABLE
  texto_sha256     TEXT NOT NULL,
  largo            INTEGER NOT NULL,              -- en caracteres
  capturado_en     TEXT NOT NULL,                 -- ISO-8601 con offset, reloj del dispositivo
  dispositivo      TEXT,                          -- etiqueta libre: "celular", "computador"
  sincronizado_en  TEXT NOT NULL,                 -- ISO-8601 UTC, reloj del servidor
  origen           TEXT NOT NULL CHECK (origen IN ('pwa','mcp','migración'))
);

-- El crudo se escribe una vez. No hay UPDATE ni DELETE posible, ni por error,
-- ni por migración, ni desde una consola sqlite abierta a mano.
CREATE TRIGGER dictado_no_se_modifica BEFORE UPDATE ON dictado
BEGIN SELECT RAISE(ABORT, 'el crudo es inmutable'); END;

CREATE TRIGGER dictado_no_se_borra BEFORE DELETE ON dictado
BEGIN SELECT RAISE(ABORT, 'el crudo no se borra'); END;
```

`texto_sha256` se calcula al insertar y una tarea diaria lo recalcula: si alguna
vez no coincide, el crudo fue tocado por fuera y la app lo dice en voz alta.

### 3.2 IDs (invariante 4)

```sql
CREATE TABLE serie_id (
  serie    TEXT PRIMARY KEY CHECK (serie IN ('S','T','A')),
  ancho    INTEGER NOT NULL,          -- S=4, T=3, A=3
  proximo  INTEGER NOT NULL
);

-- Bitácora de números emitidos: un número usado nunca vuelve a estar libre,
-- aunque su registro se borre. A-001 vive aquí, marcada anulada.
CREATE TABLE id_emitido (
  serie       TEXT NOT NULL,
  numero      INTEGER NOT NULL,
  emitido_en  TEXT NOT NULL,
  anulado     INTEGER NOT NULL DEFAULT 0,
  nota        TEXT,
  PRIMARY KEY (serie, numero)
);

CREATE TRIGGER id_emitido_no_se_borra BEFORE DELETE ON id_emitido
BEGIN SELECT RAISE(ABORT, 'un ID emitido no se libera'); END;
```

La asignación es `UPDATE serie_id SET proximo = proximo + 1 WHERE serie = ?
RETURNING proximo - 1`, dentro de la misma transacción que inserta el registro.
No se deriva de la fecha, no se deriva de `MAX(numero)` (eso reutilizaría el
número de un registro borrado), y el dispositivo nunca lo calcula.

### 3.3 Sueño

```sql
CREATE TABLE sueno (
  id             TEXT PRIMARY KEY,                -- 'S-0001'
  numero         INTEGER NOT NULL UNIQUE,
  dictado_uuid   TEXT NOT NULL REFERENCES dictado(uuid_local),
  rango_inicio   INTEGER NOT NULL DEFAULT 0,      -- partición de un dictado (§4.10)
  rango_fin      INTEGER NOT NULL,

  fecha          TEXT,                            -- día calendario según HORA (§5.2)
  hora           TEXT,                            -- 'HH:MM', 24 h
  dormido_h      REAL,
  despertar      TEXT,
  posicion       TEXT,
  recuerdo       TEXT,
  lagunas        TEXT,
  venia_de       TEXT CHECK (venia_de IS NULL OR venia_de IN ('despierto','sueño')),
  transicion     TEXT CHECK (transicion IS NULL OR transicion IN ('hipnagógico','hipnopómpico')),

  mazo           TEXT CHECK (mazo IS NULL OR mazo IN ('carta','no se usó','activado pero no se sacó')),
  noche_id       INTEGER REFERENCES noche(id),    -- declaración que le aplica (P6)

  via_lucidez    TEXT CHECK (via_lucidez IS NULL OR via_lucidez IN (
                   'espontánea','desde falso despertar','desde chequeo interno',
                   'desde parálisis','WILD desde hipnagogia')),
  reentrada_de   TEXT REFERENCES sueno(id),
  via_reentrada  TEXT CHECK (via_reentrada IS NULL OR via_reentrada IN (
                   'misma escena sostenida','posición sin moverse',
                   'intención previa a despertar','espontánea sin intención')),

  situacion      TEXT,
  acto           TEXT,
  arqueologia    TEXT,
  super_resumen  TEXT,
  super_resumen_origen       TEXT,
  super_resumen_respuesta_id INTEGER REFERENCES respuesta_modelo(id),

  estado_ficha   TEXT NOT NULL DEFAULT 'borrador'
                   CHECK (estado_ficha IN ('borrador','confirmada')),
  creado_en      TEXT NOT NULL,
  confirmado_en  TEXT,

  CHECK (rango_fin > rango_inicio),
  CHECK (id = 'S-' || substr('0000' || numero, -4)),
  -- VÍA DE REENTRADA sólo existe si existe el enlace (§5.6)
  CHECK (via_reentrada IS NULL OR reentrada_de IS NOT NULL),
  -- la etiqueta se deriva de la respuesta, no se pregunta con el tecnicismo (§5.7)
  CHECK (venia_de IS NULL OR transicion IS NULL
         OR (venia_de = 'despierto' AND transicion = 'hipnagógico')
         OR (venia_de = 'sueño'     AND transicion = 'hipnopómpico'))
);
```

**ORDEN no es una columna.** Se deriva de `fecha` + `hora` en una vista
(`v_orden`), con el S-ID como desempate cuando la hora es «no registrado». Así
no puede quedar desalineado con la hora. La migración importa el ORDEN de los
archivos y el informe lista cualquier sueño donde el ORDEN escrito no coincida
con el derivado, para que lo revises. *(Si prefieres ORDEN editable a mano, se
guarda como columna y se agrega una verificación en el cierre semanal; dímelo.)*

### 3.4 «No registrado» como dato válido (§4.2)

```sql
CREATE TABLE campo_catalogo (campo TEXT PRIMARY KEY, prioridad_repregunta INTEGER);

-- Sólo guarda los casos que no son un valor: por qué está vacío.
CREATE TABLE campo_sin_valor (
  sueno_id TEXT NOT NULL REFERENCES sueno(id),
  campo    TEXT NOT NULL REFERENCES campo_catalogo(campo),
  motivo   TEXT NOT NULL CHECK (motivo IN ('no registrado','pendiente')),
  PRIMARY KEY (sueno_id, campo)
);
```

Un campo vacío sin fila aquí significa «la ficha todavía no pasó por revisión».
Al confirmar una ficha, todo campo vacío recibe explícitamente `no registrado`:
después de confirmar, la distinción es total y no queda ningún vacío ambiguo.
`prioridad_repregunta` es el orden fijo de P3, en datos y no en el código.

### 3.5 Facetas de escala y cadena

```sql
-- MUNDO (§5.3): cadena ordenada; en la distribución cuenta el último valor.
CREATE TABLE mundo_valor (valor TEXT PRIMARY KEY, nombre TEXT NOT NULL, en_escala INTEGER NOT NULL);
-- ('1','idéntico',1) … ('5','fantástico',1), ('abstracto','abstracto',0)

CREATE TABLE sueno_mundo (
  sueno_id TEXT NOT NULL REFERENCES sueno(id),
  orden    INTEGER NOT NULL,
  valor    TEXT NOT NULL REFERENCES mundo_valor(valor),
  PRIMARY KEY (sueno_id, orden)
);

-- TONO (§5.4): emoción + intensidad 1–7; puede venir sin cifra.
CREATE TABLE sueno_tono (
  sueno_id    TEXT NOT NULL REFERENCES sueno(id),
  orden       INTEGER NOT NULL,
  emocion     TEXT NOT NULL,
  intensidad  INTEGER CHECK (intensidad IS NULL OR intensidad BETWEEN 1 AND 7),
  PRIMARY KEY (sueno_id, orden)
);

-- ESTADO (§5.6): encadenable, con matiz libre ("casi lúcido").
CREATE TABLE sueno_estado (
  sueno_id TEXT NOT NULL REFERENCES sueno(id),
  orden    INTEGER NOT NULL,
  valor    TEXT NOT NULL CHECK (valor IN (
             'no lúcido','lúcido','falso despertar','parálisis','OBE','hipnagógico')),
  matiz    TEXT,
  PRIMARY KEY (sueno_id, orden)
);

-- TÉCNICA (§5.5): lista cerrada; 'ninguna' y 'no declarada' son valores reales.
CREATE TABLE tecnica_catalogo (tecnica TEXT PRIMARY KEY);
-- MILD · chequeos · meditación guiada · SSILD · reentrada · ninguna · no declarada

CREATE TABLE sueno_tecnica (
  sueno_id TEXT NOT NULL REFERENCES sueno(id),
  tecnica  TEXT NOT NULL REFERENCES tecnica_catalogo(tecnica),
  fuente   TEXT NOT NULL CHECK (fuente IN ('declaración de la noche','declarada por Roberto')),
  PRIMARY KEY (sueno_id, tecnica)
);
```

`fuente` existe para que quede escrito que una técnica llegó copiada de la
declaración de la noche según la regla P6, y nunca inferida. Sin declaración
aplicable, la técnica del sueño es `no declarada`, puesta por código.

### 3.6 Noche y mazo (§4.8, §6.1)

```sql
CREATE TABLE carta_mazo (
  numero INTEGER PRIMARY KEY CHECK (numero BETWEEN 1 AND 44),
  nombre TEXT NOT NULL
);

CREATE TABLE noche (
  id             INTEGER PRIMARY KEY,
  fecha          TEXT NOT NULL,          -- día civil de la declaración
  declarado_en   TEXT NOT NULL,          -- momento exacto; define a qué sueños aplica (P6)
  mazo_activado  INTEGER NOT NULL DEFAULT 0 CHECK (mazo_activado IN (0,1)),
  carta_numero   INTEGER REFERENCES carta_mazo(numero),
  carta_nombre   TEXT,
  foto_id        INTEGER REFERENCES archivo(id),
  CHECK (mazo_activado = 1 OR (carta_numero IS NULL AND carta_nombre IS NULL))
);

CREATE TABLE noche_tecnica (
  noche_id INTEGER NOT NULL REFERENCES noche(id),
  tecnica  TEXT NOT NULL REFERENCES tecnica_catalogo(tecnica),
  PRIMARY KEY (noche_id, tecnica)
);

-- Carta tirada en taller sobre un sueño ya registrado (§6.5.7)
CREATE TABLE carta_tirada (
  id           INTEGER PRIMARY KEY,
  numero       INTEGER REFERENCES carta_mazo(numero),
  nombre       TEXT,
  momento      TEXT NOT NULL CHECK (momento IN ('noche','taller')),
  noche_id     INTEGER REFERENCES noche(id),
  sueno_id     TEXT REFERENCES sueno(id),
  sesion_id    TEXT REFERENCES sesion_taller(id),
  foto_id      INTEGER REFERENCES archivo(id),
  tirada_en    TEXT NOT NULL
);
```

Hace falta la lista de las 44 cartas (número + nombre) para sembrar
`carta_mazo`: queda como pendiente P10.

### 3.7 Correcciones de transcripción (invariante 3)

```sql
CREATE TABLE correccion (
  id              INTEGER PRIMARY KEY,
  dictado_uuid    TEXT NOT NULL REFERENCES dictado(uuid_local),
  inicio          INTEGER NOT NULL,
  fin             INTEGER NOT NULL,
  texto_original  TEXT NOT NULL,     -- copia de lo que dice el crudo en ese rango: ancla verificable
  texto_corregido TEXT NOT NULL,
  propuesta_por   TEXT NOT NULL CHECK (propuesta_por IN ('modelo','Roberto')),
  confirmada      INTEGER NOT NULL DEFAULT 0 CHECK (confirmada IN (0,1)),
  descartada      INTEGER NOT NULL DEFAULT 0 CHECK (descartada IN (0,1)),
  respuesta_id    INTEGER REFERENCES respuesta_modelo(id),
  creada_en       TEXT NOT NULL,
  confirmada_en   TEXT,
  CHECK (fin > inicio),
  CHECK (confirmada = 0 OR confirmada_en IS NOT NULL)
);
```

Las correcciones se anclan al **dictado**, no al sueño: si un dictado se parte en
dos, las posiciones siguen siendo válidas. Al insertar se verifica que
`texto_original` sea exactamente el fragmento del crudo en ese rango. La vista
«con correcciones marcadas» (§6.4) aplica sólo las confirmadas, al momento de
mostrar; el crudo guardado sigue intacto.

### 3.8 Entidades y apariciones (§4.4, §4.5, invariante 9)

```sql
CREATE TABLE entidad (
  id                    INTEGER PRIMARY KEY,
  tipo                  TEXT NOT NULL CHECK (tipo IN ('lugar','personaje','símbolo','motivo')),
  nombre_canonico       TEXT NOT NULL,
  descripcion_acumulada TEXT,          -- lugares y personajes
  forma                 TEXT,          -- símbolos: forma fija, dibujable (§5.9)
  nota_editorial        TEXT,          -- motivos y notas heredadas de la migración
  creada_en             TEXT NOT NULL,
  fusionada_en_id       INTEGER REFERENCES entidad(id),   -- si se fusionó, apunta a la que quedó
  UNIQUE (tipo, nombre_canonico),
  CHECK (tipo <> 'símbolo' OR forma IS NOT NULL)
);

CREATE TABLE entidad_alias (
  entidad_id INTEGER NOT NULL REFERENCES entidad(id),
  alias      TEXT NOT NULL,
  PRIMARY KEY (entidad_id, alias)
);

CREATE TABLE aparicion (
  id                   INTEGER PRIMARY KEY,
  sueno_id             TEXT NOT NULL REFERENCES sueno(id),
  entidad_id           INTEGER NOT NULL REFERENCES entidad(id),
  rol                  TEXT NOT NULL CHECK (rol IN ('elenco','escenario','símbolo','motivo')),
  forma_en_este_sueno  TEXT,
  nota                 TEXT,           -- "la madre sólo como lugar, no suma como personaje"
  UNIQUE (sueno_id, entidad_id, rol)
);

-- Fusionar o partir entidades es una acción explícita de Roberto, nunca automática.
CREATE TABLE entidad_operacion (
  id         INTEGER PRIMARY KEY,
  tipo       TEXT NOT NULL CHECK (tipo IN ('fusión','partición','alias')),
  origen_id  INTEGER REFERENCES entidad(id),
  destino_id INTEGER REFERENCES entidad(id),
  sesion_id  TEXT REFERENCES sesion_taller(id),
  autor      TEXT NOT NULL CHECK (autor = 'Roberto'),
  nota       TEXT,
  hecha_en   TEXT NOT NULL
);
```

`autor CHECK (autor = 'Roberto')` es la forma más corta de decir en el esquema
que no existe fusión automática: no hay otro valor que la base acepte.

La recurrencia **no se guarda**: `v_recurrencia` cuenta sueños distintos por
entidad, y «recurrente» es `>= 2`.

### 3.9 Asignaciones (§4.6)

```sql
CREATE TABLE asignacion (
  id                 TEXT PRIMARY KEY,             -- 'A-002'
  numero             INTEGER NOT NULL UNIQUE,
  sueno_id           TEXT REFERENCES sueno(id),
  estrato            TEXT NOT NULL CHECK (estrato IN ('I','II','III')),
  estrato_raiz       TEXT CHECK (estrato_raiz IS NULL OR estrato_raiz IN ('I','II','III')),
  afirmacion         TEXT NOT NULL,
  como_se_verifica   TEXT,
  modo_verificacion  TEXT CHECK (modo_verificacion IS NULL OR modo_verificacion IN (
                       'documento','testimonio','testimonio dentro del crudo',
                       'testimonio en sesión de taller')),
  pendiente_foto     INTEGER NOT NULL DEFAULT 0 CHECK (pendiente_foto IN (0,1)),
  resultado          TEXT,
  abierta_en_sesion  TEXT REFERENCES sesion_taller(id),
  cerrada_en_sesion  TEXT REFERENCES sesion_taller(id),
  anulada            INTEGER NOT NULL DEFAULT 0 CHECK (anulada IN (0,1)),
  creada_en          TEXT NOT NULL,
  CHECK (id = 'A-' || substr('000' || numero, -3))
);

-- El estado vive en su historial: el PDF necesita saber en qué mes cambió.
CREATE TABLE asignacion_estado (
  id            INTEGER PRIMARY KEY,
  asignacion_id TEXT NOT NULL REFERENCES asignacion(id),
  estado        TEXT NOT NULL CHECK (estado IN ('pendiente','verificada','refutada','no verificable')),
  fecha         TEXT NOT NULL,
  sesion_id     TEXT REFERENCES sesion_taller(id),
  nota          TEXT,
  registrado_en TEXT NOT NULL
);
```

El estado actual es el último del historial (`v_asignacion_actual`). Los cuatro
valores son fijos y los informes siempre agrupan por los cuatro, incluso cuando
alguno va en cero.

### 3.10 Taller (§4.7, invariante 6)

```sql
CREATE TABLE sesion_taller (
  id                 TEXT PRIMARY KEY,             -- 'T-001'
  numero             INTEGER NOT NULL UNIQUE,
  fecha              TEXT NOT NULL,
  que_se_miro        TEXT,
  fuera_del_corpus   TEXT,                         -- testimonio de vigilia, opcional
  distincion_fijada  TEXT,
  abre_cierra        TEXT,
  cerrada            INTEGER NOT NULL DEFAULT 0 CHECK (cerrada IN (0,1)),
  creada_en          TEXT NOT NULL,

  -- VEREDICTO: sólo de Roberto.
  veredicto_texto          TEXT,                   -- lo que quedó como veredicto
  veredicto_texto_dictado  TEXT,                   -- lo que escribió, antes de la versión con puntuación
  veredicto_fluidez_id     INTEGER REFERENCES respuesta_modelo(id),
  veredicto_lectura_id     INTEGER REFERENCES lectura_claude(id),
  veredicto_origen         TEXT CHECK (veredicto_origen IS NULL OR veredicto_origen IN (
                             'propio','aprobación de la lectura de Claude','mixto')),

  CHECK (
    -- sesión sin veredicto: dato válido
    (veredicto_origen IS NULL AND veredicto_texto IS NULL AND veredicto_lectura_id IS NULL)
    -- escrito por él
 OR (veredicto_origen = 'propio' AND veredicto_texto IS NOT NULL AND veredicto_lectura_id IS NULL)
    -- hizo suya la lectura: se guarda la REFERENCIA, no una copia del texto
 OR (veredicto_origen = 'aprobación de la lectura de Claude'
     AND veredicto_texto IS NULL AND veredicto_lectura_id IS NOT NULL)
 OR (veredicto_origen = 'mixto' AND veredicto_texto IS NOT NULL AND veredicto_lectura_id IS NOT NULL)
  ),
  CHECK (id = 'T-' || substr('000' || numero, -3))
);

CREATE TABLE sesion_sueno (
  sesion_id TEXT NOT NULL REFERENCES sesion_taller(id),
  sueno_id  TEXT NOT NULL REFERENCES sueno(id),
  orden     INTEGER NOT NULL,                      -- se trabajan uno por uno, en orden (§6.5.8)
  PRIMARY KEY (sesion_id, sueno_id)
);

-- El cómputo se recalcula siempre, pero se guarda lo que se aprobó ese día:
-- el corpus crece y el registro de la sesión tiene que seguir siendo fiel.
CREATE TABLE computo_snapshot (
  id            INTEGER PRIMARY KEY,
  sesion_id     TEXT NOT NULL REFERENCES sesion_taller(id),
  sueno_id      TEXT NOT NULL REFERENCES sueno(id),
  payload_json  TEXT NOT NULL,
  calculado_en  TEXT NOT NULL,
  aprobado_en   TEXT
);

CREATE TABLE lectura_claude (
  id            INTEGER PRIMARY KEY,
  sesion_id     TEXT NOT NULL REFERENCES sesion_taller(id),
  sueno_id      TEXT NOT NULL REFERENCES sueno(id),
  texto         TEXT NOT NULL,                     -- siempre se muestra marcada "hipótesis de Claude"
  respuesta_id  INTEGER NOT NULL REFERENCES respuesta_modelo(id),
  creada_en     TEXT NOT NULL
);
```

### 3.11 Lo que devuelve el modelo (§7.3)

```sql
CREATE TABLE respuesta_modelo (
  id              INTEGER PRIMARY KEY,
  tarea           TEXT NOT NULL,        -- ficha | sospechas | super_resumen | lectura | fluidez | …
  prompt_archivo  TEXT NOT NULL,        -- 'prompts/ficha/v1.md'
  prompt_sha256   TEXT NOT NULL,
  modelo          TEXT NOT NULL,
  entrada_json    TEXT,
  salida_json     TEXT,
  valida          INTEGER NOT NULL,     -- pasó el JSON Schema
  tokens_entrada  INTEGER,
  tokens_salida   INTEGER,
  sueno_id        TEXT REFERENCES sueno(id),
  sesion_id       TEXT REFERENCES sesion_taller(id),
  creada_en       TEXT NOT NULL
);
```

Cada respuesta queda guardada junto a la versión exacta del prompt que la produjo.

### 3.12 Cierres (§4.9)

```sql
CREATE TABLE cierre_semanal (
  id             INTEGER PRIMARY KEY,
  anio           INTEGER NOT NULL,
  semana         INTEGER NOT NULL,            -- semana ISO
  s_desde        TEXT REFERENCES sueno(id),
  s_hasta        TEXT REFERENCES sueno(id),
  resumen        TEXT,
  resumen_origen TEXT,
  resumen_respuesta_id INTEGER REFERENCES respuesta_modelo(id),
  cerrado_en     TEXT,
  UNIQUE (anio, semana)
);

CREATE TABLE dream_sign (
  id                 INTEGER PRIMARY KEY,
  cierre_semanal_id  INTEGER REFERENCES cierre_semanal(id),
  cierre_mensual_id  INTEGER REFERENCES cierre_mensual(id),
  texto              TEXT NOT NULL,
  orden              INTEGER NOT NULL,
  origen             TEXT NOT NULL,
  respuesta_id       INTEGER REFERENCES respuesta_modelo(id),
  CHECK ((cierre_semanal_id IS NULL) <> (cierre_mensual_id IS NULL))
);

-- Fotografía de la tasa en ese momento: un dato, no una meta (§5.10).
CREATE TABLE tasa_lucidez_snapshot (
  cierre_semanal_id  INTEGER NOT NULL REFERENCES cierre_semanal(id),
  tecnica            TEXT NOT NULL REFERENCES tecnica_catalogo(tecnica),
  noches_intentadas  INTEGER NOT NULL,
  noches_logradas    INTEGER NOT NULL,
  PRIMARY KEY (cierre_semanal_id, tecnica)
);

CREATE TABLE cierre_mensual (
  id                  INTEGER PRIMARY KEY,
  anio                INTEGER NOT NULL,
  mes                 INTEGER NOT NULL,
  tesis               TEXT,
  tesis_origen        TEXT,
  tesis_respuesta_id  INTEGER REFERENCES respuesta_modelo(id),
  hilos               TEXT,
  linea_arqueologia   TEXT,
  queda_abierto       TEXT,
  pdf_ruta            TEXT,
  pdf_sha256          TEXT,
  cerrado_en          TEXT,
  UNIQUE (anio, mes)
);

CREATE TABLE eco (
  id                 INTEGER PRIMARY KEY,
  cierre_mensual_id  INTEGER NOT NULL REFERENCES cierre_mensual(id),
  ids                TEXT NOT NULL,          -- S-IDs que lo forman
  texto              TEXT NOT NULL,
  origen             TEXT NOT NULL,
  respuesta_id       INTEGER REFERENCES respuesta_modelo(id),
  orden              INTEGER NOT NULL
);
```

### 3.13 Infraestructura del registro

```sql
CREATE TABLE disparador (
  clave              TEXT PRIMARY KEY,       -- 'soul_retrieval'
  estado             TEXT NOT NULL CHECK (estado IN ('no disparado','mostrado','pospuesto','definido')),
  cierres_restantes  INTEGER NOT NULL DEFAULT 0,
  actualizado_en     TEXT NOT NULL
);

-- Pendientes heredados de la migración (§10.4) y cosas que Roberto quiere revisar.
CREATE TABLE marca_revision (
  id          INTEGER PRIMARY KEY,
  tabla       TEXT NOT NULL,
  registro_id TEXT NOT NULL,
  motivo      TEXT NOT NULL,
  origen      TEXT NOT NULL CHECK (origen IN ('migración','app')),
  resuelta    INTEGER NOT NULL DEFAULT 0,
  creada_en   TEXT NOT NULL,
  resuelta_en TEXT
);

CREATE TABLE migracion_incidencia (
  id        INTEGER PRIMARY KEY,
  archivo   TEXT NOT NULL,
  linea     INTEGER,
  fragmento TEXT,
  motivo    TEXT NOT NULL
);

-- Fotos (carta del mazo, por ahora). Fuera de la base, cifradas; aquí sólo la referencia.
CREATE TABLE archivo (
  id         INTEGER PRIMARY KEY,
  ruta       TEXT NOT NULL,
  sha256     TEXT NOT NULL,
  tipo_mime  TEXT NOT NULL,
  bytes      INTEGER NOT NULL,
  creado_en  TEXT NOT NULL
);

-- Historial de ediciones de ficha (el crudo no tiene historial porque no cambia).
CREATE TABLE auditoria (
  id          INTEGER PRIMARY KEY,
  tabla       TEXT NOT NULL,
  registro_id TEXT NOT NULL,
  campo       TEXT,
  antes       TEXT,
  despues     TEXT,
  actor       TEXT NOT NULL CHECK (actor IN ('Roberto','sistema','migración')),
  en          TEXT NOT NULL
);
```

### 3.14 Vistas del cómputo (invariante 5)

Todo el cómputo del sistema sale de aquí, no del modelo:

| Vista | Qué entrega |
|---|---|
| `v_orden` | ORDEN derivado de fecha + hora |
| `v_mundo_final` | último valor de la cadena de MUNDO, para la distribución |
| `v_tono_principal` | la emoción de mayor intensidad por sueño; las sin cifra aparte |
| `v_recurrencia` | sueños distintos por entidad; recurrente = `>= 2` |
| `v_coocurrencia` | pares de entidades que aparecen en el mismo sueño, con cuenta |
| `v_distribucion_mundo` / `v_distribucion_tono` | **conteos por valor**, nunca promedios; fuera de escala en su propia fila |
| `v_aparicion_numero` | «2ª aparición» real acumulada del corpus, para el PDF (§8.1.4) |
| `v_tasa_lucidez` | noches intentadas vs. logradas por técnica, con `ninguna` y `no declarada` como control |
| `v_asignacion_actual` | estado vigente por asignación + tasa acumulada |
| `v_calendario` | por día: con registro / sin registro / con mazo |
| `v_semana_completa` | correlatividad de S-IDs y fichas confirmadas, para el cierre semanal |

---

## 4. La única puerta de entrada (§3.3)

```python
def registrar_sueno(crudo: str, metadatos: Metadatos, origen: Origen) -> SuenoRegistrado:
```

En una sola transacción:

1. inserta el `dictado` (o lo reconoce por `uuid_local` y no hace nada: la
   sincronización es idempotente),
2. toma el próximo S-ID de `serie_id` y lo anota en `id_emitido`,
3. inserta el `sueno` con `rango` = dictado completo y `estado_ficha = 'borrador'`,
4. asocia la declaración de noche vigente según la regla P6 y escribe las
   técnicas con `fuente = 'declaración de la noche'`, o `no declarada`,
5. marca como `pendiente` los campos que no vinieron,
6. encola la generación de la ficha borrador (una tabla `tarea` simple, sin
   broker: un hilo del backend la consume cuando hay red).

La PWA de hoy, el conector MCP de mañana y la importación de la migración
entran por esta función y por ninguna otra. El paso 6 es el único que habla con
el modelo, y ocurre **después** de que el crudo ya está guardado e inmutable:
si la API de Claude está caída o devuelve basura, el sueño ya está a salvo.

---

## 5. Captura sin conexión

- Al tocar «guardar»: se escribe en IndexedDB (`uuid_local` = UUIDv4 del
  dispositivo, `capturado_en` = reloj del dispositivo con offset) y la pantalla
  responde «guardado». Cero red en el camino crítico.
- Una cola en IndexedDB reintenta el `POST /api/dictados` con espera creciente;
  el servidor responde con el S-ID asignado y el registro local pasa a
  «sincronizado». Reintentar es inofensivo: el `uuid_local` es la llave.
- La pantalla de captura y su JS se precachean en la instalación del service
  worker, así que abre sin red y sin esperar nada.
- **La captura nunca pide contraseña.** La sesión es larga (cookie de 90 días,
  renovada en cada uso) y, si está vencida, la captura igual guarda en local: el
  login se pide recién al sincronizar, de día. Un invariante de usabilidad que
  deriva del invariante 7: a las 4 de la mañana no hay ninguna pantalla entre
  Roberto y el campo de texto.
- La repregunta única y la sugerencia única (§6.2) se calculan en el dispositivo
  con los datos locales. Sin red, sin modelo.
- En iOS conviene instalar la PWA a la pantalla de inicio (los datos de una PWA
  instalada no se desalojan como los de una pestaña) y se pide
  `navigator.storage.persist()` al instalar.

---

## 6. El modelo

- Un solo módulo (`llm/`) importa `anthropic`. Nadie más.
- Cada tarea tiene prompt en archivo versionado y JSON Schema estricto
  (`additionalProperties: false`, campos obligatorios declarados). **Ningún
  esquema tiene un campo donde quepa el crudo**: si el modelo devolviera el
  texto del sueño, la validación lo rechaza por esquema, no por buena voluntad.
- La respuesta se valida, se guarda en `respuesta_modelo`, y recién entonces se
  proyecta a las tablas del dominio, siempre en campos marcados como propuesta.
- El prompt del taller incluye el protocolo común (`prompts/protocolo-comun.md`):
  «si fuera mi sueño», Roberto es la autoridad final, el significado sale de su
  corpus, nada de diccionarios de símbolos, todo lo que no es cómputo va marcado
  como hipótesis. Una prueba verifica que todos los prompts de interpretación
  incluyan ese bloque.
- El modelo recibe el cómputo **ya calculado** y jamás una pregunta del tipo
  «¿cuántas veces apareció…?».

---

## 7. Pantallas

Orden de construcción y qué trae cada una, según §6. Nada de lo que está en §13
se construye, y la pantalla de captura tiene además una prueba que verifica que
su bundle no contiene las palabras prohibidas (racha, felicit-, «días sin»,
«¿soñaste?», conteos).

---

## 8. Seguridad y privacidad

### 8.1 El corpus no entra al repositorio

Esto no está escrito en la especificación pero se desprende del invariante 11:
los archivos `bitacora-*.md`, `entidades.md`, `taller.md`, la base y los
respaldos **no se versionan en GitHub** (que es un tercero). El `.gitignore`
los excluye explícitamente y las pruebas usan sueños sintéticos escritos para
eso. Los archivos de la migración se copian al VPS por Tailscale (`scp`), no
por el repositorio ni por el chat.

### 8.2 Resto

- Acceso sólo por Tailscale; nada escuchando en la IP pública (el servicio
  se liga a la interfaz `tailscale0`). HTTPS con los certificados de Tailscale.
- Login con contraseña (hash argon2id) sobre esa red, por si se pierde un
  dispositivo. Passkey (WebAuthn) queda como mejora posterior.
- Base cifrada con SQLCipher; la llave llega por `systemd` credential o variable
  de entorno. Dicho con honestidad: eso protege respaldos, snapshots y un disco
  robado, no un servidor comprometido en caliente. La alternativa más fuerte es
  un volumen LUKS que pide passphrase en cada arranque (cada reinicio del VPS
  necesita que estés presente). Ver la pregunta de cifrado más abajo.
- La llave de la API de Claude sólo en el servidor.
- Los logs registran tarea, duración y tamaño; **nunca** texto de sueños. Una
  prueba lo verifica pasando un sueño sintético con una cadena marcadora y
  revisando que no aparezca en ningún log.

### 8.3 Dictado: una advertencia

La especificación deja abierto usar «reconocimiento de voz del navegador». Hay
que saber que la Web Speech API de Chrome **envía el audio a servidores de
Google**: eso choca con el invariante 11 y además no funciona sin conexión. La
recomendación es el dictado del teclado nativo sobre un `<textarea>` normal: en
iPhone moderno y en Android con el paquete de idioma descargado es
reconocimiento en el dispositivo, funciona sin red, no pide permisos cada vez y
no requiere una línea de código. La prueba de una semana (P9) debería medir ese
camino, y el servicio por API queda como plan B si la calidad no alcanza.

---

## 9. Respaldos (§3.4)

- Diario: `VACUUM INTO` (snapshot consistente con WAL abierto) → `age -r <clave
  pública>` → envío fuera del VPS. La clave privada de `age` no vive en el VPS.
- Semanal: exportación `.md` completa, mismo cifrado, mismo destino.
- Retención: 30 diarias, semanales indefinidas.
- `restaurar.sh` documentado y **probado al cerrar la etapa 0**: restaura en un
  directorio temporal, corre `PRAGMA integrity_check`, recalcula los
  `texto_sha256` de todos los dictados y verifica que el rango de S-IDs no tenga
  huecos. Si eso no pasa, la etapa 0 no está cerrada.

---

## 10. Despliegue

Contenedor con el backend y el frontend ya compilado; unidad systemd con
reinicio automático; un `timer` para el respaldo diario. Actualizar es
`git pull && make desplegar` (reconstruye imagen, corre migraciones de esquema,
reinicia). Las migraciones de esquema son archivos numerados que sólo avanzan.
Todo escrito en `docs/despliegue.md` con los comandos exactos.

---

## 11. Pruebas de invariante

Esta es la parte que convierte las reglas en arquitectura. Cada invariante tiene
al menos una prueba que falla si alguien, en seis meses, lo rompe sin darse cuenta.

| # | Invariante | Prueba |
|---|---|---|
| 1 | Crudo inmutable | `UPDATE` y `DELETE` directos sobre `dictado` abortan; `PUT/PATCH/DELETE` sobre cualquier ruta de dictado devuelven 405; `repositorio/dictado.py` no define función de escritura más que `insertar`; barrido AST del backend buscando `UPDATE dictado` |
| 2 | El crudo no pasa por el modelo | ningún JSON Schema de `llm/esquemas/` admite un campo de texto largo del sueño; una respuesta simulada que incluye el crudo es rechazada; `registrar_sueno` guarda el crudo sin que el cliente de Anthropic haya sido llamado (doble espía) |
| 3 | Correcciones aparte | una corrección no confirmada no aparece en ninguna salida; el ancla se valida contra el crudo; el crudo sigue igual por hash después de 100 correcciones |
| 4 | IDs permanentes | borrar el último sueño no libera su número; el ID no depende de la fecha (dos sueños del mismo día, dos IDs); concurrencia: 50 registros simultáneos dan 50 IDs distintos y correlativos; A-001 nunca se emite |
| 5 | Cómputo separado | barrido AST: `computo/` no importa `anthropic`, `httpx`, `requests`, ni `llm`; `llm/` no escribe en ninguna tabla salvo `respuesta_modelo`; todo texto del modelo se guarda con marca de hipótesis |
| 6 | Veredicto sólo de Roberto | las combinaciones inválidas de `veredicto_*` fallan por CHECK; «hago mía esta lectura» guarda referencia y deja el texto nulo; no existe ruta ni función en `llm/` que escriba veredicto |
| 7 | Captura silenciosa | el bundle de la pantalla de captura no contiene las palabras prohibidas; con red cortada (Playwright) guarda y no hace ninguna petición; la repregunta se dispara una sola vez y por un solo campo |
| 8 | Degrada sin castigo | barrido del frontend completo: ninguna cadena de racha, día en blanco o recordatorio; no hay registro de notificaciones push |
| 9 | Facetas, no categorías | los ejes abiertos aceptan valores nuevos sin migración; los cerrados rechazan un valor fuera de lista |
| 10 | Sin diccionarios de símbolos | todos los prompts de interpretación incluyen el bloque del protocolo común; ningún archivo del repo mapea símbolo → significado |
| 11 | Privacidad | ningún log contiene el marcador del sueño sintético; el `.gitignore` cubre corpus, base y respaldos; el servicio no escucha fuera de `tailscale0` |
| §5.2 | FECHA por día calendario | 23:39 y 01:07 caen en días distintos; no existe el concepto de «noche» en el cálculo de fecha |
| §5.3 | MUNDO | `abstracto` nunca entra en la distribución 1–5 ni en ningún promedio; en una cadena cuenta el último valor |
| §5.5 | TÉCNICA | sin declaración, el sueño queda `no declarada`; nunca se propone una técnica |
| §8 | PDF determinista | los mismos datos producen bytes idénticos (metadata de reportlab fijada); el crudo que no cabe reduce letra hasta 8 pt y después interlineado, y nunca se trunca en silencio |
| §9 | Exportación fiel | ida y vuelta: exportar `.md` desde la base, reimportar en una base vacía, comparar todo |

---

## 12. Etapas, con criterio de cierre

**Etapa 0 — Infraestructura.** Esqueleto backend + PWA, login, esquema vacío con
todas las restricciones, respaldo cifrado fuera del VPS y restauración probada,
despliegue documentado.
*Cierra cuando:* la app abre desde el celular por Tailscale, pide login una vez,
y una restauración desde respaldo pasa las cuatro verificaciones de §9.
*Lo que es tuyo:* contratar/preparar el VPS, Tailscale en el VPS y los
dispositivos, la clave de la API. Yo escribo el runbook y los scripts.

**Etapa 1 — Datos y migración.** Esquema completo, parser de los `.md`,
importación con informe (§10.3), exportación `.md`, vistas de sólo lectura.
*Cierra cuando:* los 28 sueños, 21 asignaciones, sesiones y entidades están en la
base, el rango de IDs no tiene huecos, el informe no deja nada sin parsear, y la
exportación desde la base coincide con los originales salvo diferencias que
revisamos una por una. Antes del corte, nada de capturar en dos sistemas (§10.5).

**Etapa 2 — Captura y revisión.** Noche, captura sin conexión, sincronización con
ID del servidor, ficha borrador por API, sospechas de transcripción, bandeja con
resolución de entidades. Semana de prueba del dictado en paralelo, sin registrar
corpus oficial.

**Etapa 3 — Taller.** Panel con cómputo, avance por secciones, lectura del
modelo, veredicto con sus reglas, abrir/cerrar asignaciones, partir/fusionar,
carta.

**Etapa 4 — Cierres.** Semanal con verificación y tasa por técnica, dream signs,
disparador de soul retrieval, mensual con tesis y ecos.

**Etapa 5 — PDF.** Desde la base, con `layout.py`. Se compara el número de
agosto-septiembre regenerado contra el impreso.

**Etapa 6 — Visualizaciones.**

---

## 13. Lo que necesito de ti

**Bloquea la etapa 0:**
- P8: proveedor y ubicación del VPS, y destino de los respaldos externos.
- Decisión de cifrado en reposo (SQLCipher con llave en el host vs. volumen LUKS
  con passphrase en cada arranque).
- P7: ¿captura también el computador?
- Qué celular es (iPhone o Android), para el dictado y el almacenamiento de la PWA.

**Bloquea la etapa 1:**
- Los archivos del sistema actual: `bitacora-2026-08.md`, `bitacora-2026-09.md`,
  `bitacora-2026-10.md`, `indice.md`, `entidades.md`, `asignaciones.md`,
  `taller.md`, `cierres.md` si existe, y `generador-pdf-suenos 2.md` (de donde
  salen `datos.py` y `layout.py`). **No los subas al repositorio**: van al VPS por
  Tailscale (§8.1). Mientras no exista VPS, cualquier revisión de formato la
  hacemos sobre fragmentos recortados, no sobre el corpus completo.
- P1: numeración al partir un sueño. Probablemente se resuelve mirando cómo quedó
  S-0018/S-0019 en los archivos; si quedó una regla distinta a «próximo ID libre»,
  la implemento como la usaste.

**Bloquea la etapa 2:** P3 (orden de la repregunta), P5 (foto de la carta), P6
(a qué sueños aplica una declaración de noche), P9 (resultado de la prueba de
dictado), P10 **nuevo**: la lista de las 44 cartas del mazo, número y nombre.

**Bloquea la etapa 5:** P4 (color de los títulos del PDF: hay registro de
«morados confirmados» y el código usa ocre).

---

## 14. Decisiones que propongo tomar así (y que puedes vetar)

1. **ORDEN derivado** de fecha + hora en vez de columna editable (§3.3).
2. **Hora civil sin conversión de zona**: la hora del sueño es lo que marcaba el
   reloj; nada de UTC para los sueños.
3. **Sin ORM**, SQL a mano, por auditabilidad del invariante 1 y 5.
4. **Svelte 5** como framework del frontend.
5. **Snapshot del cómputo** guardado en cada sesión de taller, además de poder
   recalcularlo: el registro de la sesión tiene que ser fiel a lo que aprobaste
   ese día, aunque el corpus haya crecido.
6. **Dictado del teclado nativo** y no Web Speech API (§8.3).
7. **Cola de tareas en la propia base**, sin Redis ni Celery: un hilo que lee una
   tabla `tarea`. Un usuario, cero infraestructura extra.

---

## 15. Riesgos que veo

- **El cifrado honesto es incómodo.** Cualquier opción que no pida una
  passphrase en cada arranque deja la llave en el mismo servidor. Hay que
  elegir con los ojos abiertos.
- **La fidelidad de la migración.** El formato no uniforme de las bitácoras es
  el punto donde algo se puede perder en silencio. Mitigación: nada se importa
  «con tolerancia»; lo que el parser no entiende va a `migracion_incidencia` y
  queda en el informe, y la comparación de ida y vuelta es obligatoria.
- **Que el PDF regenerado no coincida con el impreso.** Esperable en detalles
  tipográficos. Hay que acordar qué cuenta como «coincide» antes de la etapa 5.
- **Depender de la API de Claude para la ficha.** Mitigado por diseño: el crudo
  se guarda antes y la ficha borrador se encola; si el modelo no está, hay una
  bandeja con sueños guardados sin ficha, y nada se pierde.
