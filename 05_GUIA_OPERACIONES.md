# Guía de Operaciones

## Sistema Abuelo Cómodo v2.0

---

## Introducción

Este documento está diseñado para el equipo operativo de Abuelo Cómodo: personal de almacén, asesores de ventas, y administradores que interactúan diariamente con el sistema de inventario. No asume conocimientos técnicos profundos, pero sí familiaridad con las operaciones comerciales de la empresa y el uso básico de AppSheet.

El sistema ha evolucionado desde una arquitectura basada en Google Sheets hacia una plataforma de base de datos moderna. Aunque la interfaz de AppSheet permanece similar, los procesos subyacentes ahora operan en tiempo real, eliminando las esperas de cinco a quince minutos que caracterizaban al sistema anterior.

---

## Flujos de Trabajo Principales

### Procesamiento de Órdenes Shopify

Cuando un cliente completa una compra en la tienda en línea o en los puntos de venta físicos, el sistema procesa la orden automáticamente:

**Secuencia de eventos:**

1. Cliente completa pago en Shopify
2. Shopify envía notificación instantánea al sistema
3. El sistema crea el registro de orden
4. Para cada producto ordenado:
   - Si es producto compuesto, se desglosan los componentes automáticamente
   - Se reserva inventario para el producto y sus componentes
   - Se generan los asientos contables correspondientes
5. La orden aparece en AppSheet lista para preparación

**Tiempo total de procesamiento:** Menos de 2 segundos

**Visualización en AppSheet:** Las órdenes de Shopify aparecen con el prefijo "SHO-" seguido del número de orden. Por ejemplo: SHO-1234.

---

### Flujo de Prospectos Telefónicos

Los prospectos que llegan por llamada telefónica siguen un proceso de captura, seguimiento, y eventual conversión a orden:

**Captura del prospecto:**

1. El asesor recibe la llamada
2. Ingresa los datos del prospecto en la aplicación AppSheet
3. El sistema asigna automáticamente un identificador único
4. El prospecto queda vinculado a la línea telefónica correspondiente

**Conversión a orden:**

1. El asesor confirma los productos seleccionados
2. El sistema valida disponibilidad de inventario
3. Al marcar el prospecto como cliente ("es_cliente = true"), el sistema:
   - Crea automáticamente un registro de cliente
   - Genera la orden con el prefijo del asesor (ejemplo: ANA-TLF-45)
   - Procesa los productos y sus componentes
   - Reserva el inventario correspondiente

**Identificadores de orden telefónica:** Cada asesor tiene un prefijo asignado. Las órdenes telefónicas siguen el formato `{PREFIJO}-TLF-{NÚMERO}`. El número incrementa automáticamente para cada nueva orden del asesor.

---

### Despacho de Órdenes

El proceso de despacho documenta qué productos salen del almacén y actualiza el inventario:

**Desde AppSheet:**

1. Localizar la orden en la vista de órdenes pendientes
2. Verificar que todos los productos estén preparados
3. Cambiar el estado de almacén a "despachado"
4. El sistema automáticamente:
   - Crea registros de despacho para cada producto
   - Crea registros de despacho para cada componente
   - Reduce las cantidades reservadas
   - Registra los movimientos de inventario

**Desde Shopify (fulfillment automático):**

Cuando Shopify marca una orden como cumplida (fulfilled), el sistema recibe la notificación y ejecuta el mismo proceso de despacho automáticamente.

---

## Estados del Sistema

### Estados de Orden

| Estado | Significado | Siguiente Paso |
|--------|-------------|----------------|
| Pedido recibido | Orden nueva, pendiente preparación | Preparar productos |
| En preparación | Almacén trabajando en el pedido | Completar preparación |
| Listo para envío | Productos empacados | Entregar a paquetería |
| despachado | Orden enviada | Seguimiento de entrega |
| Entregado | Cliente recibió el pedido | Cerrado |
| Cancelado | Orden cancelada | Revertir reservas |

### Estados Financieros

| Estado | Significado |
|--------|-------------|
| paid | Pago completo recibido |
| pending | Pago pendiente |
| partially_paid | Pago parcial recibido |
| refunded | Reembolso procesado |

### Estados de Fulfillment

| Estado | Significado |
|--------|-------------|
| unfulfilled | Pendiente de envío |
| partial | Envío parcial realizado |
| fulfilled | Completamente enviado |

---

## Ubicaciones y Almacenes

El sistema maneja dos ubicaciones principales:

**Playa Regatas (PR)**
- Código interno: 21449925
- Almacén principal
- Maneja el mayor volumen de inventario

**Tienda Pilares (TP)**
- Código interno: 63600984166
- Punto de venta y almacén secundario
- Inventario para venta directa

### Consideraciones por ubicación:

- Cada producto tiene cantidades independientes por ubicación
- Algunas recetas de productos varían según la ubicación
- Las órdenes se asignan a una ubicación específica para fulfillment
- Los traspasos mueven inventario entre ubicaciones

---

## Productos y Recetas

### Productos Simples

Productos individuales que se venden y despachan tal cual. Al venderse, se descuenta directamente del inventario.

### Productos Compuestos

Productos que se ensamblan a partir de múltiples componentes. Cuando se vende un producto compuesto:

1. El sistema consulta la receta del producto
2. Identifica todos los componentes necesarios
3. Calcula las cantidades según la cantidad vendida
4. Reserva inventario para cada componente
5. Al despachar, reduce inventario de cada componente

