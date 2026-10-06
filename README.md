# Banco XYZ - Microservicios, Seguridad OAuth2, Resiliencia, Kafka y Docker

## 1. Descripción del proyecto

Banco XYZ es una solución basada en una arquitectura de microservicios desarrollada con **Spring Boot** y **Spring Cloud**, orientada a la gestión de operaciones bancarias.

El proyecto integra diferentes componentes para separar las responsabilidades del sistema y facilitar su escalabilidad, mantenimiento y despliegue.

Durante esta versión se incorporan y consolidan las siguientes características:

* Arquitectura basada en microservicios.
* Configuración centralizada mediante Spring Cloud Config.
* Descubrimiento de servicios mediante Netflix Eureka.
* Comunicación entre microservicios mediante OpenFeign y Spring Cloud LoadBalancer.
* Seguridad mediante OAuth 2.0 y JWT.
* Tolerancia a fallos mediante Resilience4j.
* Comunicación asíncrona mediante Apache Kafka.
* Servicio independiente de auditoría mediante eventos Kafka.
* Procesamiento de datos mediante Spring Batch.
* Contenedorización de los microservicios mediante Docker.
* Orquestación de los componentes mediante Docker Compose.
* Persistencia mediante MySQL.
* Infraestructura Kafka desplegada mediante Docker Compose en una instancia EC2.

---

# 2. Objetivo

El objetivo del proyecto es implementar una plataforma bancaria distribuida utilizando una arquitectura de microservicios que permita:

* Gestionar cuentas y transacciones bancarias.
* Separar las responsabilidades funcionales mediante diferentes servicios.
* Centralizar la configuración de los microservicios.
* Registrar y descubrir servicios automáticamente.
* Proteger los endpoints mediante autenticación y autorización OAuth 2.0.
* Utilizar tokens JWT para la comunicación segura.
* Implementar mecanismos de tolerancia a fallos.
* Procesar eventos bancarios de manera asíncrona mediante Kafka.
* Registrar eventos de transacciones mediante un servicio de auditoría.
* Procesar información mediante trabajos batch.
* Ejecutar los microservicios de manera portable mediante contenedores Docker.
* Facilitar el despliegue de los componentes mediante Docker Compose.

---
# 3. Arquitectura de la solución

La arquitectura se divide en dos entornos principales:

1. **Docker Compose local**, donde se ejecutan los microservicios, MySQL y los componentes de infraestructura Spring Cloud.
2. **Amazon EC2**, donde se ejecuta exclusivamente la infraestructura Kafka.

Esta separación permite evitar sobrecargar la instancia EC2 con todos los microservicios, manteniendo Kafka desplegado de forma independiente.

