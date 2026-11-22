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

### 1.2 Elementos del esquema
(VPC, subredes, IGW, NAT Gateway, bastion, ALB, Target Group, Launch Template, ASG, EC2 con Docker, EFS, RDS Multi-AZ, grupos de seguridad)
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
### 1.7 Estimación de costes

## Fase 2. Capa de red y seguridad
(pendiente)

## Fase 3. Capa de datos y primera instancia de Odoo
(pendiente)

## Fase 4. Capa de presentación, Auto Scaling y pruebas de alta disponibilidad
(pendiente)

## Ampliaciones voluntarias
(si hago alguna)
