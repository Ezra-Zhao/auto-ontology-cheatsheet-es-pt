# Auto-Ontology (NVIDIA) · Guía rápida en español y portugués

> 📚 **Colección / Coleção**: esta hoja forma parte de [cheatsheets](https://github.com/Ezra-Zhao/cheatsheets) — 100 hojas de referencia en un solo repositorio. / Esta folha faz parte de [cheatsheets](https://github.com/Ezra-Zhao/cheatsheets) — 100 folhas de referência em um só repositório. ⭐

[![Repo original](https://img.shields.io/github/stars/NVIDIA/auto-ontology?style=social)](https://github.com/NVIDIA/auto-ontology/stargazers) [![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

> 🌍 Español primero · Português mais abaixo · Proyecto original: [NVIDIA/auto-ontology](https://github.com/NVIDIA/auto-ontology)

---

# 🇪🇸 Español

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

---

# 🇧🇷 Português

## O problema que ele resolve

Um agente escreve SQL sem esforço. O perigo não é a sintaxe: é ele entender errado o que «receita» significa na sua empresa, juntar duas tabelas que não devia, ou responder com total segurança a uma pergunta que os seus dados não conseguem responder.

O Auto-Ontology, framework de código aberto da NVIDIA (Apache-2.0), ataca isso antes da primeira consulta: compila os metadados dos seus bancos de dados numa **ontologia** — termos de negócio ligados a colunas, expressões SQL de métricas já revisadas e relações entre tabelas. Uma pessoa especialista revisa e **certifica** essas definições numa interface web, e a partir daí qualquer agente consulta via MCP, skills oficiais ou API REST usando esse dicionário certificado.

Resumindo: antes de deixar a IA consultar suas contas, você faz ela aprender o vocabulário que você mesmo aprovou; e se a pergunta não tiver resposta nos dados, ela tem que dizer isso em vez de inventar.

## Os cinco movimentos

| Passo | O que acontece |
|---|---|
| **Discover** | Gera a ontologia a partir dos metadados do banco, ou importa modelos semânticos que você já tem, e mostra tudo como grafo |
| **Ground** | Cada conceito de negócio fica com uma única definição, ligada entre tabelas e bancos diferentes |
| **Certify** | Uma pessoa revisa, edita e certifica termos e SQL na interface web ou pela API |
| **Validate** | Antes de consultar, verifica se a pergunta dá para responder; cada resposta volta com seu SQL e os passos que seguiu |
| **Expose** | Serve esse contexto a qualquer agente por MCP, skills e API REST |

## Contra «colar o esquema no contexto»

| | Esquema colado no prompt | Auto-Ontology |
|---|---|---|
| Tokens | Crescem com cada tabela e coluna | Só os termos e tabelas que a pergunta toca |
| Significado | Nomes de coluna; o agente infere o resto | Termos de negócio mapeados para colunas e SQL |
| Joins e métricas | Improvisados a cada pergunta | Relações resolvidas e expressões SQL revisadas |
| Perguntas sem resposta | Respondidas com confiança, muitas vezes erradas | Checagem de capacidade de resposta: o agente pede esclarecimento |

## Quickstart em cinco passos

Você precisa de: Docker com Compose v2, [uv](https://docs.astral.sh/uv/) e uma chave de API da NVIDIA (`nvapi-...`) de [build.nvidia.com](https://build.nvidia.com). O exemplo roda contra o Wide World Importers, o banco de vendas de amostra da Microsoft.

**1. Clone e crie o seu `.env`**

```bash
git clone https://github.com/NVIDIA/auto-ontology.git && cd auto-ontology
cp .env.example .env
```

Os valores que você precisa preencher, um a um:

| Variável | O que é |
|---|---|
| `DEFAULT_MODELS_API_KEY` | Sua chave `nvapi-...`; os papéis de modelo a usam se você não definir uma por papel |
| `AUTH_SECRET` | String longa e aleatória (por exemplo, `openssl rand -base64 32`) |
| `APP_URL` | `http://localhost:3000` em local |
| `POSTGRES_PASSWORD` | Senha nova para o Postgres |
| `CONNECTION_STRINGS` | Precisa levar **a mesma** senha: `postgresql://postgres:<senha>@postgres:5432/wideworldimporters` |
| `AUTO_ONTOLOGY_ADMIN_PASSWORD` | Senha nova do administrador da web |
| `PGADMIN_PASSWORD` | Senha nova para o pgAdmin |

Aviso que o próprio README do projeto sublinha: o Compose publica Postgres, pgAdmin e a web na sua máquina, então **troque todas as senhas padrão antes de subir**. O `.env.example` também traz chaves por papel de modelo com o marcador `nvapi-your-api-key-here`: substitua pela sua chave ou apague essas linhas para que usem a de `DEFAULT_MODELS_API_KEY`.

**2. Suba o Postgres e semeie o banco de exemplo** (o script de semeadura fica no repo companheiro `auto-ontology-eval`)

```bash
docker compose up -d --wait postgres
git clone https://github.com/NVIDIA/auto-ontology-eval.git ../auto-ontology-eval
(cd ../auto-ontology-eval && uv sync && cp .env.example .env && \
  POSTGRES_PASSWORD='<a mesma senha>' \
  uv run python scripts/seed/seed_wwi.py --database-name wideworldimporters --drop)
```

**3. Suba o resto da pilha**

```bash
docker compose up -d --build
```

**4. Ative a compilação semântica**

Abra `http://localhost:3000`, entre com o e-mail de administrador (`AUTO_ONTOLOGY_ADMIN_EMAIL` do seu `.env`) e a senha do passo 1, vá a **Settings → Semantic Compilation** e ligue. A ingestão roda na inicialização; quando a compilação termina você vê «Semantic compilation is ready».

**5. Pergunte no Chat**

Algo como «qual foi a receita do USB food flash drive - sushi roll na Pensilvânia em junho de 2015?» A resposta volta com o valor, o SQL executado e os passos: quais termos ele resolveu, quais joins seguiu, o que validou. Essa rastreabilidade é justamente o ponto do projeto.

Se a interface carregar sem o menu lateral depois de um reinício, é o cookie de sessão antigo: apague os dados do site `localhost:3000` e entre de novo.

## Como ler a avaliação (sem se deixar levar por um número só)

O repo companheiro [auto-ontology-eval](https://github.com/NVIDIA/auto-ontology-eval) compara o **mesmo modelo** em dois braços sobre benchmarks como BIRD, BEAVER, FDABench-Lite e Wide World Importers:

- **Controle**: o modelo recebe só o esquema do banco no prompt.
- **Tratamento**: o agente do Auto-Ontology trabalha a partir da ontologia compilada.

Um verificador determinístico pontua o SQL de cada braço por coincidência de execução. O relatório traz a diferença de precisão **e a lista de perguntas que a ontologia consertou e as que estragou**. Leia assim, pergunta por pergunta; uma porcentagem única esconde justamente o que você precisa saber.

## Dados práticos de implantação

- Conecta com dez motores: PostgreSQL, MySQL, Databricks, Snowflake, ClickHouse, Trino, Apache Kyuubi, HeavyDB, DuckDB e SQLite.
- Portas: web `3000`, backend `3001`, ingestão `3002`, MCP `3003`, Postgres `5432`. As portas 3001 e 3002 devem ficar em rede privada: o serviço de ingestão não tem autenticação própria e o backend confia na identidade repassada pelo frontend.
- Compatível com Apache Ossie, o padrão aberto para trocar modelos semânticos (importar/exportar em Settings).

## O que saber, dito claramente

- **Não existe nenhuma cifra pública de precisão text-to-SQL do projeto.** O repositório publica o framework de avaliação, não uma nota: quem te vender uma porcentagem concreta hoje está vendendo fumaça.
- **A equipe da NVIDIA não aceita pull requests externos.** Issues de integração e relatos de erro, sim; código, não.
- Você precisa da sua própria chave de API da NVIDIA e de um Postgres seu: é um serviço que você implanta, não um SaaS.
- As perguntas preditivas vão para o KumoRFM, um serviço externo para o qual as tabelas conectadas são enviadas. Confira o que ele envia antes de ativar.

## Links

- Repositório: https://github.com/NVIDIA/auto-ontology
- Avaliação: https://github.com/NVIDIA/auto-ontology-eval
- Chaves de API: https://build.nvidia.com

---

Guía elaborada a partir de la documentación oficial del proyecto · Guia elaborada a partir da documentação oficial do projeto. El software pertenece a sus autores · O software pertence aos seus autores.

Licencias: Auto-Ontology es Apache-2.0; esta guía se publica bajo licencia MIT. · Licenças: o Auto-Ontology é Apache-2.0; este guia é publicado sob a licença MIT.

📣 Telegram 频道：https://t.me/EzraZhao777（各平台的更新都同步到这里）
