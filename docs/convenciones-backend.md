# Convenciones de backend

Extraídas de `Narfware/enigmapp-backend` (commit `e7f7efa`). Son la referencia para el backend del
software de talleres. Cada regla describe lo que hace ese repo; las adaptaciones a Next.js y lo que
conviene no copiar van en secciones aparte, al final.

## Estructura de carpetas

```
src/<app>/
  index.ts                         arranque: lee el entorno, crea la conexión, inicia contenedores
  contexts/<contexto>/
    domain/
      <agregado>.ts                la entidad raíz, en la raíz de domain/
      value-objects/<vo>.ts
      repositories/<x>Repository.ts   interfaz
      interfaces/<puerto>.ts          resto de puertos de salida
      exceptions/<error>.ts
      events/<hechoEnPasado>.ts       eventos de dominio
    application/
      commands/<verboSustantivo>.ts   escrituras, un caso de uso por fichero
      queries/<verboSustantivo>.ts    lecturas, un caso de uso por fichero
      subscribers/<accion>On<Evento>.ts   reacciones a eventos de dominio
    infrastructure/
      <tecnologia><Puerto>.ts      adaptadores sueltos en la raíz de infrastructure/
      controllers/<casoDeUso>Controller.ts
      repositories/<tecnologia><X>Repository.ts
      persistence/postgres/drizzle/schemas/<x>Schema.ts
      dependency-injection/index.ts
  shared/
    dependency-injection/container.ts
    infrastructure/                conexión a base de datos, servidor
tests/<app>/                       espejo exacto de src/
  contexts/<contexto>/
    __mocks__/<puerto>Mock.ts
    domain/<Clase>Mother.ts
    application/commands/<casoDeUso>.test.ts
    application/queries/<casoDeUso>.test.ts
    infrastructure/...             un test por adaptador
  shared/infrastructure/persistence/<x>Arranger.ts
drizzle/                           migraciones SQL generadas
```

- Un contexto acotado por carpeta. Lo compartido entre contextos va a `shared/`.
- `tests/` replica la ruta de `src/` fichero a fichero. Los tests no viven junto al código.

## Nombres

- Ficheros en `camelCase.ts`: `authenticateChallenge.ts`, `postgresUserRepository.ts`.
  Excepción: los Object Mothers en `PascalCase` (`UserMother.ts`).
- Clases en `PascalCase`, una clase exportada por fichero, con export nombrado (sin `default`).
- Casos de uso: verbo + sustantivo, sin sufijo. `CreateUser`, no `CreateUserUseCase`.
- Puertos: nombre del rol, sin prefijo `I`. `UserRepository`, `TimeProvider`.
- Adaptadores: tecnología + puerto. `NodeCryptoNonceGenerator`, `DrizzlePostgresUserRepository`.
- Controladores: caso de uso + `Controller`.
- Excepciones: nombre del problema, sin sufijo `Error`. `InvalidNonce`.
- Tablas Drizzle: `<plural>Table` (`usersTable`); nombre SQL en plural y minúsculas.
- Código, nombres y tests en inglés. El texto que ve el usuario, en español.

## Dependencias entre capas

- `domain` no importa nada de fuera de `domain`: ni librerías, ni `application`, ni `infrastructure`.
- `application` importa solo de `domain`.
- `infrastructure` importa de `domain` y `application`, y es el único sitio con librerías externas
  (Express, Drizzle, `crypto`, `jsonwebtoken`).
- Imports relativos; no hay alias de rutas.

## Relación entre contextos

Decisión propia de narfcar: es un monolito modular, un solo despliegue con cada contexto aislado.

- Un contexto no importa el dominio de otro ni lee sus tablas.
- **Para leer de otro contexto se usa el query bus:** `await queryBus.ask(new GetWorkshopSettingsQuery(id))`.
  `QueryBus` es un puerto en `shared/domain`; la implementación es `InMemoryQueryBus`, síncrona.
- La clase de la query y el tipo de su respuesta son el contrato público del contexto dueño, y lo
  único suyo que otro contexto puede importar. Ambos son datos planos, sin dependencias.
- Quien responde es el caso de uso de `application/queries/` del contexto dueño, registrado como
  manejador de esa query. Dentro del propio contexto se le puede llamar directamente.
- No hay command bus: un contexto nunca ordena una escritura a otro. Si algo debe pasar en otro
  contexto, reacciona a un evento de dominio.
- Entre contextos solo viajan identificadores y primitivos, nunca entidades.
- Cada contexto es dueño de sus tablas, aunque compartan base de datos.
- `shared/` contiene solo lo común de verdad: identificadores, `Money`, `TimeProvider`, `EventBus`.
- La frontera de un contexto la marca el lenguaje: se separa cuando la misma palabra significa
  otra cosa, no por tipo de dato.

## Dominio y orientación a objetos

- **El comportamiento vive en la entidad.** El caso de uso no decide: `user.verifyNonce(...)`
  comprueba caducidad, valor y firma, y cambia el estado. Nada de entidades anémicas.