**Ejemplo:** Si el producto "Cama Completa" tiene una receta con 15 componentes (colchón, base, almohadas, sábanas, etc.), la venta de 1 unidad reservará los 15 componentes en las cantidades especificadas en la receta.

### Consulta de Recetas en AppSheet

La vista de recetas muestra:
- Producto padre (el producto compuesto)
- Elemento componente (cada parte)
- Cantidad por producto
- Tipo de receta (SALIDA para ventas, ENTRADA para compras)
- Ubicación (si aplica específicamente a PR o TP)

---

## Inventario

### Campos de Inventario

| Campo | Descripción |
|-------|-------------|
| Cantidad Disponible | Stock físico en almacén |
| Cantidad Reservada | Asignado a órdenes pendientes |
| Cantidad Entrante | Esperada de órdenes de compra |
| Cantidad Virtual | Disponible - Reservada + Entrante |

### Reglas de Inventario

- La cantidad reservada se incrementa al crear una orden
- La cantidad reservada se reduce al despachar
- La cantidad disponible se reduce al despachar
- Los ajustes manuales deben registrarse con motivo

### Alertas de Inventario

Cuando un producto tiene cantidad virtual negativa o cercana a cero, considerar:
1. Verificar órdenes pendientes de despacho
2. Revisar órdenes de compra en tránsito
3. Evaluar necesidad de reorden

---

## Contabilidad Automática

El sistema genera asientos contables automáticamente basándose en reglas configuradas.

### Tipos de Asiento

Los asientos se categorizan por:
- Tipo de operación: Venta, Garantía, Intercambio, Obsequio
- Categoría del producto: Camas, Accesorios, Colchones, etc.
- Sucursal: General (PR), Tienda (TP)

### Reglas de Generación

Las reglas determinan:
- Qué cuenta contable afectar
- Cómo calcular el monto (basado en precio, costo, o fijo)
- Cuántos asientos generar por línea de orden

**Ejemplo:** Una venta de Cama en Playa Regatas puede generar múltiples asientos: ingreso por venta, costo de mercancía, y ajustes según el método de pago.

---

## Solución de Problemas Comunes

### Orden no aparece en AppSheet

**Posibles causas:**
1. La orden se acaba de crear (esperar 30 segundos y refrescar)
2. Filtro activo en la vista que excluye la orden
3. La orden está en otra ubicación

**Pasos a seguir:**
1. Refrescar la aplicación AppSheet
2. Verificar los filtros activos
3. Buscar por número de orden específico
4. Si persiste, reportar al administrador

### Producto sin inventario pero hay stock físico

**Posibles causas:**
1. El inventario está reservado para otras órdenes
2. El stock no se ha actualizado tras recepción
3. Error en el registro de ubicación

**Pasos a seguir:**
1. Verificar cantidad reservada vs disponible
2. Revisar órdenes pendientes que reservan el producto
3. Confirmar que el producto está en la ubicación correcta
4. Si hay discrepancia real, realizar ajuste de inventario

### Orden duplicada

**Posible causa:** Shopify envió la notificación dos veces

**Comportamiento esperado:** El sistema detecta duplicados automáticamente. La segunda notificación se ignora sin crear registros adicionales.

**Si ve duplicados reales:**
1. Verificar que sean órdenes diferentes (distintos order_id)
2. Si son idénticas, reportar al administrador para revisión

### Componentes no se desglosan

**Posibles causas:**
1. El producto no tiene receta configurada
2. La receta está inactiva
3. La receta es específica para otra ubicación

**Pasos a seguir:**
1. Verificar que el producto esté marcado como compuesto
2. Revisar que exista receta activa tipo SALIDA
3. Confirmar que la receta aplica a la ubicación de la orden

---

## Reportes Disponibles

### Reporte de Órdenes

Disponible en AppSheet, muestra:
- Órdenes por rango de fecha
- Filtrado por estado, ubicación, asesor
- Totales de venta

### Reporte de Inventario

Muestra stock actual por:
- Producto
- Ubicación
- Estado (disponible, reservado, entrante)

### Reporte de Despachos

Historial de productos despachados:
- Por orden
- Por fecha
- Por ubicación

---

## Contactos de Soporte

### Problemas de Sistema

Para errores técnicos, comportamiento inesperado, o datos incorrectos:
- Documentar el problema con capturas de pantalla
- Anotar la hora exacta del incidente
- Incluir números de orden o identificadores relevantes
- Escalar al administrador del sistema

### Solicitudes de Cambio

Para nuevas funcionalidades o modificaciones:
- Describir la necesidad operativa
- Proporcionar ejemplos de casos de uso
- Indicar la urgencia o prioridad

---

## Glosario

| Término | Definición |
|---------|------------|
| Edge Function | Programa que procesa las notificaciones de Shopify |
| Trigger | Regla automática que ejecuta acciones al cambiar datos |
| Webhook | Notificación instantánea entre sistemas |
| SKU | Código único de producto |
| Fulfillment | Proceso de preparación y envío de orden |
| Receta | Lista de componentes de un producto compuesto |
| Asiento contable | Registro de movimiento financiero |
| Idempotencia | Garantía de que una operación no se duplica |

---

## Actualizaciones del Sistema

Este documento refleja el estado del sistema a la fecha de creación. Las actualizaciones futuras pueden modificar:
- Flujos de trabajo
- Estados disponibles
- Campos en AppSheet
- Reglas de negocio

Consultar con el administrador del sistema para confirmar la vigencia de los procedimientos descritos.

---

*Versión del Documento: 1.0*  
*Versión del Sistema: 2.0*  
*Autor: Ivan Duarte, Full Stack Developer*
