# Auto-Ontology (NVIDIA) · Guía rápida en español

Proyecto original: [NVIDIA/auto-ontology](https://github.com/NVIDIA/auto-ontology) · [Português](README.pt.md)


## El problema que resuelve

Un agente puede escribir SQL sin despeinarse. El peligro no es la sintaxis: es que entienda mal qué significa «ingresos» en tu empresa, una dos tablas que no debía, o responda con total seguridad a una pregunta que tus datos no pueden responder.

Auto-Ontology, el framework de código abierto de NVIDIA (Apache-2.0), ataca eso antes de la primera consulta: compila los metadatos de tus bases de datos en una **ontología** — términos de negocio ligados a columnas, expresiones SQL de métricas ya revisadas y relaciones entre tablas. Una persona experta revisa y **certifica** esas definiciones en una interfaz web, y a partir de ahí cualquier agente consulta a través de MCP, skills oficiales o la API REST usando ese diccionario certificado.

Dicho corto: antes de dejar que la IA consulte tus cuentas, le haces aprender el vocabulario que tú mismo aprobaste; y si la pregunta no tiene respuesta en los datos, tiene que decirlo en vez de inventarla.

## Los cinco movimientos

| Paso | Qué pasa |
|---|---|
| **Discover** | Genera la ontología desde los metadatos de la base, o importa modelos semánticos que ya tengas, y la muestra como grafo |
| **Ground** | Cada concepto de negocio queda con una sola definición, enlazada entre tablas y bases distintas |
| **Certify** | Una persona revisa, edita y certifica términos y SQL en la interfaz web o por API |
| **Validate** | Antes de consultar, comprueba si la pregunta se puede responder; cada respuesta vuelve con su SQL y los pasos que siguió |
| **Expose** | Sirve ese contexto a cualquier agente por MCP, skills y API REST |

## Frente a «pegar el esquema en el contexto»

| | Esquema pegado en el prompt | Auto-Ontology |
|---|---|---|
| Tokens | Crecen con cada tabla y columna | Solo los términos y tablas que la pregunta toca |
| Significado | Nombres de columna; el agente infiere el resto | Términos de negocio mapeados a columnas y a SQL |
| Joins y métricas | Improvisados en cada pregunta | Relaciones resueltas y expresiones SQL revisadas |
| Preguntas sin respuesta | Respondidas con confianza, a menudo mal | Comprobación de capacidad de respuesta: el agente pide aclaración |

## Quickstart en cinco pasos

Necesitas: Docker con Compose v2, [uv](https://docs.astral.sh/uv/) y una clave de API de NVIDIA (`nvapi-...`) de [build.nvidia.com](https://build.nvidia.com). El ejemplo corre contra Wide World Importers, la base de ventas de muestra de Microsoft.

**1. Clona y crea tu `.env`**

```bash
git clone https://github.com/NVIDIA/auto-ontology.git && cd auto-ontology
cp .env.example .env
```

Los valores que tienes que rellenar, uno a uno:

| Variable | Qué es |
|---|---|
| `DEFAULT_MODELS_API_KEY` | Tu clave `nvapi-...`; los roles de modelo la usan si no defines una por rol |
| `AUTH_SECRET` | Cadena larga y aleatoria (por ejemplo, `openssl rand -base64 32`) |
| `APP_URL` | `http://localhost:3000` en local |
| `POSTGRES_PASSWORD` | Contraseña nueva para Postgres |
| `CONNECTION_STRINGS` | Debe llevar **la misma** contraseña: `postgresql://postgres:<contraseña>@postgres:5432/wideworldimporters` |
| `AUTO_ONTOLOGY_ADMIN_PASSWORD` | Contraseña nueva del administrador de la web |
| `PGADMIN_PASSWORD` | Contraseña nueva para pgAdmin |

Aviso que el propio README del proyecto subraya: Compose publica Postgres, pgAdmin y la web en tu máquina, así que **cambia todas las contraseñas por defecto antes de arrancar**. El `.env.example` también trae claves por rol de modelo con el marcador `nvapi-your-api-key-here`: sustitúyelas por tu clave o borra esas líneas para que usen la de `DEFAULT_MODELS_API_KEY`.

**2. Levanta Postgres y siembra la base de ejemplo** (el script de siembra vive en el repo compañero `auto-ontology-eval`)

```bash
docker compose up -d --wait postgres
git clone https://github.com/NVIDIA/auto-ontology-eval.git ../auto-ontology-eval
(cd ../auto-ontology-eval && uv sync && cp .env.example .env && \
  POSTGRES_PASSWORD='<la misma contraseña>' \
  uv run python scripts/seed/seed_wwi.py --database-name wideworldimporters --drop)
```

**3. Levanta el resto de la pila**

```bash
docker compose up -d --build
```

**4. Activa la compilación semántica**

Abre `http://localhost:3000`, entra con el correo de administrador (`AUTO_ONTOLOGY_ADMIN_EMAIL` de tu `.env`) y la contraseña del paso 1, ve a **Settings → Semantic Compilation** y enciéndelo. La ingestión corre al arrancar; cuando la compilación termina verás «Semantic compilation is ready».

**5. Pregunta en Chat**

Algo como «¿cuáles fueron los ingresos del USB food flash drive - sushi roll en Pensilvania en junio de 2015?» La respuesta vuelve con la cifra, el SQL ejecutado y los pasos: qué términos resolvió, qué joins siguió, qué validó. Esa trazabilidad es justo el punto del proyecto.

Si la interfaz carga sin el menú lateral tras un reinicio, es la cookie de sesión vieja: borra los datos del sitio `localhost:3000` y entra de nuevo.

## Cómo leer su evaluación (sin dejarte llevar por un solo número)

El repo compañero [auto-ontology-eval](https://github.com/NVIDIA/auto-ontology-eval) compara al **mismo modelo** en dos brazos sobre benchmarks como BIRD, BEAVER, FDABench-Lite y Wide World Importers:

- **Control**: el modelo recibe solo el esquema de la base en el prompt.
- **Tratamiento**: el agente de Auto-Ontology trabaja desde la ontología compilada.

Un verificador determinista puntúa el SQL de cada brazo por coincidencia de ejecución. El informe da la diferencia de precisión **y la lista de preguntas que la ontología arregló y las que estropeó**. Léelo así, pregunta a pregunta; un único porcentaje esconde justo lo que necesitas saber.

## Datos prácticos de despliegue

- Conecta con diez motores: PostgreSQL, MySQL, Databricks, Snowflake, ClickHouse, Trino, Apache Kyuubi, HeavyDB, DuckDB y SQLite.
- Puertos: web `3000`, backend `3001`, ingestión `3002`, MCP `3003`, Postgres `5432`. Los puertos 3001 y 3002 deben quedarse en red privada: el servicio de ingestión no trae autenticación propia y el backend confía en la identidad que le pasa el frontend.
- Compatible con Apache Ossie, el estándar abierto para intercambiar modelos semánticos (importar/exportar desde Settings).

## Lo que conviene saber, dicho claro

- **No hay ninguna cifra pública de precisión text-to-SQL del proyecto.** El repositorio publica el marco de evaluación, no una nota: cualquiera que te venda un porcentaje concreto hoy está vendiendo humo.
- **El equipo de NVIDIA no acepta pull requests externos.** Issues de integración y reportes de error sí; código, no.
- Necesitas tu propia clave de API de NVIDIA y un Postgres propio: es un servicio que despliegas tú, no un SaaS.
- Las preguntas predictivas van a KumoRFM, un servicio externo al que se suben las tablas conectadas. Revisa qué envía antes de activarlo.

## Enlaces

- Repositorio: https://github.com/NVIDIA/auto-ontology
- Evaluación: https://github.com/NVIDIA/auto-ontology-eval
- Claves de API: https://build.nvidia.com
