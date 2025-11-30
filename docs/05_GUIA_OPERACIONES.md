# Guia de Operaciones

## Sistema Abuelo Comodo v2.0

---

## Introduccion

Este documento esta disenado para el equipo operativo de Abuelo Comodo: personal de almacen, asesores de ventas, y administradores que interactuan diariamente con el sistema de inventario. No asume conocimientos tecnicos profundos, pero si familiaridad con las operaciones comerciales de la empresa y el uso basico de AppSheet.

El sistema ha evolucionado desde una arquitectura basada en Google Sheets hacia una plataforma de base de datos moderna. Aunque la interfaz de AppSheet permanece similar, los procesos subyacentes ahora operan en tiempo real, eliminando las esperas de cinco a quince minutos que caracterizaban al sistema anterior.

---

## Flujos de Trabajo Principales

### Procesamiento de Ordenes Shopify

Cuando un cliente completa una compra en la tienda en linea o en los puntos de venta fisicos, el sistema procesa la orden automaticamente:

**Secuencia de eventos:**

1. Cliente completa pago en Shopify
2. Shopify envia notificacion instantanea al sistema
3. El sistema crea el registro de orden
4. Para cada producto ordenado:
   - Si es producto compuesto, se desglosan los componentes automaticamente
   - Se reserva inventario para el producto y sus componentes
   - Se generan los asientos contables correspondientes
5. La orden aparece en AppSheet lista para preparacion

**Tiempo total de procesamiento:** Menos de 2 segundos

**Visualizacion en AppSheet:** Las ordenes de Shopify aparecen con el prefijo "SHO-" seguido del numero de orden. Por ejemplo: SHO-1234.

---

### Flujo de Prospectos Telefonicos

Los prospectos que llegan por llamada telefonica siguen un proceso de captura, seguimiento, y eventual conversion a orden:

**Captura del prospecto:**

1. El asesor recibe la llamada
2. Ingresa los datos del prospecto en la aplicacion AppSheet
3. El sistema asigna automaticamente un identificador unico
4. El prospecto queda vinculado a la linea telefonica correspondiente

**Conversion a orden:**

1. El asesor confirma los productos seleccionados
2. El sistema valida disponibilidad de inventario
3. Al marcar el prospecto como cliente ("es_cliente = true"), el sistema:
   - Crea automaticamente un registro de cliente
   - Genera la orden con el prefijo del asesor (ejemplo: ANA-TLF-45)
   - Procesa los productos y sus componentes
   - Reserva el inventario correspondiente

**Identificadores de orden telefonica:** Cada asesor tiene un prefijo asignado. Las ordenes telefonicas siguen el formato `{PREFIJO}-TLF-{NUMERO}`. El numero incrementa automaticamente para cada nueva orden del asesor.

---

### Sincronizacion Automatica con Shopify (NUEVO)

El sistema ahora sincroniza automaticamente las ordenes telefonicas con Shopify. Esto permite:

- Ver todas las ordenes (online y telefonicas) en un solo lugar
- Usar Shopify para el proceso de fulfillment unificado
- Mantener un inventario centralizado

**Como funciona:**

1. El asesor crea una orden telefonica en AppSheet
2. El sistema guarda la orden en Supabase
3. Automaticamente (en 2-3 segundos) la orden aparece en Shopify
4. El campo `uploaded_to_shopify` se marca como TRUE
5. El numero de orden Shopify se asigna (ejemplo: SHO-16062)

**Identificacion de ordenes sincronizadas:**

| Campo | Descripcion |
|-------|-------------|
| shopify_order_id | ID interno de Shopify |
| shopify_order_number | Numero visible en Shopify |
| order_number | Formato SHO-##### |
| uploaded_to_shopify | TRUE si ya esta en Shopify |
| shopify_upload_date | Fecha/hora de sincronizacion |

**Etiquetas en Shopify:** Las ordenes telefonicas aparecen con las etiquetas:
- `telefonico` - Identifica el origen
- `{nombre_vendedor}` - Asesor que creo la orden
- `{order_id}` - ID original (ejemplo: LAU-TLF-213)

**Nota importante:** Las ordenes historicas (creadas antes del 28 de noviembre 2025) NO se sincronizan automaticamente. Esto es una proteccion para evitar duplicados.

---

### Despacho de Ordenes

El proceso de despacho documenta que productos salen del almacen y actualiza el inventario:

**Desde AppSheet:**

1. Localizar la orden en la vista de ordenes pendientes
2. Verificar que todos los productos esten preparados
3. Cambiar el estado de almacen a "despachado"
4. El sistema automaticamente:
   - Crea registros de despacho para cada producto
   - Crea registros de despacho para cada componente
   - Reduce las cantidades reservadas
   - Registra los movimientos de inventario

