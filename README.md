# AA1 - Odoo 19 en AWS con arquitectura de 3 capas en alta disponibilidad

Módulo: Diseño, implementación y gestión de infraestructuras cloud y DevOps (2º DAM)
Alumno: Sandro López Díaz
Entrega: 26 de enero de 2027

En este repositorio no hay contraseñas, claves ni endpoints reales. Van como `<CAMBIAR>`.

## Fase 1. Análisis, diseño y documentación
### 1.1 Arquitectura de 3 capas
#### Monolito

Un monolito es una aplicación que está entera en un solo servidor: la aplicación (Odoo), la base de datos y los ficheros están juntos en la misma máquina. Es lo que tiene ahora la empresa de Zaragoza. Es como una ciudad con una sola cafetería: el cliente pide, el trabajador prepara el café y todo pasa dentro del mismo local.

**Ventajas:** es sencillo, barato y rápido de montar.

**Inconvenientes:**
- Si el servidor se cae, se cae todo (SPOF, punto único de fallo). Si se va la luz en la única cafetería, no se sirve ningún café y la empresa deja de facturar.
- Solo se puede escalar en vertical, comprando una máquina más grande. Así aguanta más carga, pero sigue siendo un solo servidor y no mejora la disponibilidad.
- Para actualizarlo o reiniciarlo hay que parar el servicio.

**Para la empresa de Zaragoza** esto es un problema, porque necesita seguir funcionando y facturando aunque algo falle. Por eso se migra a AWS con copias iguales de la aplicación repartidas en varias zonas de disponibilidad (centros de datos independientes): si una falla, las otras siguen dando servicio (redundancia).

#### Arquitectura de 2 capas

Aquí se separa la aplicación de la base de datos. Odoo corre en varios servidores EC2 repartidos en dos zonas de disponibilidad, detrás de un balanceador (ALB). La base de datos está en RDS Multi-AZ, con una principal y una standby en la otra zona. Los ficheros de Odoo van en un EFS compartido y un Auto Scaling Group añade o quita servidores según la carga. Es como tener dos cafeterías con la misma carta y una caja fuerte con copia en otro local: si una se cierra, la otra sigue sirviendo.

Frente al monolito aguanta mucho mejor los fallos, porque ya no depende de un solo servidor ni de una sola zona, y además puede escalar añadiendo servidores. Su punto débil es la seguridad: los servidores Odoo siguen en subredes públicas con IP pública. Si alguien encuentra un fallo en un Odoo mal securizado, puede entrar en él y saltar a los demás servidores y a la base de datos.

#### Arquitectura de 3 capas

Se separa en tres capas y las dos últimas pasan a red privada. Los usuarios, y también los atacantes, solo pueden entrar por el balanceador. Es como una cafetería con un recepcionista en la puerta (el ALB): los clientes solo hablan con él, y los camareros y la caja fuerte están dentro, sin puerta a la calle.

| Capa | Qué contiene | Quién puede llegar |
|---|---|---|
| Presentación (pública) | ALB, NAT Gateway y bastion, en las subredes públicas | Los usuarios, solo por el ALB (puerto 80). El bastion, solo desde mi IP. |
| Aplicación (privada) | Servidores EC2 con Docker y Odoo (Auto Scaling Group) y EFS | Solo el ALB por el puerto 80 y el bastion por SSH. |
| Datos (privada) | RDS PostgreSQL Multi-AZ | Solo los servidores Odoo, por el puerto 5432. |

#### Comparación

| | Monolito | 2 capas | 3 capas |
|---|---|---|---|
| Dónde está todo | Odoo, base de datos y ficheros en un solo servidor | Odoo en varios servidores en subredes públicas; base de datos aparte, en RDS Multi-AZ | Odoo y base de datos en subredes privadas; en las públicas solo el ALB, el NAT y el bastion |
| Si falla un servidor o una zona | Se cae todo | Sigue funcionando con los servidores y la base de datos de la otra zona | Igual que en 2 capas |
| Escalado | Solo vertical (una máquina más grande) | Horizontal, con Auto Scaling Group | Horizontal, con Auto Scaling Group |
| Seguridad | Todo en una sola máquina, normalmente pública | Los servidores Odoo están expuestos: un fallo en uno puede llegar a los demás y a la base de datos | La única entrada es el ALB; Odoo y la base de datos no son accesibles desde Internet |
| Coste y complejidad | El más barato y el más sencillo | Más caro: varios servidores, RDS Multi-AZ y ALB | El más caro y complejo: añade NAT y bastion (el NAT y RDS cobran por hora) |

### 1.2 Elementos del esquema
Los elementos del esquema, agrupados por lo que hacen.

#### Red y acceso