```text
                         ┌──────────────────────────┐
                         │       AMAZON EC2         │
                         │                          │
                         │       KAFKA CLUSTER      │
                         │                          │
                         │  ┌───────┐ ┌───────┐     │
                         │  │Kafka 1│ │Kafka 2│...  │
                         │  └───────┘ └───────┘     │
                         │                          │
                         │  Kafka UI :8090          │
                         └────────────┬─────────────┘
                                      │
                         Kafka Bootstrap Servers
                         29092 / 39092 / 49092
                                      │
                                      │
┌─────────────────────────────────────▼─────────────────────────────────┐
│                         DOCKER COMPOSE LOCAL                          │
│                                                                       │
│  ┌──────────────────┐       ┌──────────────────────┐                  │
│  │  Config Server   │       │   Discovery Server   │                  │
│  │      :8888       │       │        :8761         │                  │
│  └──────────────────┘       └──────────┬───────────┘                  │
│                                        │                              │
│                  ┌─────────────────────┼─────────────────────┐        │
│                  │                     │                     │        │
│                  ▼                     ▼                     ▼        │
│          ┌────────────┐        ┌────────────┐        ┌────────────┐   │
│          │  BFF Web   │        │ BFF Mobile │        │  BFF ATM   │   │
│          │   :8082    │        │   :8083    │        │   :8084    │   │
│          │Resilience4j│        │Resilience4j│        │Resilience4j│   │
│          │    + JWT   │        │    + JWT   │        │    + JWT   │   │
│          └─────┬──────┘        └─────┬──────┘        └─────┬──────┘   │
│                │                     │                     │          │
│                └─────────────────────┼─────────────────────┘          │
│                                      │                                │
│                               OpenFeign                               │
│                                      │                                │
│                                      ▼                                │
│                             ┌─────────────────┐                       │
│                             │  Backend Core   │───────────────────────┼──► Kafka EC2
│                             │      :8081      │                       │
│                             └────────┬────────┘                       │
│                                      │                                │
│                                      ▼                                │
│                               ┌─────────────┐                         │
│                               │    MySQL    │                         │
│                               │    :3307    │                         │
│                               └─────────────┘                         │
│                                                                       │
│  ┌────────────────┐                  ┌──────────────────────────┐     │
│  │  Auth Server   │                  │    Auditoria Service     │◄────┼── Kafka
│  │     :9000      │                  │          :8086           │     │
│  │ OAuth2 + JWT   │                  │     Consumer Group       │     │
│  └───────┬────────┘                  │      auditoria-group     │     │
│          │                           └──────────────────────────┘     │
│          │                                                            │
│          │ JWT                                                        │
│          ▼                                                            │
│     BFF Web / Mobile / ATM                                            │
│                                                                       │
│  ┌─────────────────────┐                                              │
│  │     Banco Batch     │                                              │
│  │        :8080        │──────────────► MySQL                         │
│  └─────────────────────┘                                              │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

## Componentes principales

### Docker Compose local

El entorno local contiene los componentes de la aplicación y la infraestructura necesaria para su funcionamiento:

- **Config Server:** `:8888`
- **Discovery Server / Eureka:** `:8761`
- **Auth Server:** `:9000`
- **Backend Core:** `:8081`
- **BFF Web:** `:8082`
- **BFF Mobile:** `:8083`
- **BFF ATM:** `:8084`
- **Banco Batch:** `:8080`
- **Auditoria Service:** `:8086`
- **MySQL:** `:3307`

Los BFF utilizan **Resilience4j** para tolerancia a fallos y **JWT** para la seguridad de las solicitudes.

La comunicación entre los BFF y Backend Core se realiza mediante **OpenFeign**, utilizando el descubrimiento de servicios proporcionado por Eureka.

### Amazon EC2

La instancia EC2 se utiliza exclusivamente para ejecutar la infraestructura de **Apache Kafka**, compuesta por:

- Kafka Broker 1.
- Kafka Broker 2.
- Kafka Broker 3.
- ZooKeeper.
- Kafka UI.
- Servicio de inicialización de tópicos.

Los brokers Kafka se exponen mediante los puertos:

```text
29092
39092
49092
```

La interfaz Kafka UI está disponible mediante el puerto:

```text
8090
```

Los microservicios que utilizan Kafka se conectan al clúster mediante la variable:

```text
KAFKA_BOOTSTRAP_SERVERS
```

### Flujo de seguridad

El **Auth Server** funciona como Authorization Server y emite tokens JWT mediante OAuth 2.0.

El flujo de autenticación es:

```text
Cliente
   │
   │ Solicita token
   ▼
Auth Server :9000
   │
   │ JWT
   ▼
Cliente
   │
   │ Bearer Token
   ▼
BFF Web / Mobile / ATM
   │
   │ Validación JWT
   ▼
Backend Core
```

### Flujo de eventos

Backend Core publica los eventos de transacciones en Kafka.

```text
Backend Core
     │
     │ Publica evento
     ▼
Kafka Cluster - EC2
     │
     │ Consume evento
     ▼
Auditoria Service :8086
     │
     └── Consumer Group: auditoria-group
