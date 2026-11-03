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
### 1.5 Grupos de seguridad
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
