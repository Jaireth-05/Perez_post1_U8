# Post-contenido — Unidad 8: Patrones Arquitectónicos II

## Descripción
Entrega del post-contenido de la Unidad 8 (Patrones de Diseño de Software, sexto semestre).
Es un único proyecto Spring Boot que gestiona hallazgos de auditoría interna. En la **Parte 1**
se construye con Clean Architecture; en la **Parte 2** se amplía con un dashboard para el comité
y una bitácora de cambios de estado para Cumplimiento, decidiendo con criterios si CQRS o Event
Sourcing realmente hacían falta.

## Parte 1 — Clean Architecture (Hallazgos de Auditoría)
El sistema está dividido en los cuatro círculos concéntricos, y el código solo depende hacia
adentro, en dirección a `domain/`:

- **Entities** (`domain/`): el Aggregate Root `HallazgoAuditoria` y sus Value Objects
  (`HallazgoId`, `Severidad`, `EstadoHallazgo`, `PlanRemediacion`, `TransicionInvalidaException`).
- **Use Cases** (`usecase/`): una interfaz por caso de uso, los puertos de salida (`port/`) y
  sus implementaciones sin anotaciones de framework (`impl/`).
- **Interface Adapters** (`adapter/`): el controller REST con sus DTOs y el manejador de errores
  (`in/web`), y las entidades JPA, repositorios Spring Data y adaptadores (`out/persistence`).
- **Frameworks & Drivers** (`config/` más Spring Boot, JPA y H2): `AuditoriaConfiguration`
  conecta explícitamente cada caso de uso con sus puertos..

```
src/main/java/com/example/auditoria/
├── domain/
│   ├── entity/HallazgoAuditoria.java            ← Aggregate Root
│   └── valueobject/                             ← HallazgoId, Severidad, EstadoHallazgo,
│                                                  PlanRemediacion, TransicionInvalidaException
├── usecase/
│   ├── *UseCase.java                            ← Registrar, IniciarRemediacion, Cerrar, Reabrir,
│   │                                              Consultar, ObtenerDashboard, ConsultarHistorial
│   ├── HallazgoNotFoundException.java
│   ├── port/                                    ← HallazgoRepositoryPort, HistorialAuditoriaPort
│   │                                              y los records de lectura (ConteoCategoria,
│   │                                              PromedioCategoria, DashboardAuditoriaView,
│   │                                              CambioEstadoView)
│   └── impl/                                    ← un Service por caso de uso
├── adapter/
│   ├── in/web/                                  ← HallazgoController, GlobalExceptionHandler, dto/
│   └── out/persistence/                         ← HallazgoJpaEntity, HallazgoJpaRepository,
│                                                  HallazgoRepositoryAdapter,
│                                                  HistorialCambioEstado* y HistorialAuditoriaAdapter
├── config/                                      ← AuditoriaConfiguration, TransactionalDecorators
└── AuditoriaHallazgosApplication.java
```

`domain/` y `usecase/` no importan Spring ni JPA; `ReglaDeDependenciaTest` lo comprueba en cada
build. `HallazgoAuditoriaTest` crea el agregado y lo hace transicionar usando solo JUnit, sin
levantar Spring.

## Parte 2 — Análisis costo-beneficio de CQRS/Event Sourcing
Los dos requisitos nuevos son un dashboard (conteos por severidad y estado, y promedio de días
hasta el cierre por área) y una trazabilidad legal de cada cambio de estado. Antes de programar
se evaluaron contra los criterios de la guía:

**Escala y carga.** Lo usan unos pocos auditores, y el comité consulta el dashboard una vez al
mes. No hay concurrencia real ni una brecha entre el volumen de lecturas y el de escrituras: ambos
son bajos. Mantener infraestructura distinta para leer y para escribir no resolvería ningún
problema que hoy exista.

**Complejidad de las consultas.** Todo el dashboard sale de una sola tabla con `GROUP BY`. Los dos
conteos se escriben en JPQL con *interface projections*; el promedio de días necesita `DATEDIFF`,
que JPQL no ofrece, así que se escribió como consulta SQL nativa sobre la misma tabla. En ningún
caso se requiere otra base de datos ni otro esquema.

**Consistencia.** Un reporte generado bajo demanda antes de la reunión es suficiente; no se pide
tiempo real. Como la lectura se hace sobre las mismas tablas donde se escribe, el resultado
siempre está al día, algo que un modelo de lectura separado (con consistencia eventual) no
garantizaría.

**Naturaleza de la trazabilidad exigida.** Cumplimiento quiere ver la cronología de cambios: de
qué estado a cuál, con qué motivo y cuándo. No necesita rearmar el hallazgo reproduciendo
eventos, porque su estado actual ya está guardado. Una bitácora adicional que conviva con ese
estado cubre la necesidad.

**Señales de sobre-ingeniería.** Falta un experto de negocio para modelar eventos, el equipo es
una persona sin experiencia previa en Event Sourcing, y el esfuerzo de dos modelos, replay y
versionado de eventos supera con creces un problema que se reduce a tres consultas y una tabla.

