-- =============================================
-- ERP MVP Grupo 8 - Script de base de datos
-- MariaDB / MySQL compatible
-- =============================================

-- =============================================
-- CREATE DATABASE
-- =============================================
DROP DATABASE IF EXISTS `erp_mvp_grupo8`;
CREATE DATABASE `erp_mvp_grupo8`
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
USE `erp_mvp_grupo8`;

-- Aseguramos el motor por defecto
SET default_storage_engine = INNODB;
SET FOREIGN_KEY_CHECKS = 0;

-- =============================================
-- TABLAS BÁSICAS: clientes, productos, usuarios
-- =============================================

-- Clientes
CREATE TABLE `clientes` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
  `nombre` VARCHAR(150) NOT NULL,
  `cuit` VARCHAR(20) NULL,
  `direccion` VARCHAR(200) NULL,
  `telefono` VARCHAR(50) NULL,
  `email` VARCHAR(120) NULL,
  `activo` TINYINT(1) NOT NULL DEFAULT 1,
  `fecha_creacion` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_clientes_cuit` (`cuit`),
  UNIQUE KEY `uk_clientes_email` (`email`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Productos (artículos)
CREATE TABLE `productos` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
  `nombre` VARCHAR(200) NOT NULL,
  `codigo` VARCHAR(60) NOT NULL,
  `precio` DECIMAL(12,2) NOT NULL DEFAULT 0.00,
  `stock` DECIMAL(12,2) NOT NULL DEFAULT 0.00,
  `activo` TINYINT(1) NOT NULL DEFAULT 1,
  `fecha_creacion` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_productos_codigo` (`codigo`),
  CHECK (`precio` >= 0),
  CHECK (`stock` >= 0)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Usuarios del sistema
CREATE TABLE `usuarios` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
  `nombre` VARCHAR(150) NOT NULL,
  `usuario` VARCHAR(80) NOT NULL,
  `email` VARCHAR(150) NOT NULL,
  `password_hash` VARCHAR(255) NOT NULL,
  `rol` ENUM('vendedor','admin') NOT NULL DEFAULT 'vendedor',
  `activo` TINYINT(1) NOT NULL DEFAULT 1,
  `fecha_creacion` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_usuarios_usuario` (`usuario`),
  UNIQUE KEY `uk_usuarios_email` (`email`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- =============================================
-- VENTAS: cabecera y detalle
-- =============================================

-- Cabecera de ventas
CREATE TABLE `ventas` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
  `cliente_id` INT UNSIGNED NOT NULL,
  `usuario_id` INT UNSIGNED NOT NULL,
  `fecha` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `total` DECIMAL(12,2) NOT NULL DEFAULT 0.00,
  `estado` ENUM('borrador','confirmada','anulada') NOT NULL DEFAULT 'borrador',
  PRIMARY KEY (`id`),
  KEY `idx_ventas_cliente` (`cliente_id`),
  KEY `idx_ventas_usuario` (`usuario_id`),
  CONSTRAINT `fk_ventas_cliente` FOREIGN KEY (`cliente_id`) REFERENCES `clientes` (`id`) ON UPDATE RESTRICT ON DELETE RESTRICT,
  CONSTRAINT `fk_ventas_usuario` FOREIGN KEY (`usuario_id`) REFERENCES `usuarios` (`id`) ON UPDATE RESTRICT ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Detalle de ventas
CREATE TABLE `venta_detalles` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
  `venta_id` INT UNSIGNED NOT NULL,
  `producto_id` INT UNSIGNED NOT NULL,
  `cantidad` DECIMAL(12,2) NOT NULL,
  `precio` DECIMAL(12,2) NOT NULL,
  `subtotal` DECIMAL(12,2) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_detalles_venta` (`venta_id`),
  KEY `idx_detalles_producto` (`producto_id`),
  CONSTRAINT `fk_detalles_venta` FOREIGN KEY (`venta_id`) REFERENCES `ventas` (`id`) ON UPDATE RESTRICT ON DELETE CASCADE,
  CONSTRAINT `fk_detalles_producto` FOREIGN KEY (`producto_id`) REFERENCES `productos` (`id`) ON UPDATE RESTRICT ON DELETE RESTRICT,
  CHECK (`cantidad` > 0),
  CHECK (`precio` >= 0),
  CHECK (`subtotal` >= 0)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- =============================================
-- STOCK: movimientos y ajustes manuales
-- =============================================

