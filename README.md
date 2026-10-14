# Práctica: Despliegue e Instalación de tres SGBD en Linux (Oracle 23ai, PostgreSQL y MariaDB)

Este repositorio contiene la documentación detallada y las instrucciones paso a paso para el despliegue, configuración y verificación de tres Sistemas Gestores de Bases de Datos (SGBD) sobre una máquina virtual con sistema operativo Linux.

---

## Datos de la práctica

- **Nombre y Apellidos:** Roberto Muñoz Galán
- **Grupo:** ASIR 2
- **Título de la tarea:** Instalación de tres SGBD
- **Fecha:** 13/10/2026

---

## Índice

## Índice
1. [Entorno y Características del Sistema](#1-entorno-y-características-del-sistema)
2. [Despliegue y Configuración Paso a Paso](#2-despliegue-y-configuración-paso-a-paso)
   - 2.1. [Oracle Database 23ai Free con Podman](#21-oracle-database-23ai-free-con-podman)
   - 2.2. [PostgreSQL](#22-postgresql)
   - 2.3. [MariaDB](#23-mariadb)
3. [Resumen y Comprobaciones de Conexión](#3-resumen-y-comprobaciones-de-conexión)
4. [Configuración de Red para Acceso Remoto](#4-configuración-de-red-para-acceso-remoto)
   - 4.1. [PostgreSQL (Acceso Externo)](#41-postgresql-acceso-externo)
   - 4.2. [MariaDB (Acceso Externo)](#42-mariadb-acceso-externo)
5. [Incidencias Encontradas y Soluciones](#5-incidencias-encontradas-y-soluciones)
6. [Evidencias y Fuentes Consultadas](#6-evidencias-y-fuentes-consultadas)

---

## 1. Entorno y Características del Sistema

### Especificaciones de la Máquina Virtual

* **Sistema Operativo:** Linux Mint / Debian *(especificar versión exacta usada)*
* **Memoria RAM:** 4 GB
* **Procesadores (vCPU):** 2 cores
* **Almacenamiento:** 250 GB
* **Gestor de Contenedores:** Podman

### Versiones Instaladas de los SGBD

* **Oracle Database:** 23ai Free (v23.5.0.0) desplegado en Podman
* **PostgreSQL:** *(Indicar versión instalada, ej. PostgreSQL 15 / 16)*
* **MariaDB:** *(Indicar versión instalada, ej. MariaDB 10.11)*

---

## 2. Despliegue y Configuración Paso a Paso

### 2.1. Oracle Database 23ai Free con Podman

1. **Instalación de Podman y descarga de la imagen:**
   Actualizamos los repositorios del sistema e instalamos el motor de contenedores Podman. Posteriormente, descargamos la imagen oficial de Oracle Database 23ai Free.

   ```bash
   sudo apt update
   sudo apt install podman -y
   sudo podman pull container-registry.oracle.com/database/free:23.5.0.0
   sudo podman images
   ```

2. **Creación del volumen persistente y despliegue del contenedor:**
   Para asegurar la persistencia de los datos en caso de reinicio o eliminación del contenedor, creamos un volumen dedicado y desplegamos la base de datos publicando el puerto TCP `1521`.

   ```bash
   sudo podman volume create oracle_23ai_datos
   sudo podman run -d --name cont-oracle -p 1521:1521 --restart=always -v oracle_23ai_datos:/opt/oracle/oradata:Z container-registry.oracle.com/database/free:23.5.0.0
   ```

3. **Verificación del estado e inspección de logs:**
   Comprobamos la correcta ejecución del contenedor y consultamos los registros para obtener la contraseña auto-generada de administración (`SYS`).

   ```bash
   sudo podman ps
   sudo podman logs cont-oracle
   ```

4. **Creación del usuario de trabajo en la PDB (`FREEPDB1`):**
   Accedemos a la herramienta `sqlplus` dentro del contenedor como administrador `SYSDBA`, nos posicionamos en la Pluggable Database `FREEPDB1`, creamos el usuario de trabajo `orauser`, asignamos permisos básicos de conexión y creación de tablas, y le otorgamos cuota en el tablespace.

   ```bash
   sudo podman exec -it cont-oracle sqlplus / as sysdba
   ```

   *Instrucciones ejecutadas dentro de SQL\*Plus:*

   ```sql
   SHOW PDBS;
   ALTER SESSION SET CONTAINER = FREEPDB1;
   CREATE USER orauser IDENTIFIED BY "TU_CLAVE";
   GRANT CREATE SESSION, CREATE TABLE TO orauser;
   ALTER USER orauser QUOTA 100M ON USERS;
   EXIT;
   ```

---

### 2.2. PostgreSQL

1. **Instalación e inicio del servicio:**
   Instalamos el servidor de PostgreSQL junto con módulos adicionales y el controlador PHP correspondiente. Habilitamos e iniciamos el servicio en el sistema.

   ```bash
   sudo apt update
   sudo apt install postgresql postgresql-contrib php-pgsql -y
   sudo systemctl start postgresql
   sudo systemctl enable postgresql
   ```

2. **Creación de la base de datos y usuario:**
   Accedemos con la cuenta de sistema `postgres` para interactuar con la consola de `psql`, creamos la base de datos `dbpg` y asignamos privilegios al usuario `pguser`.

   ```bash
   sudo -i -u postgres
   psql
   ```

   *Instrucciones ejecutadas en `psql`:*

   ```sql
   CREATE DATABASE dbpg;
   CREATE USER pguser WITH PASSWORD 'TU_CLAVE';
   GRANT ALL PRIVILEGES ON DATABASE dbpg TO pguser;
   \q
   ```

3. **Cierre de sesión del usuario administrador:**

   ```bash
   exit
   ```

---

### 2.3. MariaDB

1. **Instalación y puesta en marcha del servicio:**
   Instalamos MariaDB Server y MariaDB Client, iniciando el servicio inmediatamente.

   ```bash
   sudo apt update
   sudo apt install mariadb-server mariadb-client -y
   sudo systemctl enable --now mariadb
   ```

2. **Creación de la base de datos y usuario local:**
   Accedemos mediante autenticación administrativa directa con socket (`sudo mariadb`).

   ```bash
   sudo mariadb
   ```

   *Instrucciones ejecutadas en MariaDB:*

   ```sql
   CREATE DATABASE dbm CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   CREATE USER 'muser'@'localhost' IDENTIFIED BY 'TU_CLAVE';
   GRANT ALL PRIVILEGES ON dbm.* TO 'muser'@'localhost';
   FLUSH PRIVILEGES;
   EXIT;
   ```

---

## 3. Resumen y Comprobaciones de Conexión

### Tabla Resumen de Servicios

| SGBD | IP / Host | Base / Servicio | Puerto | Usuario | Contraseña |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Oracle 23ai Free** | `localhost` / IP VM | `FREEPDB1` | `1521` | `orauser` | La elegida |
| **PostgreSQL** | `localhost` | `dbpg` | `5432` | `pguser` | La elegida |
| **MariaDB** | `localhost` | `dbm` | `3306` | `muser` | La elegida |

### Verificación de Puertos Abiertos (`netcat`)

Comprobamos la visibilidad y apertura de los puertos TCP necesarios mediante `nc`:

```bash
nc -zv localhost 1521
nc -zv localhost 5432
nc -zv localhost 3306
```

### Pruebas de Autenticación con Clientes Nativos

Comprobación de la conexión local para cada uno de los usuarios creados:

1. **Prueba de conexión en Oracle (SQL\*Plus):**

   ```bash
   sudo podman exec -it cont-oracle sqlplus orauser@//localhost:1521/FREEPDB1
   ```

2. **Prueba de conexión en PostgreSQL (`psql`):**

   ```bash
   psql -h localhost -U pguser -d dbpg -W
   ```

3. **Prueba de conexión en MariaDB (`mariadb`):**

   ```bash
   mariadb -u muser -p dbm
   ```

---

## 4. Configuración de Red para Acceso Remoto

### 4.1. PostgreSQL (Acceso Externo)

1. **Modificación de `postgresql.conf`:**
   Editamos el archivo de configuración principal (ejemplo en `/etc/postgresql/XX/main/postgresql.conf`) ajustando la directiva de escucha:

   ```text
   listen_addresses = 'IP_DE_LA_VM'
   ```

2. **Modificación de `pg_hba.conf`:**
   Añadimos la regla de autenticación remota restringida por red en `/etc/postgresql/XX/main/pg_hba.conf`:

   ```text
   host    dbpg            pguser          192.168.1.0/24          scram-sha-256
   ```

3. **Reinicio del servicio y regla de Firewall:**

   ```bash
   sudo systemctl restart postgresql
   sudo ufw allow from 192.168.1.0/24 to any port 5432 proto tcp
   ```

4. **Prueba desde un equipo cliente remoto:**

   ```bash
   psql -h <IP_DE_LA_VM> -U pguser -d dbpg -W
   ```

### 4.2. MariaDB (Acceso Externo)

1. **Modificación de la dirección de escucha (`bind-address`):**
   Editamos `/etc/mysql/mariadb.conf.d/50-server.cnf` o `/etc/mysql/my.cnf`:

   ```ini
   [mysqld]
   bind-address = IP_DE_LA_VM
   ```

2. **Creación del usuario remoto en MariaDB:**

   ```bash
   sudo mariadb
   ```

   ```sql
   CREATE USER 'muser'@'IP_DEL_CLIENTE' IDENTIFIED BY 'TU_CLAVE';
   GRANT ALL PRIVILEGES ON dbm.* TO 'muser'@'IP_DEL_CLIENTE';
   FLUSH PRIVILEGES;
   EXIT;
   ```

3. **Reinicio del servicio y regla de Firewall:**

   ```bash
   sudo systemctl restart mariadb
   sudo ufw allow from IP_DEL_CLIENTE to any port 3306 proto tcp
   ```

4. **Prueba desde un equipo cliente remoto:**

   ```bash
   mariadb -h IP_DE_LA_VM -u muser -p dbm
   ```

---

## 5. Incidencias Encontradas y Soluciones

* **Primera inicialización de Oracle 23ai:** Durante el primer despliegue, el contenedor tarda un par de minutos en estar disponible. Se verificó mediante `podman logs -f cont-oracle` hasta confirmar que los servicios estaban listos antes de ejecutar `sqlplus`.
* **Restricción de interfaces por defecto en PostgreSQL/MariaDB:** Inicialmente los motores responden solo en `127.0.0.1`. Se solucionó adaptando las directivas `listen_addresses` y `bind-address` junto con la apertura de puertos en UFW.

*(Si no se presentaron problemas en tu caso, puedes indicar: "No se registraron incidencias adicionales durante el despliegue y las pruebas de conexión").*

---

## 6. Evidencias y Fuentes Consultadas

### Evidencias de Funcionamiento

Las capturas de pantalla de la ejecución y verificación se encuentran almacenadas dentro de la carpeta `evidencias/`:

* `evidencias/01-podman-oracle.png`: Muestra el contenedor de Oracle en estado saludable y ejecución de comandos.
* `evidencias/02-conexion-postgresql.png`: Muestra la conexión exitosa con el usuario `pguser`.
* `evidencias/03-conexion-mariadb.png`: Muestra la conexión exitosa con el usuario `muser`.

### Fuentes Consultadas

* [Documentación oficial de Oracle Database Free](https://www.oracle.com/es/database/free/get-started/)
* Documentación oficial de PostgreSQL (`postgresql.conf` & `pg_hba.conf`)
* Knowledge Base oficial de MariaDB sobre configuración de red y gestión de usuarios.