| Elemento | Qué es y qué hace en esta arquitectura |
|---|---|
| VPC (`odoo-vpc`, 10.100.0.0/16) | Es el local entero con sus paredes: nuestra red privada dentro de AWS, aislada del resto, y todo lo demás se monta dentro. Tiene activados DNS hostnames y DNS resolution, que hacen falta para montar EFS por su nombre DNS. |
| Zonas de disponibilidad (AZ A y AZ B) | Centros de datos independientes de la misma región. Repartir cada capa entre las dos es lo que da la alta disponibilidad. |
| Subredes | Trozos de la VPC. Hay 6, tres en cada AZ: públicas (tienen ruta al IGW), de aplicación (privadas, salen a Internet por el NAT) y de datos (privadas y sin salida a Internet). |
| Internet Gateway (IGW) | Es la puerta de la calle del local: por ahí entra y sale todo lo que es público. Conecta la VPC con Internet y solo la usan las subredes públicas. |
| NAT Gateway | Deja que las instancias de las subredes privadas salgan a Internet (instalar paquetes, descargar la imagen de Docker) sin que nadie de fuera pueda iniciar una conexión hacia ellas. Va en la subred pública de la AZ A, con una IP elástica, porque necesita salir por el IGW. Solo hay uno. |
| Bastion host | Una EC2 pequeña (t3.micro) en la subred pública A. Es el único sitio por el que se entra por SSH a las instancias privadas (con `ssh -J`). Su grupo de seguridad solo deja entrar desde mi IP. |
| Grupos de seguridad (SG) | Son el portero de cada puerta, con su lista de quién puede pasar y por dónde. Son cortafuegos con estado que se asocian a cada recurso y se encadenan (el origen de cada regla es otro SG), así cada capa solo habla con la siguiente. Ver 1.5. |

#### Capa de aplicación

| Elemento | Qué es y qué hace en esta arquitectura |
|---|---|
| Application Load Balancer (ALB) | Balanceador de capa 7 (HTTP) expuesto a Internet en las dos subredes públicas. Recibe las peticiones de los usuarios por el puerto 80, las reparte entre los servidores Odoo y solo envía tráfico a los que están sanos. Es la única entrada pública a la aplicación. |
| Target Group | El grupo de destinos del ALB, es decir, la lista de instancias Odoo. Comprueba su salud con un health check HTTP a `/web/login` (códigos 200-399). Si una falla, el ALB deja de enviarle tráfico. |
| Launch Template | Es la ficha para formar a un camarero nuevo: dice cómo es cada instancia Odoo que se crea (Ubuntu Server 24.04, t3.small o t3.medium, SG-EC2, sin IP pública y el script de arranque, user-data, 02). |
| Auto Scaling Group (ASG) | Es el encargado de turnos: mantiene entre 2 y 4 camareros (instancias Odoo, deseada 2) repartidos entre las dos subredes de aplicación. Si uno falla pone otro y si hay mucha gente suma más (política de seguimiento de objetivo, por ejemplo CPU media al 50 %). Usa el health check del ELB y un periodo de gracia de 600 s para dar tiempo a que cada instancia arranque. |
| EC2 con Docker | Cada instancia es un Ubuntu con Docker que ejecuta la imagen oficial `odoo:19.0`. Odoo es el ERP que usan los usuarios. La instalación la hace el user-data al arrancar. |

#### Datos

| Elemento | Qué es y qué hace en esta arquitectura |
|---|---|
| Amazon EFS | Sistema de ficheros compartido por NFS. Se monta en todas las instancias como `/var/lib/odoo` del contenedor, con un punto de montaje en cada subred de aplicación. Ahí se guardan los adjuntos (filestore) y las sesiones, así que son los mismos vengan de la instancia que vengan. Su SG solo deja entrar el puerto 2049 desde SG-EC2. |
| Amazon RDS for PostgreSQL Multi-AZ | La base de datos de Odoo, gestionada por AWS. Tiene una instancia principal en la AZ A y una standby en la AZ B con réplica síncrona; si falla la principal, AWS promociona la standby sola (failover). Está en las subredes de datos (DB subnet group), sin acceso público, y solo acepta conexiones de SG-EC2. |

### 1.3 Diagrama propio
### 1.4 Plan de direccionamiento y tablas de rutas

Plan de direccionamiento de la VPC `odoo-vpc` (10.100.0.0/16), repartida en dos zonas de disponibilidad (AZ A y AZ B):

