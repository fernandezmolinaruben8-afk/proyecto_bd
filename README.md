# proyecto_bd
Rubén Fernández Molina

Nicolas Ruiz Mengual

Oscar Castaño Cárdenas

Vamos a realizar un proyecto  sobre una funeraria con los siguientes apartados:

1-.Definicion del negocio

2-.Modelo E-R

3-.Modelo Relacional

4-.Script con definicion de BBDD

5-.Vistas

6-.Procedimientos

7-.Disparadores

8-.Ejercicios de Transacciones

# PROYECTO BBDD — Funeraria Trujillo

> Proyecto realizado por **Nicolas Ruiz Mengual**, **Oscar Castaño Cárdenas** y **Rubén Fernández Molina**.

---

## Descripción

Funeraria Trujillo es una empresa de servicios exequiales cuya gestión integral se centra en la formalización de contratos personalizados para la atención de fallecidos y el soporte a sus familias. El modelo de negocio se estructura en cuatro pilares fundamentales:

1. Gestión de Clientes y Contratos
2. Servicios y Personal
3. Control de Ubicaciones
4. Trazabilidad de Información

---

## Índice

1. [Modelo E.R.](#1-modelo-er)
2. [Modelo Relacional](#2-modelo-relacional)
3. [Script de Definición BBDD](#3-script-de-definición-bbdd)
4. [Vistas](#4-vistas)
5. [Procedimientos](#5-procedimientos)
6. [Disparadores](#6-disparadores)
7. [Transacciones](#7-transacciones)

---

## 1. Modelo E.R.

### Argumento

- La empresa organiza funerales, por lo que necesita registrar **nombre, apellidos, DNI, teléfono, dirección y email** del cliente que contrata sus servicios.
- Un cliente puede hacer **varios contratos** de defunción.
- De cada contrato se almacena su **estado, total y fecha**.
- Un contrato incluye **un solo difunto**, del que se guarda nombre, apellidos, DNI, fecha de nacimiento, fecha de defunción y causa de muerte.
- Cada difunto tiene **una sola ubicación**, de la que se conoce el sector, la fila, el número y la disponibilidad.
- Un contrato puede contener **muchos servicios**, de los que se registra nombre, descripción y precio base.
- **Muchos empleados** proveen **muchos servicios**; del personal se almacena nombre, apellidos, DNI y cargo.

---

## 2. Modelo Relacional

```
Cliente:    (id_cliente, nombre, apellido1, apellido2, dni, teléfono, dirección, email)
Difunto:    (id_difunto, nombre, apellido1, apellido2, dni, fecha_nacimiento, fecha_defuncion, causa_muerte, id_ubicacion FK)
Ubicación:  (id_ubicacion, sector, fila, número, disponibilidad)
Servicio:   (id_servicio, nombre, descripcion, precio_base)
Personal:   (id_empleado, nombre, apellido1, apellido2, dni, cargo)
Contrato:   (id_contrato, id_cliente, id_difunto, estado, total, fecha_contrato)
CONTIENE:   (id_contrato, id_servicio)
PROVEE:     (id_servicio, id_empleado)
```

### Formas Normales

**1FN** — En las tablas `DIFUNTO`, `CLIENTE` y `PERSONAL`, el campo `apellidos` se divide en `apellido1` y `apellido2` para facilitar búsquedas.

**2FN** —  Está en 2FN ya que está en 1FN y todos los atributos no clave tienen dependencia funcional completa de la clave primaria.

**3FN** — El campo `total` es un valor calculado (suma de precios de servicios). Técnicamente, para una 3FN estricta, no debería almacenarse directamente en la tabla, ya que depende de los servicios vinculados.

---

## 3. Script de Definición BBDD

```sql
CREATE SCHEMA IF NOT EXISTS funeraria;
USE funeraria;

CREATE TABLE IF NOT EXISTS cliente (
    id_cliente  INT AUTO_INCREMENT PRIMARY KEY,
    nombre      VARCHAR(50)  NOT NULL,
    apellido1   VARCHAR(35)  NOT NULL,
    apellido2   VARCHAR(35),
    dni         VARCHAR(9)   UNIQUE NOT NULL,
    telefono    VARCHAR(20),
    direccion   VARCHAR(100),
    email       VARCHAR(80)
);

CREATE TABLE IF NOT EXISTS ubicacion (
    id_ubicacion  INT AUTO_INCREMENT PRIMARY KEY,
    sector        INT,
    fila          INT,
    numero        INT,
    disponibilidad ENUM('Disponible', 'No disponible') DEFAULT 'Disponible'
);

CREATE TABLE IF NOT EXISTS difunto (
    id_difunto       INT AUTO_INCREMENT PRIMARY KEY,
    nombre           VARCHAR(50)  NOT NULL,
    apellido1        VARCHAR(35)  NOT NULL,
    apellido2        VARCHAR(35),
    dni              VARCHAR(9)   UNIQUE NOT NULL,
    fecha_nacimiento DATE,
    fecha_defuncion  DATE,
    causa_muerte     VARCHAR(255),
    id_ubicacion     INT UNIQUE,
    CONSTRAINT fk_difunto_ubicacion FOREIGN KEY (id_ubicacion)
        REFERENCES ubicacion(id_ubicacion)
);

CREATE TABLE IF NOT EXISTS contrato (
    id_contrato    INT AUTO_INCREMENT PRIMARY KEY,
    id_cliente     INT NOT NULL,
    id_difunto     INT UNIQUE NOT NULL,
    estado         VARCHAR(50),
    fecha_contrato DATE,
    CONSTRAINT fk_contrato_cliente FOREIGN KEY (id_cliente)
        REFERENCES cliente(id_cliente),
    CONSTRAINT fk_contrato_difunto FOREIGN KEY (id_difunto)
        REFERENCES difunto(id_difunto)
);

CREATE TABLE IF NOT EXISTS personal (
    id_empleado INT AUTO_INCREMENT PRIMARY KEY,
    nombre      VARCHAR(50) NOT NULL,
    apellido1   VARCHAR(35) NOT NULL,
    apellido2   VARCHAR(35),
    dni         VARCHAR(9)  UNIQUE NOT NULL,
    cargo       VARCHAR(50)
);

CREATE TABLE IF NOT EXISTS servicio (
    id_servicio  INT AUTO_INCREMENT PRIMARY KEY,
    nombre       VARCHAR(100) NOT NULL,
    descripcion  VARCHAR(125),
    precio_base  DECIMAL(10,2) NOT NULL
);

CREATE TABLE IF NOT EXISTS contiene (
    id_contrato              INT NOT NULL,
    id_servicio              INT NOT NULL,
    cantidad                 INT DEFAULT 1,
    precio_unitario_aplicado DECIMAL(10,2),
    PRIMARY KEY (id_contrato, id_servicio),
    CONSTRAINT fk_dc_contrato FOREIGN KEY (id_contrato) REFERENCES contrato(id_contrato),
    CONSTRAINT fk_dc_servicio FOREIGN KEY (id_servicio) REFERENCES servicio(id_servicio)
);

CREATE TABLE IF NOT EXISTS provee (
    id_servicio INT NOT NULL,
    id_empleado INT NOT NULL,
    PRIMARY KEY (id_servicio, id_empleado),
    CONSTRAINT fk_sp_servicio FOREIGN KEY (id_servicio) REFERENCES servicio(id_servicio),
    CONSTRAINT fk_sp_personal FOREIGN KEY (id_empleado) REFERENCES personal(id_empleado)
);
```

---

## 4. Vistas

### 1. Difuntos con su ubicación
Muestra los datos básicos de cada difunto junto con su sector, fila y número asignados.
```sql
CREATE OR REPLACE VIEW vista_ubicacion_difuntos AS
SELECT
    d.nombre, d.apellido1, d.dni,
    u.sector, u.fila, u.numero
FROM difunto AS d
LEFT JOIN ubicacion AS u ON d.id_ubicacion = u.id_ubicacion;
```

### 2. Facturación por contrato
Calcula el importe acumulado total de cada contrato sumando el precio por la cantidad de todos los servicios asociados.
```sql
CREATE OR REPLACE VIEW vista_total_contratos AS
SELECT
    co.id_contrato,
    cl.nombre AS cliente,
    SUM(cn.cantidad * cn.precio_unitario_aplicado) AS total_facturado
FROM contrato AS co
JOIN cliente  AS cl ON co.id_cliente  = cl.id_cliente
JOIN contiene AS cn ON co.id_contrato = cn.id_contrato
GROUP BY co.id_contrato;
```

### 3. Personal y servicios
Lista qué trabajadores de la plantilla están capacitados para ofrecer cada uno de los servicios específicos de la funeraria.
```sql
CREATE OR REPLACE VIEW vista_empleados_servicios AS
SELECT
    p.nombre AS empleado, p.cargo,
    s.nombre AS servicio
FROM personal p
JOIN provee   AS pr ON p.id_empleado  = pr.id_empleado
JOIN servicio AS s  ON pr.id_servicio = s.id_servicio;
```

### 4. Servicios más vendidos
Filtra y expone los nombres de los servicios de alta demanda que se han vendido en más de 5 ocasiones.

```sql
CREATE OR REPLACE VIEW vista_servicios_populares AS
SELECT
    s.nombre,
    COUNT(cn.id_contrato) AS veces_contratado,
    SUM(cn.cantidad)      AS cantidad_total_vendida
FROM servicio s
JOIN contiene cn ON s.id_servicio = cn.id_servicio
GROUP BY s.id_servicio
HAVING veces_contratado > 5;
```

### 5. Carga de trabajo por empleado
Contabiliza la cantidad total de especialidades de servicio que tiene asignadas cada empleado para medir su volumen de funciones.
```sql
CREATE OR REPLACE VIEW vista_carga_trabajo_personal AS
SELECT
    p.nombre, p.apellido1, p.cargo,
    COUNT(pr.id_servicio) AS total_servicios_especialidad
FROM personal AS p
LEFT JOIN provee pr ON p.id_empleado = pr.id_empleado
GROUP BY p.id_empleado;
```

---

## 5. Procedimientos

### 1. Registrar un nuevo difunto y asignar ubicación

Inserta un difunto y marca su ubicación como "No disponible".

```sql
DELIMITER //
CREATE PROCEDURE sp_registrar_difunto(
    IN p_nombre       VARCHAR(50),
    IN p_apellido1    VARCHAR(35),
    IN p_apellido2    VARCHAR(35),
    IN p_dni          VARCHAR(9),
    IN p_fecha_nac    DATE,
    IN p_fecha_def    DATE,
    IN p_causa        VARCHAR(255),
    IN p_id_ubicacion INT
)
BEGIN
    INSERT INTO difunto (nombre, apellido1, apellido2, dni,
                         fecha_nacimiento, fecha_defuncion, causa_muerte, id_ubicacion)
    VALUES (p_nombre, p_apellido1, p_apellido2, p_dni,
            p_fecha_nac, p_fecha_def, p_causa, p_id_ubicacion);

    UPDATE ubicacion
    SET disponibilidad = 'No disponible'
    WHERE id_ubicacion = p_id_ubicacion;
END //
DELIMITER ;
```

### 2. Añadir servicio a un contrato

Busca automáticamente el `precio_base` del servicio y lo guarda como `precio_unitario_aplicado`.

```sql
DELIMITER //
CREATE PROCEDURE sp_agregar_servicio_contrato(
    IN p_id_contrato INT,
    IN p_id_servicio INT,
    IN p_cantidad    INT
)
BEGIN
    DECLARE v_precio_actual DECIMAL(10,2);

    SELECT precio_base INTO v_precio_actual
    FROM servicio WHERE id_servicio = p_id_servicio;

    INSERT INTO contiene (id_contrato, id_servicio, cantidad, precio_unitario_aplicado)
    VALUES (p_id_contrato, p_id_servicio, p_cantidad, v_precio_actual);
END //
DELIMITER ;
```

### 3. Eliminar un contrato y sus dependencias

Elimina primero los registros hijos en `contiene` y luego el contrato.

```sql
DELIMITER //
CREATE PROCEDURE sp_eliminar_contrato(IN p_id_contrato INT)
BEGIN
    DELETE FROM contiene WHERE id_contrato = p_id_contrato;
    DELETE FROM contrato WHERE id_contrato = p_id_contrato;
END //
DELIMITER ;
```

### 4. Traslado de Difunto
Cambia la ubicación de un difunto, liberando de forma automática su espacio anterior y ocupando el nuevo.
```sql
DELIMITER //
CREATE PROCEDURE sp_trasladar_difunto(
    IN p_id_difunto INT,
    IN p_nueva_ubicacion INT
)
BEGIN
    DECLARE v_ubicacion_antigua INT;

    -- 1. Averiguar qué ubicación tenía antes el difunto
    
    SELECT id_ubicacion INTO v_ubicacion_antigua 
    FROM difunto 
    WHERE id_difunto = p_id_difunto;

    -- 2. Liberar la ubicación antigua
    
    UPDATE ubicacion SET disponibilidad = 'Disponible' WHERE id_ubicacion = v_ubicacion_antigua;

    -- 3. Ocupar la nueva ubicación
    UPDATE ubicacion SET disponibilidad = 'No disponible' WHERE id_ubicacion = p_nueva_ubicacion;

    -- 4. Asignar la nueva ubicación al difunto
    UPDATE difunto SET id_ubicacion = p_nueva_ubicacion WHERE id_difunto = p_id_difunto;
END //
DELIMITER ;
```
### 5. Actualizar precio de un servicio
Modifica el precio base de un servicio específico de la funeraria a partir de su identificador único.
```sql
DELIMITER //
CREATE PROCEDURE sp_actualizar_precio_servicio(
    IN p_id_servicio  INT,
    IN p_nuevo_precio DECIMAL(10,2)
)
BEGIN
    UPDATE servicio
    SET precio_base = p_nuevo_precio
    WHERE id_servicio = p_id_servicio;
END //
DELIMITER ;
```

### 6. Generar resumen de contrato (factura simple)

Devuelve datos del cliente, el difunto y el total a pagar.

```sql
DELIMITER //
CREATE PROCEDURE sp_resumen_facturacion_contrato(IN p_id_contrato INT)
BEGIN
    SELECT
        co.id_contrato,
        cl.nombre    AS cliente_nombre,
        cl.apellido1 AS cliente_apellido,
        d.nombre     AS difunto_nombre,
        SUM(cn.cantidad * cn.precio_unitario_aplicado) AS total_a_pagar
    FROM contrato co
    JOIN cliente  cl ON co.id_cliente  = cl.id_cliente
    JOIN difunto  d  ON co.id_difunto  = d.id_difunto
    JOIN contiene cn ON co.id_contrato = cn.id_contrato
    WHERE co.id_contrato = p_id_contrato
    GROUP BY co.id_contrato;
END //
DELIMITER ;
```

---

## 6. Disparadores

### 1. Liberar ubicación al eliminar un difunto
Libera y marca como disponible de forma automática un espacio cuando el registro de su difunto es eliminado.
```sql
DELIMITER //
CREATE TRIGGER tr_liberar_ubicacion_delete
AFTER DELETE ON difunto FOR EACH ROW
BEGIN
    UPDATE ubicacion SET disponibilidad = 'Disponible'
    WHERE id_ubicacion = OLD.id_ubicacion;
END //
DELIMITER ;
```

### 2. Validar que no se asigne una ubicación ocupada
Bloquea la inserción de un difunto si el espacio asignado ya se encuentra ocupado por otro.
```sql
DELIMITER //
CREATE TRIGGER tr_validar_ubicacion_disponible
BEFORE INSERT ON difunto FOR EACH ROW
BEGIN
    DECLARE estado_ubi VARCHAR(20);
    SELECT disponibilidad INTO estado_ubi FROM ubicacion
    WHERE id_ubicacion = NEW.id_ubicacion;
    IF estado_ubi = 'No disponible' THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Error: La ubicación ya está ocupada.';
    END IF;
END //
DELIMITER ;
```

### 3. Evitar precios negativos en servicios
Cancela cualquier intento de asignar un coste negativo a un servicio restaurando de forma automática su importe anterior.
```sql
DELIMITER //
CREATE TRIGGER tr_validar_precio_positivo
BEFORE UPDATE ON servicio FOR EACH ROW
BEGIN
    IF NEW.precio_base < 0 THEN
        SET NEW.precio_base = OLD.precio_base;
    END IF;
END //
DELIMITER ;
```

### 4. Formatear DNI a mayúsculas automáticamente
Convierte y guarda de forma automática en letras mayúsculas el documento de identidad de cualquier cliente nuevo.
```sql
DELIMITER //
CREATE TRIGGER tr_cliente_dni_upper
BEFORE INSERT ON cliente FOR EACH ROW
BEGIN
    SET NEW.dni = UPPER(NEW.dni);
END //
DELIMITER ;
```

### 5. Impedir la eliminación de clientes con contratos activos
Cancela la eliminación de un cliente si este cuenta con acuerdos de servicios funerarios vigentes en el sistema.
```sql
DELIMITER //
CREATE TRIGGER tr_prevenir_borrado_cliente
BEFORE DELETE ON cliente FOR EACH ROW
BEGIN
    DECLARE total_contratos INT;
    SELECT COUNT(*) INTO total_contratos FROM contrato
    WHERE id_cliente = OLD.id_cliente;
    IF total_contratos > 0 THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'No se puede eliminar un cliente con contratos vinculados.';
    END IF;
END //
DELIMITER ;
```

### 6. Asignación automática de fecha de contrato
Registra de forma automática el día actual en los nuevos acuerdos comerciales si no se introduce ninguno de forma manual.
```sql
DELIMITER //
CREATE TRIGGER tr_fecha_contrato_default
BEFORE INSERT ON contrato FOR EACH ROW
BEGIN
    IF NEW.fecha_contrato IS NULL THEN
        SET NEW.fecha_contrato = CURDATE();
    END IF;
END //
DELIMITER ;
```

### 7. Control de cantidad mínima en servicios del contrato
Fuerza que el volumen mínimo contratado de un servicio sea de al menos una unidad si se introduce un valor nulo o negativo.
```sql
DELIMITER //
CREATE TRIGGER tr_validar_cantidad_servicio
BEFORE INSERT ON contiene FOR EACH ROW
BEGIN
    IF NEW.cantidad <= 0 THEN
        SET NEW.cantidad = 1;
    END IF;
END //
DELIMITER ;
```

### 8. Log de cambios de email de clientes
Registra en un historial de movimientos el correo antiguo y el nuevo cada vez que un cliente modifica sus datos de contacto.
```sql
DELIMITER //
CREATE TRIGGER tr_log_email_cliente
BEFORE UPDATE ON cliente FOR EACH ROW
BEGIN
    IF OLD.email <> NEW.email THEN
        INSERT INTO log_movimientos (tabla_afectada, accion, fecha, usuario)
        VALUES ('cliente', CONCAT('Email cambiado de ', OLD.email, ' a ', NEW.email), NOW(), USER());
    END IF;
END //
DELIMITER ;
```

### 9. Prevención de ubicaciones sin sector o número
Bloquea la creación de nuevos espacios si no se especifican obligatoriamente los datos físicos de sector y número.
```sql
DELIMITER //
CREATE TRIGGER tr_validar_datos_ubicacion
BEFORE INSERT ON ubicacion FOR EACH ROW
BEGIN
    IF NEW.sector IS NULL OR NEW.numero IS NULL THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Error: El sector y el número son obligatorios para crear una ubicación.';
    END IF;
END //
DELIMITER ;
```

### 10. Sincronizar disponibilidad al cambiar ubicación de difunto
Actualiza de forma automática los estados de ocupación de las zonas físicas cuando un difunto es trasladado de un lugar a otro.
```sql
DELIMITER //
CREATE TRIGGER tr_cambio_ubicacion_difunto
AFTER UPDATE ON difunto FOR EACH ROW
BEGIN
    IF OLD.id_ubicacion <> NEW.id_ubicacion THEN
        UPDATE ubicacion SET disponibilidad = 'Disponible'
        WHERE id_ubicacion = OLD.id_ubicacion;
        UPDATE ubicacion SET disponibilidad = 'No disponible'
        WHERE id_ubicacion = NEW.id_ubicacion;
    END IF;
END //
DELIMITER ;
```

---

## 7. Transacciones

### 1. Registrar difunto, asignar ubicación y crear contrato



Este procedimiento automatiza esta transacción haciendo registrar al difunto, ocupando la ubicación y generando el contrato en un solo bloque seguro.

---

```sql
DELIMITER //
CREATE PROCEDURE sp_registrar_servicio_completo (
    IN p_id_cliente INT,
    IN p_nombre_difunto VARCHAR(50),
    IN p_apellido1_difunto VARCHAR(35),
    IN p_apellido2_difunto VARCHAR(35),
    IN p_dni_difunto VARCHAR(9),
    IN p_fecha_nac DATE,
    IN p_fecha_def DATE,
    IN p_causa_muerte VARCHAR(255),
    IN p_id_ubicacion INT,
    IN p_estado_contrato VARCHAR(50)
)
BEGIN
    -- Manejo de errores: si ocurre una excepción, deshace todo (ROLLBACK)
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Error en la transacción: No se pudo registrar el servicio completo.';
    END;

    -- Iniciar la transacción de forma segura
    START TRANSACTION;

    -- 1. Insertar el difunto (Tabla: difunto)
    INSERT INTO difunto (nombre, apellido1, apellido2, dni, fecha_nacimiento, fecha_defuncion, causa_muerte, id_ubicacion)
    VALUES (p_nombre_difunto, p_apellido1_difunto, p_apellido2_difunto, p_dni_difunto, p_fecha_nac, p_fecha_def, p_causa_muerte, p_id_ubicacion);

    -- 2. Actualizar el estado de la ubicación a 'No disponible' (Tabla: ubicacion)
    UPDATE ubicacion 
    SET disponibilidad = 'No disponible' 
    WHERE id_ubicacion = p_id_ubicacion;

    -- 3. Crear el contrato asociado utilizando el ID del difunto recién generado (Tabla: contrato)
    INSERT INTO contrato (id_cliente, id_difunto, estado, fecha_contrato)
    VALUES (p_id_cliente, LAST_INSERT_ID(), p_estado_contrato, CURDATE());

    -- Si todo se ejecutó correctamente, guardamos los cambios definitivamente
    COMMIT;
END //
DELIMITER ;

```

---

### 2. Mueve a un difunto de una tumba a otra, asegurando que la ubicación antigua se libere y la nueva se ocupe en el mismo instante.

```sql
DELIMITER //

CREATE PROCEDURE sp_tx_traslado_difunto (
    IN p_id_difunto INT,
    IN p_id_ubicacion_nueva INT
)
BEGIN
    DECLARE v_id_ubicacion_vieja INT;

    -- Manejo de errores
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Error en la transacción: No se pudo realizar el traslado del difunto.';
    END;

    START TRANSACTION;

    -- 1. Obtener la ubicación actual antes de modificarla
    SELECT id_ubicacion INTO v_id_ubicacion_vieja 
    FROM difunto 
    WHERE id_difunto = p_id_difunto;

    -- 2. Liberar la ubicación antigua
    UPDATE ubicacion 
    SET disponibilidad = 'Disponible' 
    WHERE id_ubicacion = v_id_ubicacion_vieja;

    -- 3. Ocupar la nueva ubicación
    UPDATE ubicacion 
    SET disponibilidad = 'No disponible' 
    WHERE id_ubicacion = p_id_ubicacion_nueva;

    -- 4. Actualizar la ficha del difunto con su nueva localización
    UPDATE difunto 
    SET id_ubicacion = p_id_ubicacion_nueva 
    WHERE id_difunto = p_id_difunto;

    COMMIT;
END //

DELIMITER ;
```

### 3. Actualización precio
Este procedimiento actualiza de forma masiva el precio de una categoría o un servicio específico en el catálogo (servicio), pero garantiza que no se rompa la integridad de la base de datos si ocurre un error en la actualización.

```sql
DELIMITER //

CREATE PROCEDURE sp_tx_actualizar_tarifas_seguro (
    IN p_id_servicio INT,
    IN p_nuevo_precio DECIMAL(10,2)
)
BEGIN
    -- Manejo de errores: Si el nuevo precio rompe alguna regla o falla la BD, se cancela todo
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Error en la transacción: No se pudo actualizar la tarifa de forma segura.';
    END;

    -- Iniciar transacción
    START TRANSACTION;

    -- 1. Validar primero que el precio no sea negativo (aunque tengas un trigger, la Tx lo protege)
    IF p_nuevo_precio < 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Error: El precio no puede ser inferior a 0.';
    END IF;

    -- 2. Actualizar el catálogo de servicios (Tabla: servicio) 
    UPDATE servicio 
    SET precio_base = p_nuevo_precio 
    WHERE id_servicio = p_id_servicio; 

    -- Nota: Gracias a la 2FN de tu modelo, los contratos antiguos no cambiarán su total, 
    -- ya que la tabla 'contiene' almacena el 'precio_unitario_aplicado' de forma estática.

    COMMIT;
END //

DELIMITER ;

```

### 4. Eliminar empleado y sus vínculos con servicios
Los empleados están vinculados a los servicios que proveen a través de la tabla provee. Si un miembro del personal se da de baja por enfermedad o vacaciones, necesitas quitarle de forma inmediata todos sus servicios asignados y transferirlos a otro empleado disponible en un solo movimiento atómico para que la funeraria no deje de prestar ese soporte.

```sql
DELIMITER //

CREATE PROCEDURE sp_tx_reemplazar_personal_servicio (
    IN p_id_empleado_saliente INT,
    IN p_id_empleado_entrante INT
)
BEGIN
    -- Manejo de errores
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Error en la transacción: No se pudo realizar la reasignación de personal.';
    END;

    START TRANSACTION;

    -- 1. Duplicar las asignaciones de servicios del empleado saliente para el empleado entrante (Tabla: provee) 
    -- Usamos INSERT IGNORE por si el empleado entrante ya tenía asignado alguno de esos servicios previamente.
    INSERT IGNORE INTO provee (id_servicio, id_empleado)
    SELECT id_servicio, p_id_empleado_entrante
    FROM provee
    WHERE id_empleado = p_id_empleado_saliente;

    -- 2. Eliminar de forma segura las asignaciones del empleado que sale de este turno/puesto
    DELETE FROM provee 
    WHERE id_empleado = p_id_empleado_saliente;

    -- Si ambos pasos se completan, el relevo de funciones queda registrado de forma limpia
    COMMIT;
END //

DELIMITER ;

```

### 5. Cancelar un contrato y liberar la ubicación
Al anular un contrato de manera integral, elimina la relación de servicios, libera el espacio ocupado en el cementerio y borra el contrato original.

```sql
DELIMITER //

CREATE PROCEDURE sp_tx_anular_contrato_total (
    IN p_id_contrato INT
)
BEGIN
    DECLARE v_id_difunto INT;

    -- Manejo de errores
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Error en la transacción: No se pudo anular el contrato de forma segura.';
    END;

    START TRANSACTION;

    -- 1. Obtener el ID del difunto vinculado al contrato
    SELECT id_difunto INTO v_id_difunto 
    FROM contrato 
    WHERE id_contrato = p_id_contrato;

    -- 2. Liberar la ubicación reservada en la tabla correspondiente
    UPDATE ubicacion 
    SET disponibilidad = 'Disponible' 
    WHERE id_ubicacion = (SELECT id_ubicacion FROM difunto WHERE id_difunto = v_id_difunto);

    -- 3. Eliminar primero los servicios asignados en la tabla intermedia (Hijos)
    DELETE FROM contiene 
    WHERE id_contrato = p_id_contrato;

    -- 4. Eliminar el registro del contrato principal (Padre)
    DELETE FROM contrato 
    WHERE id_contrato = p_id_contrato;

    COMMIT;
END //

DELIMITER ;

```