-- Movimientos de stock (derivados de ventas o ajustes)
CREATE TABLE `stock_movimientos` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
  `producto_id` INT UNSIGNED NOT NULL,
  `venta_id` INT UNSIGNED NULL,
  `usuario_id` INT UNSIGNED NULL,
  `tipo` ENUM('venta','anulacion','ajuste') NOT NULL,
  `cantidad` DECIMAL(12,2) NOT NULL,
  `fecha` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `idx_mov_producto` (`producto_id`),
  KEY `idx_mov_venta` (`venta_id`),
  KEY `idx_mov_usuario` (`usuario_id`),
  CONSTRAINT `fk_mov_producto` FOREIGN KEY (`producto_id`) REFERENCES `productos` (`id`) ON UPDATE RESTRICT ON DELETE RESTRICT,
  CONSTRAINT `fk_mov_venta` FOREIGN KEY (`venta_id`) REFERENCES `ventas` (`id`) ON UPDATE RESTRICT ON DELETE CASCADE,
  CONSTRAINT `fk_mov_usuario` FOREIGN KEY (`usuario_id`) REFERENCES `usuarios` (`id`) ON UPDATE RESTRICT ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Ajustes manuales de stock (para registrar ajustes con motivo)
CREATE TABLE `stock_ajustes` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
  `producto_id` INT UNSIGNED NOT NULL,
  `usuario_id` INT UNSIGNED NULL,
  `fecha` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `cantidad` DECIMAL(12,2) NOT NULL,
  `motivo` VARCHAR(255) NULL,
  PRIMARY KEY (`id`),
  KEY `idx_ajuste_producto` (`producto_id`),
  KEY `idx_ajuste_usuario` (`usuario_id`),
  CONSTRAINT `fk_ajuste_producto` FOREIGN KEY (`producto_id`) REFERENCES `productos` (`id`) ON UPDATE RESTRICT ON DELETE RESTRICT,
  CONSTRAINT `fk_ajuste_usuario` FOREIGN KEY (`usuario_id`) REFERENCES `usuarios` (`id`) ON UPDATE RESTRICT ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

SET FOREIGN_KEY_CHECKS = 1;

-- =============================================
-- TRIGGERS
-- =============================================
DELIMITER $$

-- Recalcular subtotal antes de insertar detalle
DROP TRIGGER IF EXISTS `trg_venta_detalles_bi_calc_subtotal` $$
CREATE TRIGGER `trg_venta_detalles_bi_calc_subtotal`
BEFORE INSERT ON `venta_detalles`
FOR EACH ROW
BEGIN
  IF NEW.cantidad <= 0 THEN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'La cantidad debe ser mayor a 0';
  END IF;
  IF NEW.precio < 0 THEN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'El precio no puede ser negativo';
  END IF;
  SET NEW.subtotal = ROUND(NEW.cantidad * NEW.precio, 2);
END $$

-- Recalcular subtotal antes de actualizar detalle
DROP TRIGGER IF EXISTS `trg_venta_detalles_bu_calc_subtotal` $$
CREATE TRIGGER `trg_venta_detalles_bu_calc_subtotal`
BEFORE UPDATE ON `venta_detalles`
FOR EACH ROW
BEGIN
  IF NEW.cantidad <= 0 THEN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'La cantidad debe ser mayor a 0';
  END IF;
  IF NEW.precio < 0 THEN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'El precio no puede ser negativo';
  END IF;
  SET NEW.subtotal = ROUND(NEW.cantidad * NEW.precio, 2);
END $$

-- Actualizar total de la venta después de insertar detalle
DROP TRIGGER IF EXISTS `trg_venta_detalles_ai_update_total` $$
CREATE TRIGGER `trg_venta_detalles_ai_update_total`
AFTER INSERT ON `venta_detalles`
FOR EACH ROW
BEGIN
  UPDATE `ventas` v
  SET v.total = (
    SELECT IFNULL(SUM(d.subtotal), 0) FROM `venta_detalles` d WHERE d.venta_id = NEW.venta_id
  )
  WHERE v.id = NEW.venta_id;
END $$

-- Actualizar total de la venta después de actualizar detalle
DROP TRIGGER IF EXISTS `trg_venta_detalles_au_update_total` $$
CREATE TRIGGER `trg_venta_detalles_au_update_total`
AFTER UPDATE ON `venta_detalles`
FOR EACH ROW
BEGIN
  -- Si cambió de venta, actualizamos ambas
  IF (NEW.venta_id <> OLD.venta_id) THEN
    UPDATE `ventas` v
    SET v.total = (
      SELECT IFNULL(SUM(d.subtotal), 0) FROM `venta_detalles` d WHERE d.venta_id = OLD.venta_id
    )
    WHERE v.id = OLD.venta_id;
  END IF;
  UPDATE `ventas` v
  SET v.total = (
    SELECT IFNULL(SUM(d.subtotal), 0) FROM `venta_detalles` d WHERE d.venta_id = NEW.venta_id
  )
  WHERE v.id = NEW.venta_id;