**Conclusión.** CQRS y Event Sourcing completos no se justifican por ahora. Se optó por la
extensión liviana: tres métodos de consulta agregada en el mismo `HallazgoRepositoryPort` y una
bitácora de solo inserción que no sustituye al estado actual.

## Decisiones de diseño
1. **`Severidad` (enum simple) frente a `EstadoHallazgo` (enum con máquina de estados).**
   `EstadoHallazgo` encierra una regla del negocio: solo son válidas las transiciones
   ABIERTO → EN_REMEDIACION → CERRADO → REABIERTO → EN_REMEDIACION. Por eso expone
   `puedeTransicionarA` y la regla vive junto al valor que gobierna. `Severidad` solo clasifica en
   cuatro niveles y no tiene reglas ni transiciones; añadirle métodos sería inventar
   comportamiento que nadie necesita. Lo contrario tendría sentido si la severidad llegara a tener
   una regla propia, como un plazo máximo de remediación según el nivel.
2. **`PlanRemediacion` como Value Object embebido y no como agregado aparte.** Un hallazgo no puede
   entrar a EN_REMEDIACION sin un plan válido ni cerrarse sin haberlo tenido, y esa condición debe
   cumplirse siempre dentro de la misma transacción. Aplicando el criterio de límite de
   consistencia transaccional (Sección 3.3 de la guía), lo que debe ser consistente de inmediato
   pertenece al mismo agregado. Con un agregado separado y su propio repositorio podría existir,
   por un instante, un hallazgo en remediación sin plan. Por eso es un record inmutable dentro de
   `HallazgoAuditoria`, persistido en la misma tabla.
3. **Extensión liviana en vez de CQRS/Event Sourcing completos.** Según los criterios de la
   Sección 7: no hay diferencia de escala entre leer y escribir, las consultas se resuelven con
   `GROUP BY` sobre un único esquema y el comité no exige tiempo real. Se ampliaron el puerto y el
   repositorio JPA existentes, sin repositorio ni modelo de lectura separados.
4. **Bitácora simple (`historial_cambios_estado`) en lugar de un Event Store.** Un Event Store
   obligaría a que el agregado dejara de guardar su estado y se reconstruyera por replay en cada
   lectura, un cambio profundo sobre algo que ya funciona, sin que nadie necesite estados
   intermedios. Las señales de sobre-ingeniería de la Sección 7.2 se cumplen (sin experto de
   eventos, sin experiencia del equipo, costo de dos modelos desproporcionado). La bitácora es
   solo de inserción: `HistorialAuditoriaPort` ofrece únicamente `registrar` y
   `listarPorHallazgo`, la entidad es inmutable y su repositorio no tiene métodos de borrado ni
   actualización. Guardar el hallazgo y anotar su cambio ocurren en una sola transacción, gracias
   a decoradores con `TransactionTemplate` definidos en `config/` (`TransactionalDecorators`), de
   modo que los casos de uso siguen libres de Spring. `HallazgoJpaEntity` continúa siendo la única
   fuente del estado actual.

## Notas y limitaciones
- `HallazgoAuditoria.reconstituir(...)` rehidrata el agregado desde la base sin reproducir
  transiciones, para conservar el estado y la fecha de cierre reales.
- Los casos de uso de consulta devuelven entidades de dominio; el controller las convierte en DTOs,
  así `usecase/` no depende de `adapter/`.
- La inmutabilidad de la bitácora se garantiza a nivel de aplicación. Una garantía legal más fuerte
  pediría permisos solo de `INSERT` en la base o encadenar hashes por registro.
- No hay autenticación en el laboratorio, por lo que la bitácora no guarda *quién* hizo el cambio;
  con seguridad real se añadiría un campo de usuario.
- El cálculo del promedio usa SQL propio de H2 (`DATEDIFF`).

## Cómo ejecutar
```
$ mvn clean package
$ mvn spring-boot:run
$ bash docs/probar.sh      # recorre todos los endpoints con curl
```
Endpoints: `POST /api/hallazgos`, `PATCH /api/hallazgos/{id}/iniciar-remediacion`,
`/cerrar`, `/reabrir`, `GET /api/hallazgos`, `GET /api/hallazgos/{id}`,
`GET /api/hallazgos/dashboard` y `GET /api/hallazgos/{id}/historial`.

## Evidencia
Capturas de los endpoints en `docs/capturas/`.
<!-- Insertar: ![registrar](docs/capturas/01-registrar.png) ... -->

## Herramientas utilizadas
- Java 17, Spring Boot 3.x, Spring Data JPA, H2
- Apache Maven, Postman/curl, Git, GitHub

## Conclusiones
Clean Architecture me permitió dejar las reglas del hallazgo en el dominio, probables sin Spring,
y relegar la base de datos y HTTP a los adaptadores. En la Parte 2 comprobé que un requisito que
suena a CQRS o Event Sourcing no obliga a usarlos: tras revisar escala, consultas y lo que Cumplimiento
realmente pide, bastó ampliar el repositorio existente y añadir una bitácora. Volvería a evaluar la
decisión si el dashboard debiera atender millones de hallazgos o usar otra tecnología, si Cumplimiento
pidiera reconstruir el estado de un hallazgo en una fecha pasada, o si otros sistemas debieran
reaccionar a cada cambio de estado.