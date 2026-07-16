# gestor-incidentes-servicios

**Gestión colaborativa de incidentes en servicios**

Aplicación web desarrollada como Trabajo Práctico de **Diseño de Sistemas (DDS)** en la
**UTN FRBA**. Es una plataforma donde comunidades de usuarios reportan, siguen y resuelven
**incidentes** que afectan a servicios prestados por distintas entidades (empresas de
transporte, líneas, estaciones, establecimientos y sus servicios asociados).

## De qué se trata

El sistema modela un problema real: los servicios que usamos a diario (un ascensor de una
estación, un baño de una terminal, una escalera mecánica, etc.) fallan, y esas fallas afectan
a mucha gente. La aplicación permite que las personas se organicen en **comunidades de interés**
para detectar esos problemas, informarlos y gestionar su ciclo de vida hasta la resolución.

Los ejes del dominio son:

- **Entidades y establecimientos.** Las entidades (empresas, líneas, estaciones) agrupan
  establecimientos, y cada establecimiento ofrece **servicios** concretos que pueden sufrir
  incidentes.
- **Incidentes.** Se abren cuando un servicio deja de funcionar correctamente, se asocian a una
  o más comunidades y se cierran cuando el problema se resuelve. El sistema mantiene su estado y
  permite consultarlos y generar informes.
- **Comunidades y roles.** Los usuarios se agrupan en comunidades, envían solicitudes de ingreso
  y son gestionados por administradores. Existen distintos **roles** (administrador, lector,
  prestador) con permisos diferenciados.
- **Notificaciones.** Ante un incidente, los miembros interesados son notificados por distintos
  medios (correo electrónico, SMS/WhatsApp vía Twilio), con estrategias de envío inmediato o
  diferido según la preferencia de cada usuario.
- **Carga masiva.** Se pueden incorporar organizaciones y datos a través de archivos CSV.
- **Informes.** El sistema genera informes de incidentes agregados según distintos criterios.

## Arquitectura

Es una aplicación web **MVC** con renderizado en el servidor:

- **Capa web (Javalin).** Un router central expone las rutas y delega en *controllers* creados a
  través de una *factory*. Incluye middleware de autenticación, manejo de sesiones y control de
  acceso por rol.
- **Vistas (Handlebars).** Las páginas se renderizan con plantillas `.hbs` integradas al motor de
  render de Javalin. Los estáticos (CSS, imágenes) se sirven desde `resources/public`.
- **Dominio (models).** El corazón del sistema: incidentes, entidades, establecimientos,
  servicios, comunidades, usuarios y roles, además de tareas programadas y estrategias de
  notificación.
- **Persistencia (JPA).** El estado se persiste mediante JPA (con `jpa-extras`), sobre **MySQL**
  en producción y **HSQLDB** en desarrollo/tests. Los repositorios encapsulan el acceso a datos.
- **Integraciones externas.** Consumo de una API de georreferenciación (provincias/municipios) vía
  Retrofit, envío de correos con Commons Email y de mensajería con el SDK de Twilio.
- **Jobs.** Notificadores periódicos ejecutados como tareas programadas (cron) para el envío
  diferido de avisos.

## Stack tecnológico

| Área | Tecnología |
|------|------------|
| Lenguaje | Java 17 |
| Servidor web | Javalin 5 |
| Vistas | Handlebars.java |
| Persistencia | JPA + `jpa-extras`, MySQL / HSQLDB |
| HTTP externo | Retrofit 2 + Gson |
| Notificaciones | Commons Email, Twilio SDK |
| CSV | Apache Commons CSV |
| Utilidades | Lombok |
| Tests | JUnit 5, Mockito |
| Calidad | Checkstyle (Google), SpotBugs, JaCoCo |
| Build | Maven |
| Deploy | Docker / Render (Procfile) |

## Estructura del repositorio

```
src/main/java/ar/edu/utn/frba/dds/
├── server/          # Javalin: App, Server, Router, handlers, middlewares, init
├── controllers/     # Controllers MVC (login, incidentes, comunidades, entidades, ...)
└── models/          # Dominio: incidentes, entidades, comunidades, servicios,
                     #          repositorios, notificaciones, tareas, jobs, georef
src/main/resources/
├── templates/       # Vistas Handlebars (.hbs)
└── public/          # Estáticos (css, img)
```

## Seguridad

La dependencia de Handlebars.java se mantiene en **4.5.2 o superior** para incorporar la
corrección de la vulnerabilidad de *path traversal* en `FileTemplateLoader`
([CVE-2026-55760](https://github.com/advisories/GHSA-r4gv-qr8j-p3pg)).

El stack de Jackson se fija en **2.18.8** para incorporar las correcciones de dos bypasses del
`PolymorphicTypeValidator` en `jackson-databind`: uno vía parámetros genéricos, que permitía colar
una clase denegada como argumento de tipo de un contenedor permitido
([CVE-2026-54512](https://github.com/advisories/GHSA-j3rv-43j4-c7qm)), y otro vía arrays, donde
`allowIfSubTypeIsArray()` no validaba el tipo del componente
([CVE-2026-54513](https://github.com/advisories/GHSA-rmj7-2vxq-3g9f)).

La versión se gobierna importando el `jackson-bom` desde `dependencyManagement` en el `pom.xml`, en
lugar de fijarla dependencia por dependencia. Esto es intencional: `jackson-databind` es una
dependencia directa, pero `jackson-core`, `jackson-annotations`, `jackson-datatype-jsr310` y
`jackson-dataformat-xml` entran de forma transitiva vía `twilio`. Subir solo `jackson-databind`
dejaría el resto del stack en una versión anterior, y Jackson no soporta esa mezcla en runtime. Al
agregar nuevas dependencias de Jackson, **no les pongas `<version>`**: dejá que el BOM las alinee.

---

> Proyecto académico — UTN FRBA, Diseño de Sistemas.