```

El tópico utilizado para las transacciones bancarias es:

```text
transacciones-bancarias
```



# 4. Arquitectura de despliegue con Docker

Los microservicios de la aplicación se encuentran contenedorizados mediante Docker.

El archivo principal:

```text
docker-compose.yaml
```

permite levantar los componentes de la aplicación junto con MySQL.

La infraestructura Kafka se mantiene separada y se ejecuta en una instancia EC2 mediante:

```text
docker-compose.kafka.yaml
```

Esta separación permite evitar el consumo excesivo de memoria en una única instancia EC2 y facilita la ejecución de los microservicios de manera local mediante Docker.

## 4.1 Componentes ejecutados mediante Docker Compose local

```text
Docker Compose
│
├── MySQL
├── Config Server
├── Discovery Server
├── Auth Server
├── Backend Core
├── BFF Web
├── BFF Mobile
├── BFF ATM
├── Banco Batch
└── Auditoria Service
```

## 4.2 Infraestructura Kafka en EC2

```text
EC2
│
├── ZooKeeper 1
├── ZooKeeper 2
├── ZooKeeper 3
│
├── Kafka Broker 1
├── Kafka Broker 2
├── Kafka Broker 3
│
├── Kafka UI
└── kafka-init
```

Los microservicios se conectan al clúster Kafka mediante la configuración:

```text
KAFKA_BOOTSTRAP_SERVERS
```

Los brokers utilizan los puertos:

```text
29092
39092
49092
```

La interfaz de Kafka UI utiliza:

```text
8090
```

---

# 5. Estructura del proyecto

El repositorio contiene los diferentes microservicios organizados de manera independiente:

```text
Banco-XYZ/
│
├── auth-server/
│   ├── src/
│   ├── Dockerfile
│   └── pom.xml
│
├── backend-core/
│   ├── src/
│   ├── Dockerfile
│   └── pom.xml
│
├── bff-web/
│   ├── src/
│   ├── Dockerfile
│   └── pom.xml
│
├── bff-mobile/
│   ├── src/
│   ├── Dockerfile
│   └── pom.xml
│
├── bff-atm/
│   ├── src/
│   ├── Dockerfile
│   └── pom.xml
│
├── config-server/
│   ├── src/
│   ├── Dockerfile
│   └── pom.xml
│
├── discovery-server/
│   ├── src/
│   ├── Dockerfile
│   └── pom.xml
│
├── banco-xyz-batch/
│   ├── src/
│   ├── Dockerfile
│   └── pom.xml
│
├── auditoria-service/
│   ├── src/
│   ├── Dockerfile
│   └── pom.xml
│
├── docker-compose.yaml
├── docker-compose.kafka.yaml
└── README.md
```

---

# 6. Microservicios

## 6.1 Auth Server

Responsable de la autenticación y autorización mediante **OAuth 2.0**.

Utiliza Spring Authorization Server y genera tokens JWT para los clientes autorizados.

Puerto:

```text
9000
```

El cliente utilizado para las pruebas de autenticación es:

```text
Client ID: banco-xyz-client
Client Secret: banco-xyz-secret
```

Scopes disponibles:

```text
cuentas.read
cuentas.write
```

Estas credenciales corresponden al entorno académico y de demostración. En un entorno productivo deberían almacenarse mediante variables de entorno o un gestor de secretos.

---

## 6.2 Backend Core

Contiene la lógica principal del sistema bancario.

Entre sus responsabilidades se encuentran:

* Gestión de cuentas.
* Gestión de transacciones.
* Validación de operaciones.
* Persistencia de información.
* Publicación de eventos en Kafka.
* Integración con Eureka.
* Comunicación con MySQL.

Puerto:

```text
8081
```

---

## 6.3 BFF Web

Backend for Frontend destinado a clientes web.

Puerto:

```text
8082
```

Utiliza:

* Spring Security.
* OAuth 2.0.
* JWT.
* OpenFeign.
* Eureka.
* LoadBalancer.
* Resilience4j.

---

## 6.4 BFF Mobile

Backend for Frontend destinado a clientes móviles.

Puerto:

```text
8083
```

Utiliza los mismos mecanismos de seguridad, descubrimiento y resiliencia utilizados por el BFF Web.

---

## 6.5 BFF ATM

Backend for Frontend destinado a operaciones realizadas desde cajeros automáticos.

Puerto:

```text
8084
```

Incluye:

* OAuth 2.0.
* JWT.
* Eureka.
* OpenFeign.
* LoadBalancer.
* Resilience4j.

---

## 6.6 Config Server

Centraliza la configuración de los diferentes microservicios mediante **Spring Cloud Config Server**.

Puerto:

```text
8888
```

Permite mantener la configuración separada de la lógica de cada aplicación.

---

## 6.7 Discovery Server

Implementa el descubrimiento de servicios mediante **Netflix Eureka**.

Puerto:

```text
8761
```

Los microservicios se registran en Eureka para poder ser descubiertos dinámicamente.

---

## 6.8 Banco Batch

Servicio encargado de ejecutar procesos batch mediante **Spring Batch**.

Puerto:

```text
8080
```

Los procesos incluyen:

* Procesamiento de transacciones.
* Cálculo de intereses.
* Generación de estados anuales.

Utiliza MySQL para almacenar la información de negocio y las tablas de metadatos de Spring Batch.

---

## 6.9 Auditoria Service

Servicio independiente encargado de consumir los eventos publicados por Backend Core mediante Apache Kafka.

Puerto:

```text
8086
```

Utiliza un grupo de consumidores:

```text
auditoria-group
```

y consume eventos desde el tópico:

```text
transacciones-bancarias
```

---

# 7. Seguridad OAuth 2.0 y JWT

La seguridad utiliza OAuth 2.0 mediante el flujo **Client Credentials**.

El flujo general es:

```text
Cliente
   │
   │ Solicita token
   ▼