**Desde Shopify (fulfillment automatico):**

Cuando Shopify marca una orden como cumplida (fulfilled), el sistema recibe la notificacion y ejecuta el mismo proceso de despacho automaticamente.

---

## Estados del Sistema

### Estados de Orden

| Estado | Significado | Siguiente Paso |
|--------|-------------|----------------|
| Pedido recibido | Orden nueva, pendiente preparacion | Preparar productos |
| En preparacion | Almacen trabajando en el pedido | Completar preparacion |
| Listo para envio | Productos empacados | Entregar a paqueteria |
| despachado | Orden enviada | Seguimiento de entrega |
| Entregado | Cliente recibio el pedido | Cerrado |
| Cancelado | Orden cancelada | Revertir reservas |
| Archivado | Orden eliminada (soft delete) | No requiere accion |

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
| unfulfilled | Pendiente de envio |
| partial | Envio parcial realizado |
| fulfilled | Completamente enviado |

---

## Ubicaciones y Almacenes

El sistema maneja dos ubicaciones principales:

**Playa Regatas (PR)**
- Codigo interno: 21449925
- Almacen principal
- Maneja el mayor volumen de inventario

**Tienda Pilares (TP)**
- Codigo interno: 63600984166
- Punto de venta y almacen secundario
- Inventario para venta directa

### Consideraciones por ubicacion:

- Cada producto tiene cantidades independientes por ubicacion
- Algunas recetas de productos varian segun la ubicacion
- Las ordenes se asignan a una ubicacion especifica para fulfillment
- Los traspasos mueven inventario entre ubicaciones

---

## Productos y Recetas

### Productos Simples

Productos individuales que se venden y despachan tal cual. Al venderse, se descuenta directamente del inventario.

### Productos Compuestos

Productos que se ensamblan a partir de multiples componentes. Cuando se vende un producto compuesto:

1. El sistema consulta la receta del producto
2. Identifica todos los componentes necesarios
3. Calcula las cantidades segun la cantidad vendida
4. Reserva inventario para cada componente
5. Al despachar, reduce inventario de cada componente

**Ejemplo:** Si el producto "Cama Completa" tiene una receta con 15 componentes (colchon, base, almohadas, sabanas, etc.), la venta de 1 unidad reservara los 15 componentes en las cantidades especificadas en la receta.

### Consulta de Recetas en AppSheet

La vista de recetas muestra:
- Producto padre (el producto compuesto)
- Elemento componente (cada parte)
- Cantidad por producto
- Tipo de receta (SALIDA para ventas, ENTRADA para compras)
- Ubicacion (si aplica especificamente a PR o TP)

---

## Inventario

### Campos de Inventario

| Campo | Descripcion |
|-------|-------------|
| Cantidad Disponible | Stock fisico en almacen |
| Cantidad Reservada | Asignado a ordenes pendientes |
| Cantidad Entrante | Esperada de ordenes de compra |
| Cantidad Virtual | Disponible - Reservada + Entrante |

### Reglas de Inventario

- La cantidad reservada se incrementa al crear una orden
- La cantidad reservada se reduce al despachar
- La cantidad disponible se reduce al despachar
- Los ajustes manuales deben registrarse con motivo

### Alertas de Inventario

Cuando un producto tiene cantidad virtual negativa o cercana a cero, considerar:
1. Verificar ordenes pendientes de despacho
2. Revisar ordenes de compra en transito
3. Evaluar necesidad de reorden

---

## Contabilidad Automatica

El sistema genera asientos contables automaticamente basandose en reglas configuradas.

### Tipos de Asiento

Los asientos se categorizan por:
- Tipo de operacion: Venta, Garantia, Intercambio, Obsequio
- Categoria del producto: Camas, Accesorios, Colchones, etc.
- Sucursal: General (PR), Tienda (TP)

### Reglas de Generacion

Las reglas determinan:
- Que cuenta contable afectar
- Como calcular el monto (basado en precio, costo, o fijo)
- Cuantos asientos generar por linea de orden

**Ejemplo:** Una venta de Cama en Playa Regatas puede generar multiples asientos: ingreso por venta, costo de mercancia, y ajustes segun el metodo de pago.

---

## Eliminacion de Ordenes (Soft Delete)

El sistema utiliza "soft delete" para eliminar ordenes. Esto significa que las ordenes no se borran permanentemente, sino que se marcan como archivadas.

### Procedimiento para Archivar una Orden

**Paso 1: En Shopify**
1. Ir a Orders y encontrar la orden
2. Click en "More actions" y seleccionar "Archive"
3. NO seleccionar "Cancel order" (puede generar reembolsos)

**Paso 2: En Supabase**
La orden se puede archivar usando la funcion del sistema:
```
archive_order(order_id, motivo, quien_archiva)
```