- **Estado privado, sin getters ni setters.** Campos `private`, identidad `private readonly`.
  El estado sale solo por `toPrimitives()`.
- **Serialización explícita:** cada entidad y value object tiene `toPrimitives()` y
  `static fromPrimitives()`, y la entidad exporta su tipo `Primitive<Entidad>`. Es la única frontera
  con persistencia y transporte.
- **Construcción:** `static create(...)` para crear algo nuevo según las reglas del dominio;
  `fromPrimitives` para rehidratar lo ya guardado.
- **Value objects** inmutables (`readonly`), con `equals()` y métodos que responden preguntas
  (`isExpired(now)`, `hasSameValue(value)`, `isBefore(time)`). Las constantes de negocio van junto
  al value object que las usa (`NONCE_EXPIRATION_TIME_IN_MINUTES`).
- **Nada de reloj ni azar directos en el dominio.** El tiempo entra por `TimeProvider` y lo
  aleatorio por un puerto (`NonceGenerator`); así los tests son deterministas.
- **Inyección por método (doble despacho):** cuando la entidad necesita un servicio, lo recibe como
  parámetro del método (`user.generateNonce(nonceGenerator)`), no lo guarda.
- **Errores de dominio como excepciones propias** que extienden `Error`, en `domain/exceptions/`.
  Se lanzan con cláusulas de guarda al principio del método.
- Los puertos devuelven objetos de dominio, nunca primitivos sueltos ni filas de base de datos.

## Casos de uso

- Una clase con un único método público `execute`.
- Dependencias por constructor, tipadas con las interfaces del dominio, como `private` en la firma.
- Entrada: un objeto de primitivos con tipo `<CasoDeUso>Params` declarado en el mismo fichero
  (o un primitivo suelto si solo hay uno). Nunca objetos de Express ni entidades.
- Forma fija: **cargar** el agregado por el repositorio, **invocar** un método del dominio,
  **guardar**, y devolver lo mínimo.
- Sin `if` de negocio en el caso de uso: las reglas están en el dominio.
- Si el dominio lanza, no se guarda (lo comprueba el test con `assertSaveNotCalled`).

## Commands y queries

Decisión propia de narfcar; no viene de enigmapp, donde los casos de uso no están separados.

- Cada caso de uso es un **command** (escribe) o una **query** (lee), nunca las dos cosas. Viven en
  `application/commands/` y `application/queries/`.
- **Command:** carga el agregado, invoca el dominio y guarda. Por norma no devuelve datos; si falla,
  lanza una excepción de dominio. Por pragmatismo puede devolver algo cuando hace falta, como el
  identificador de lo que acaba de crear, pero es la excepción.
- **Identificadores:** los genera el backend, en el caso de uso, a través de un puerto
  (`IdGenerator`), para no meter azar en el dominio. Las escrituras son síncronas.
- **Query:** no modifica nada ni llama a `save`. Devuelve primitivos pensados para la pantalla, no
  entidades. Puede leer por un puerto de lectura propio en lugar de reconstruir el agregado.
- La creación de un agregado es un command; cada cambio posterior es otro command distinto. En la
  recepción: `OpenWorkOrder` crea la orden y cada paso siguiente es una mutación con su caso de uso.
- Misma base de datos para escribir y leer, sin buses ni event sourcing, salvo que se decida otra cosa.

## Eventos de dominio

Decisión propia de narfcar; enigmapp no tiene eventos de dominio.

- Los agregados extienden `AggregateRoot`, que ofrece `record(event)` y `pullDomainEvents()`.
- Cada método que cambia el estado registra un evento. Los eventos viven en `domain/events/`, se
  nombran en pasado (`PhotoAttached`, `WorkOrderOpened`), son inmutables y llevan el id del agregado,
  el momento (del `TimeProvider`), el usuario y sus datos como primitivos.
- **Al repositorio solo se le pasa el agregado:** `save(order)`. Nunca eventos.
- El command guarda y después publica:
  `await repository.save(order)` y `await eventBus.publish(order.pullDomainEvents())`.
- `EventBus` es un puerto en `shared/domain`. La implementación es `InMemoryEventBus`: síncrona y
  esperada con `await` antes de responder, porque en Vercel la función se congela al devolver la
  respuesta. Sin broker externo.
- Los suscriptores viven en `application/subscribers/` y se nombran `<Acción>On<Evento>`.
- **El historial de un agregado es una proyección:** un suscriptor escribe cada evento en una tabla
  de solo añadir, y se lee con una query. El agregado no guarda su historial.
- No es event sourcing: el estado del agregado se guarda en sus tablas; los eventos son una salida.
- Limitación asumida: guardar el agregado y proyectar el evento no son atómicos. Si un suscriptor
  falla tras el guardado, falta esa entrada. Si llega a importar, se envuelve el command en una
  transacción compartida con los suscriptores, que al ser síncronos lo permiten.

## Infraestructura

- **Repositorios:** reciben la conexión por constructor. `save` es un upsert
  (`insert ... onConflictDoUpdate`) a partir de `toPrimitives()`. `find` devuelve la entidad con
  `fromPrimitives`. El mapeo entre columnas y primitivos se hace a mano en el repositorio.