| Subred | CIDR | AZ | Tipo | Tabla de rutas | Recursos |
|---|---|---|---|---|---|
| odoo-public-a | 10.100.1.0/24 | A | Pública | odoo-rt-public | ALB, NAT Gateway, bastion |
| odoo-public-b | 10.100.2.0/24 | B | Pública | odoo-rt-public | ALB |
| odoo-app-a | 10.100.3.0/24 | A | Privada (aplicación) | odoo-rt-app | EC2 Odoo, EFS |
| odoo-app-b | 10.100.4.0/24 | B | Privada (aplicación) | odoo-rt-app | EC2 Odoo, EFS |
| odoo-data-a | 10.100.5.0/24 | A | Privada (datos) | odoo-rt-data | RDS principal |
| odoo-data-b | 10.100.6.0/24 | B | Privada (datos) | odoo-rt-data | RDS standby |

Tablas de rutas:

| Tabla | Rutas | Subredes asociadas |
|---|---|---|
| odoo-rt-public | 10.100.0.0/16 → local; 0.0.0.0/0 → Internet Gateway | odoo-public-a, odoo-public-b |
| odoo-rt-app | 10.100.0.0/16 → local; 0.0.0.0/0 → NAT Gateway | odoo-app-a, odoo-app-b |
| odoo-rt-data | 10.100.0.0/16 → local (sin salida a Internet) | odoo-data-a, odoo-data-b |

### 1.5 Grupos de seguridad

Tabla de los 5 grupos de seguridad, encadenados entre sí: cada capa solo admite tráfico de la anterior. El origen de las reglas es otro grupo de seguridad y no una IP, porque así las instancias nuevas del ASG ya tienen permiso sin tocar nada.

| Grupo | Se asocia a | Entrada | Origen | Por qué |
|---|---|---|---|---|
| SG-ALB | ALB | HTTP 80 | 0.0.0.0/0 | Es la única puerta pública: los usuarios llegan desde cualquier IP. |
| SG-Bastion | Bastion | SSH 22 | Mi IP | Solo yo administro. Si estuviera abierto a todo Internet, cualquiera podría intentar entrar. |
| SG-EC2 (Odoo) | Instancias Odoo | HTTP 80 | SG-ALB | Odoo solo atiende al balanceador, nadie puede saltarse el ALB. |
| SG-EC2 (Odoo) | Instancias Odoo | SSH 22 | SG-Bastion | A las instancias privadas solo se entra saltando por el bastion. |
| SG-EFS | Puntos de montaje de EFS | NFS 2049 | SG-EC2 | Solo los servidores Odoo pueden montar los ficheros compartidos. |
| SG-Postgres | RDS | 5432 | SG-EC2 | Solo los servidores Odoo hablan con la base de datos. |

### 1.6 Por qué Odoo necesita EFS y RDS

El Auto Scaling Group puede eliminar una instancia en cualquier momento (si baja la carga o si falla el health check) y crear otra nueva. Cuando una instancia desaparece, se va con ella su disco y todo lo que había dentro del contenedor. Por eso las instancias de Odoo no pueden guardar nada importante: tienen que ser desechables, como los camareros de una cafetería, que entran y salen del turno mientras el negocio sigue funcionando.

| Qué se guarda | Dónde va | Qué pasaría si estuviera en la instancia |
|---|---|---|
| Los datos del negocio (clientes, facturas, pedidos) | RDS PostgreSQL Multi-AZ | Se perderían al borrar la instancia y cada servidor tendría su propia base de datos, distinta de las demás. Con RDS hay una única base de datos, con una copia en la otra zona. Es la caja fuerte de la cafetería, con copia en otro local. |
| Adjuntos (filestore), logotipos, plantillas y sesiones | EFS, montado en `/var/lib/odoo` del contenedor | Cada instancia tendría sus propios ficheros: el ALB te llevaría a una u otra y verías cosas distintas, y las sesiones se perderían al cambiar de instancia. Con EFS todas ven lo mismo. Es el almacén compartido por todos los camareros. |

Así, una instancia nueva solo tiene que conectarse a la misma base de datos y montar el mismo EFS para quedar lista, sin perder nada. Da igual qué camarero esté de turno.

Por qué RDS y no PostgreSQL en una EC2: montar la base de datos en una EC2 sería comprar la caja fuerte y encargarme yo de las copias, la cerradura y los fallos. Con RDS contrato a una empresa que lleva la caja fuerte por mí (copias, cifrado, parches y cambio a la copia si falla): cuesta más, pero no me preocupo de nada de eso.

### 1.7 Estimación de costes

## Fase 2. Capa de red y seguridad
(pendiente)

## Fase 3. Capa de datos y primera instancia de Odoo
(pendiente)

## Fase 4. Capa de presentación, Auto Scaling y pruebas de alta disponibilidad
(pendiente)

## Ampliaciones voluntarias
(si hago alguna)