END $$

-- Actualizar total de la venta después de borrar detalle
DROP TRIGGER IF EXISTS `trg_venta_detalles_ad_update_total` $$
CREATE TRIGGER `trg_venta_detalles_ad_update_total`
AFTER DELETE ON `venta_detalles`
FOR EACH ROW
BEGIN
  UPDATE `ventas` v
  SET v.total = (
    SELECT IFNULL(SUM(d.subtotal), 0) FROM `venta_detalles` d WHERE d.venta_id = OLD.venta_id
  )
  WHERE v.id = OLD.venta_id;
END $$

-- Al cambiar el estado de la venta, generar movimientos de stock
DROP TRIGGER IF EXISTS `trg_ventas_au_stock_movs` $$
CREATE TRIGGER `trg_ventas_au_stock_movs`
AFTER UPDATE ON `ventas`
FOR EACH ROW
BEGIN
  -- Confirmación: descuenta stock
  IF (OLD.estado = 'borrador' AND NEW.estado = 'confirmada') THEN
    INSERT INTO `stock_movimientos` (`producto_id`, `venta_id`, `usuario_id`, `tipo`, `cantidad`, `fecha`)
    SELECT d.producto_id, NEW.id, NEW.usuario_id, 'venta', -d.cantidad, CURRENT_TIMESTAMP
    FROM `venta_detalles` d
    WHERE d.venta_id = NEW.id;
  END IF;

  -- Anulación: repone stock
  IF (OLD.estado = 'confirmada' AND NEW.estado = 'anulada') THEN
    INSERT INTO `stock_movimientos` (`producto_id`, `venta_id`, `usuario_id`, `tipo`, `cantidad`, `fecha`)
    SELECT d.producto_id, NEW.id, NEW.usuario_id, 'anulacion', d.cantidad, CURRENT_TIMESTAMP
    FROM `venta_detalles` d
    WHERE d.venta_id = NEW.id;
  END IF;
END $$

-- Aplicar movimiento de stock: validar y actualizar productos.stock
DROP TRIGGER IF EXISTS `trg_stock_movimientos_ai_apply` $$
CREATE TRIGGER `trg_stock_movimientos_ai_apply`
AFTER INSERT ON `stock_movimientos`
FOR EACH ROW
BEGIN
  DECLARE v_stock_actual DECIMAL(12,2);
  SELECT p.stock INTO v_stock_actual FROM `productos` p WHERE p.id = NEW.producto_id FOR UPDATE;
  IF (v_stock_actual + NEW.cantidad) < 0 THEN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Stock no puede quedar negativo';
  END IF;
  UPDATE `productos` p SET p.stock = v_stock_actual + NEW.cantidad WHERE p.id = NEW.producto_id;
END $$

-- Al crear un ajuste manual, generar un movimiento de tipo 'ajuste'
DROP TRIGGER IF EXISTS `trg_stock_ajustes_ai_create_mov` $$
CREATE TRIGGER `trg_stock_ajustes_ai_create_mov`
AFTER INSERT ON `stock_ajustes`
FOR EACH ROW
BEGIN
  INSERT INTO `stock_movimientos` (`producto_id`, `venta_id`, `usuario_id`, `tipo`, `cantidad`, `fecha`)
  VALUES (NEW.producto_id, NULL, NEW.usuario_id, 'ajuste', NEW.cantidad, NEW.fecha);
END $$

DELIMITER ;

-- =============================================
-- ÍNDICES ADICIONALES (si hiciera falta)
-- =============================================
-- Ya se declararon índices para FKs y UNIQs.

-- =============================================
-- CONSULTAS DE EJEMPLO
-- =============================================

-- Productos con poco stock (<= 5)
-- Nota: ajustar el umbral según necesidad
SELECT p.id, p.nombre, p.codigo, p.stock
FROM `productos` p
WHERE p.activo = 1 AND p.stock <= 5
ORDER BY p.stock ASC, p.nombre ASC;

-- Ventas del día (confirmadas) con su total
SELECT v.id,
       v.fecha,
       c.nombre    AS cliente,
       u.usuario   AS vendedor,
       v.total
FROM `ventas` v
JOIN `clientes` c ON c.id = v.cliente_id
JOIN `usuarios` u ON u.id = v.usuario_id
WHERE DATE(v.fecha) = CURRENT_DATE
  AND v.estado = 'confirmada'
ORDER BY v.fecha DESC, v.id DESC;