- **Esquemas Drizzle** dentro del contexto, en `infrastructure/persistence/`. Los value objects se
  aplanan en columnas (`nonce`, `nonceExpirationTime`).
- **Controladores:** una clase con `execute(request, response)`. Solo extraen datos de la petición,
  piden el caso de uso al contenedor, lo ejecutan y responden. Sin lógica.
- **Inyección de dependencias:** a mano, sin librería. Cada contexto tiene su
  `dependency-injection/index.ts` que construye adaptadores y casos de uso y los registra en un
  `Container` (un `Map` indexado por la clase). Se expone `init<Contexto>Container(database)` y
  `get<Contexto>Container()`.
- **Configuración** por variables de entorno, leídas en el arranque o en el adaptador que las usa,
  con error explícito si faltan.

## Tests

- **Vitest**, importando `describe`, `it`, `expect`, `vi` de forma explícita.
- **Unitarios de casos de uso** (`tests/.../application/`): es donde se prueba el dominio. Se
  instancia el caso de uso con mocks en `beforeEach` y se llama a `vi.restoreAllMocks()` en
  `afterEach`.
- **Mocks escritos a mano**, uno por puerto, en `__mocks__/`. Implementan la interfaz y exponen:
  - `returnOn<Método>(valor)` para preparar la respuesta,
  - `assert<Método>HaveBeenCalledWith(...)` y `assert<Método>NotCalled()` para verificar,
  - un valor por defecto sensato (de un Mother) si no se prepara nada.
  Los `vi.fn()` quedan encapsulados; el test no los toca.
- **Object Mothers** en `tests/.../domain/`: `<Clase>Mother.create(...)` con valores por defecto y
  parámetros opcionales, más métodos con nombre para casos (`UserMother.nonceExpired()`,
  `NonceMother.expired()`).
- **Aserción sobre el agregado completo:** se compara lo guardado con una entidad esperada
  construida entera (`assertSaveHaveBeenCalledWith(new User(...))`), no campo a campo.
- **Integración de adaptadores** (`tests/.../infrastructure/`): contra la dependencia real.
  - Repositorios: Postgres real con Testcontainers. Un `Arranger` compartido levanta el contenedor,
    aplica las migraciones de `drizzle/`, trunca tablas en `beforeEach` y cierra en `afterAll`.
  - Controladores: servidor real con `supertest`, contenedor de dependencias real y Postgres real.
  - Adaptadores de librería (`crypto`, JWT): se ejercitan de verdad, con casos positivos y negativos.
- **Nombres:** `describe('<Caso de uso en lenguaje natural>')`, `it('Should ... when ...')`.
- **Orden:** preparar, ejecutar, verificar, separados por una línea en blanco.

## Estilo

- Prettier: 120 columnas, 4 espacios, comillas simples, sin punto y coma, sin coma final,
  flechas sin paréntesis con un solo parámetro.
- ESLint con `typescript-eslint` recomendado.
- TypeScript en modo `strict`.
- Cláusulas de guarda en una línea: `if (...) throw new X()`.
- Sin comentarios en el código: los nombres explican.

## Adaptación a Next.js en Vercel

Esto no viene de enigmapp; es lo que cambia al llevar las mismas convenciones al stack elegido.

- Los **controladores** pasan a ser Route Handlers y Server Actions: mismo papel, misma regla de no
  contener lógica.
- **Contenedor de dependencias:** en funciones sin estado no hay arranque único. El contenedor se
  crea de forma perezosa la primera vez que se pide en cada instancia.
- **Conexión a Postgres** a través del pooler de Supabase, con pocas conexiones por instancia.
- **Todas las escrituras** pasan por un caso de uso en el servidor. El navegador solo lee.
- **Tiempo real:** Supabase Realtime en lugar de socket.io.
- **Tests de integración:** Testcontainers se mantiene para repositorios; para los Route Handlers,
  se invoca el handler directamente en lugar de `supertest`.
- Los componentes React quedan fuera de estas reglas: son un adaptador de entrada más y no
  contienen reglas de negocio.

## Lo que no conviene copiar tal cual

Detalles vistos en enigmapp que parecen descuidos, no convenciones.

- `Time.addMinutes` y `subtractMinutes` modifican la fecha interna además de devolver un `Time`
  nuevo; un value object no debería mutar.
- `Nonce.equals` compara con `===` el resultado de `toPrimitives()`, que son dos objetos distintos,
  así que siempre da falso. En `Time` funciona porque el primitivo es un número.
- `find` no trata el caso de no encontrado: falla con un error de tipo en lugar de una excepción de
  dominio.
- No hay traducción de excepciones de dominio a códigos HTTP en los controladores.
- `createUserController.test.ts` importa `beforeEach` de `node:test` en vez de Vitest, así que la
  limpieza de la base de datos no se ejecuta.
- No hay tests propios de entidades ni value objects; se cubren solo a través de los casos de uso.
- No hay validación de la entrada en los controladores.