### Campos de Archivado

| Campo | Descripcion |
|-------|-------------|
| archived_at | Fecha/hora cuando se archivo |
| archived_reason | Motivo del archivado |
| archived_by | Quien realizo el archivado |

### Importante

- Las ordenes archivadas NO se eliminan de la base de datos
- Se pueden restaurar si es necesario
- Mantienen el historial para auditoria
- No afectan los calculos de inventario actual

---

## Solucion de Problemas Comunes

### Orden no aparece en AppSheet

**Posibles causas:**
1. La orden se acaba de crear (esperar 30 segundos y refrescar)
2. Filtro activo en la vista que excluye la orden
3. La orden esta en otra ubicacion

**Pasos a seguir:**
1. Refrescar la aplicacion AppSheet
2. Verificar los filtros activos
3. Buscar por numero de orden especifico
4. Si persiste, reportar al administrador

### Orden telefonica no aparece en Shopify

**Posibles causas:**
1. La orden es historica (creada antes del 28 de noviembre 2025)
2. Error en el numero de telefono (debe tener 10 digitos)
3. La orden no tiene productos agregados

**Pasos a seguir:**
1. Verificar que la orden tenga productos en order_items
2. Revisar que el telefono tenga 10 digitos
3. Verificar el campo `uploaded_to_shopify`
4. Revisar los logs del sistema si persiste

### Producto sin inventario pero hay stock fisico

**Posibles causas:**
1. El inventario esta reservado para otras ordenes
2. El stock no se ha actualizado tras recepcion
3. Error en el registro de ubicacion

**Pasos a seguir:**
1. Verificar cantidad reservada vs disponible
2. Revisar ordenes pendientes que reservan el producto
3. Confirmar que el producto esta en la ubicacion correcta
4. Si hay discrepancia real, realizar ajuste de inventario

### Orden duplicada

**Posible causa:** Shopify envio la notificacion dos veces

**Comportamiento esperado:** El sistema detecta duplicados automaticamente. La segunda notificacion se ignora sin crear registros adicionales.

**Si ve duplicados reales:**
1. Verificar que sean ordenes diferentes (distintos order_id)
2. Si son identicas, reportar al administrador para revision

### Componentes no se desglosan

**Posibles causas:**
1. El producto no tiene receta configurada
2. La receta esta inactiva
3. La receta es especifica para otra ubicacion

**Pasos a seguir:**
1. Verificar que el producto este marcado como compuesto
2. Revisar que exista receta activa tipo SALIDA
3. Confirmar que la receta aplica a la ubicacion de la orden

---

## Reportes Disponibles

### Reporte de Ordenes

Disponible en AppSheet, muestra:
- Ordenes por rango de fecha
- Filtrado por estado, ubicacion, asesor
- Totales de venta

### Reporte de Inventario

Muestra stock actual por:
- Producto
- Ubicacion
- Estado (disponible, reservado, entrante)

### Reporte de Despachos

Historial de productos despachados:
- Por orden
- Por fecha
- Por ubicacion

---

## Contactos de Soporte

### Problemas de Sistema

Para errores tecnicos, comportamiento inesperado, o datos incorrectos:
- Documentar el problema con capturas de pantalla
- Anotar la hora exacta del incidente
- Incluir numeros de orden o identificadores relevantes
- Escalar al administrador del sistema

### Solicitudes de Cambio

Para nuevas funcionalidades o modificaciones:
- Describir la necesidad operativa
- Proporcionar ejemplos de casos de uso
- Indicar la urgencia o prioridad

---

## Glosario

| Termino | Definicion |
|---------|------------|
| Edge Function | Programa que procesa las notificaciones de Shopify |
| Trigger | Regla automatica que ejecuta acciones al cambiar datos |
| Webhook | Notificacion instantanea entre sistemas |
| SKU | Codigo unico de producto |
| Fulfillment | Proceso de preparacion y envio de orden |
| Receta | Lista de componentes de un producto compuesto |
| Asiento contable | Registro de movimiento financiero |
| Idempotencia | Garantia de que una operacion no se duplica |
| Soft Delete | Marcar como eliminado sin borrar permanentemente |
| Sincronizacion | Proceso de enviar datos entre sistemas |

---

## Actualizaciones del Sistema

Este documento refleja el estado del sistema a la fecha de creacion. Las actualizaciones futuras pueden modificar:
- Flujos de trabajo
- Estados disponibles
- Campos en AppSheet
- Reglas de negocio

Consultar con el administrador del sistema para confirmar la vigencia de los procedimientos descritos.

---

*Version del Documento: 1.1*
*Version del Sistema: 2.0*
*Ultima Actualizacion: 28 de Noviembre, 2025*
*Autor: Ivan Duarte, Full Stack Developer*