Auth Server :9000
   │
   │ JWT
   ▼
Cliente
   │
   │ Authorization: Bearer <token>
   ▼
BFF
   │
   │ Validación JWT
   ▼
Endpoint protegido
```

El Auth Server funciona como **Authorization Server**, mientras que los BFF funcionan como **OAuth2 Resource Server**.

Los endpoints protegidos requieren un token JWT válido.

---

# 8. Resiliencia con Resilience4j

Los BFF implementan mecanismos de tolerancia a fallos mediante Resilience4j.

Se utilizan principalmente:

* Retry.
* Circuit Breaker.
* Fallback.
* Respuestas HTTP controladas ante indisponibilidad del Backend Core.

Esto permite evitar que una falla temporal del Backend Core provoque una falla completa de los clientes.

---

# 9. Comunicación asíncrona con Kafka

Backend Core publica eventos relacionados con las transacciones bancarias en el tópico:

```text
transacciones-bancarias
```

El servicio Auditoria consume estos eventos de forma independiente.

```text
Backend Core
      │
      │ Publica evento
      ▼
Kafka
      │
      │ Consume evento
      ▼
Auditoria Service
```

El evento contiene información asociada a la operación realizada, incluyendo datos como:

* Cuenta.
* Monto.
* Fecha.
* Tipo de evento.
* Saldo posterior.

El identificador de cuenta se utiliza como clave del mensaje para mantener el orden de los eventos asociados a una misma cuenta dentro de una partición.

---

# 10. Persistencia

El proyecto utiliza **MySQL 8.4** como sistema de gestión de base de datos.

Puerto expuesto localmente:

```text
3307
```

Base de datos:

```text
banco_xyz
```

Usuario:

```text
banco_user
```

La conexión utilizada por los microservicios dentro de Docker Compose utiliza el nombre del servicio MySQL:

```text
jdbc:mysql://mysql:3306/banco_xyz
```

Mientras que para ejecución local fuera de Docker se utiliza:

```text
jdbc:mysql://localhost:3307/banco_xyz
```

---

# 11. Tecnologías utilizadas

| Tecnología                  | Uso                                     |
| --------------------------- | --------------------------------------- |
| Java 21                     | Lenguaje de programación                |
| Spring Boot                 | Desarrollo de microservicios            |
| Spring Cloud                | Componentes de arquitectura distribuida |
| Spring Cloud Config         | Configuración centralizada              |
| Netflix Eureka              | Service Discovery                       |
| Spring Cloud OpenFeign      | Comunicación entre servicios            |
| Spring Cloud LoadBalancer   | Balanceo de carga                       |
| Spring Security             | Seguridad                               |
| Spring Authorization Server | OAuth 2.0                               |
| JWT                         | Autenticación basada en tokens          |
| Resilience4j                | Tolerancia a fallos                     |
| Spring Kafka                | Comunicación asíncrona                  |
| Apache Kafka                | Broker de eventos                       |
| Spring Batch                | Procesamiento batch                     |
| Spring Data JPA             | Persistencia                            |
| Hibernate                   | ORM                                     |
| MySQL 8.4                   | Base de datos                           |
| Docker                      | Contenedorización                       |
| Docker Compose              | Orquestación                            |
| Amazon EC2                  | Infraestructura para Kafka              |
| Maven                       | Gestión y construcción del proyecto     |
| Git / GitHub                | Control de versiones                    |

---

# 12. Puertos

| Componente        | Puerto |
| ----------------- | -----: |
| Banco Batch       |   8080 |
| Backend Core      |   8081 |
| BFF Web           |   8082 |
| BFF Mobile        |   8083 |
| BFF ATM           |   8084 |
| Auditoria Service |   8086 |
| Auth Server       |   9000 |
| Discovery Server  |   8761 |
| Config Server     |   8888 |
| MySQL             |   3307 |
| Kafka Broker 1    |  29092 |
| Kafka Broker 2    |  39092 |
| Kafka Broker 3    |  49092 |
| Kafka UI          |   8090 |

---

# 13. Requisitos

Para ejecutar el proyecto se requiere instalar:

* Java 21.
* Maven o Maven Wrapper.
* Docker.
* Docker Compose.
* Git.
* Acceso a la instancia EC2 donde se encuentra desplegada la infraestructura Kafka.

---

# 14. Configuración previa

Clonar el repositorio:

```bash
git clone <URL_DEL_REPOSITORIO>
```

Ingresar al directorio del proyecto:

```bash
cd <DIRECTORIO_DEL_PROYECTO>
```

Antes de iniciar los microservicios, se debe disponer de la infraestructura Kafka ejecutándose en la instancia EC2.

La dirección de los brokers debe configurarse mediante:

```text
KAFKA_BOOTSTRAP_SERVERS
```

Ejemplo:

```text
<EC2_HOST>:29092,<EC2_HOST>:39092,<EC2_HOST>:49092
```

Se recomienda utilizar variables de entorno para evitar definir directamente direcciones específicas de infraestructura en el código fuente.

---

# 15. Ejecución con Docker Compose

El archivo principal de la aplicación es:

```text
docker-compose.yaml
```

Para construir las imágenes y levantar todos los servicios:

```bash
docker compose up -d --build
```

Para comprobar el estado de los contenedores:

```bash
docker compose ps
```

Para visualizar los logs de un servicio específico:

```bash
docker compose logs -f backend-core
```

También se pueden consultar los logs de otros servicios:

```bash
docker compose logs -f auth-server
docker compose logs -f bff-web
docker compose logs -f bff-mobile
docker compose logs -f bff-atm
docker compose logs -f auditoria-service
docker compose logs -f banco-batch
```

Para detener los servicios:

```bash
docker compose down
```

El proyecto utiliza un volumen Docker para conservar los datos de MySQL:

```text
banco_xyz_data
```

Por esta razón, `docker compose down` no elimina los datos almacenados en la base de datos.

Para eliminar completamente los contenedores y el volumen de datos:

```bash
docker compose down -v
```

Este último comando debe utilizarse únicamente cuando se requiera reiniciar completamente la base de datos.

---

# 16. Ejecución de Kafka

La infraestructura Kafka se ejecuta de forma independiente en la instancia EC2.

El archivo utilizado es:

```text
docker-compose.kafka.yaml
```

Dentro de la instancia EC2:

```bash
docker compose -f docker-compose.kafka.yaml up -d
```

Para revisar los servicios:

```bash
docker compose -f docker-compose.kafka.yaml ps
```

La infraestructura contiene:

* 3 nodos ZooKeeper.
* 3 brokers Kafka.
* Kafka UI.
* Servicio de inicialización de tópicos.

El tópico utilizado por la aplicación es:

```text
transacciones-bancarias
```

---

# 17. Construcción individual de un microservicio

Cada microservicio puede ser construido individualmente mediante Maven.

Por ejemplo:

```bash
cd backend-core
.\mvnw.cmd clean package
```

Luego se puede construir su imagen Docker:

```bash
docker build -t banco-xyz-backend-core .
```

El mismo procedimiento puede aplicarse a los demás microservicios que poseen un `Dockerfile`.

---

# 18. Configuración para Docker

Los microservicios utilizan variables de entorno para adaptar sus conexiones cuando se ejecutan dentro de contenedores.

Entre las principales variables se encuentran:

```text
DB_URL
DB_USERNAME
DB_PASSWORD
EUREKA_SERVER_URL
AUTH_SERVER_URL
AUTH_SERVER_ISSUER
KAFKA_BOOTSTRAP_SERVERS
```

Ejemplo de conexión a MySQL desde Docker Compose:

```text
DB_URL=jdbc:mysql://mysql:3306/banco_xyz
```

Ejemplo de conexión al Discovery Server desde un contenedor:

```text
EUREKA_SERVER_URL=http://host.docker.internal:8761/eureka/
```

Ejemplo de conexión al Auth Server desde otro contenedor:

```text
AUTH_SERVER_URL=http://auth-server:9000
```

---

# 19. Control de versiones

El código fuente se mantiene en Git y GitHub.

Las modificaciones se registran mediante commits y ramas según las necesidades del desarrollo.

Ejemplo:

```bash
git add .
git commit -m "Implementa Docker y Docker Compose para microservicios"
git push origin main
```

---

# 20. Consideraciones de despliegue

La arquitectura separa los microservicios de la infraestructura Kafka.

Los microservicios y MySQL pueden ejecutarse mediante Docker Compose en el entorno local, mientras que Kafka se mantiene desplegado en una instancia EC2.

Esta estrategia permite reducir los requerimientos de memoria de la instancia EC2 y mantener el clúster Kafka independiente de los servicios de negocio.

La comunicación entre ambos entornos se realiza mediante los puertos expuestos de los brokers Kafka.

---

# 21. Resumen de componentes

```text
┌─────────────────────────────────────────────────────────┐
│                    BANCO XYZ                            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Config Server              :8888                       │
│  Discovery Server           :8761                       │
│  Auth Server                :9000                       │
│                                                         │
│  BFF Web                    :8082                       │
│  BFF Mobile                 :8083                       │
│  BFF ATM                    :8084                       │
│                                                         │
│  Backend Core               :8081                       │
│  Banco Batch                :8080                       │
│  Auditoria Service          :8086                       │
│                                                         │
│  MySQL                      :3307                       │
│                                                         │
│  ─────────────────────────────────────────────          │
│                                                         │
│  Kafka Cluster - EC2                                    │
│  Broker 1                  :29092                       │
│  Broker 2                  :39092                       │
│  Broker 3                  :49092                       │
│  Kafka UI                  :8090                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```
