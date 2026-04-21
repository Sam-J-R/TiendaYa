1. BPM
@startuml DiagramaBPM_VentaDelivery

|Cliente|
start
note right: Solicitud de pedido con delivery
:Registrar pedido;
note right: Formulario de pedido

|Sistema|
:Validar datos del pedido;
if (¿Datos completos?) then (si)
  :Consultar inventario;
  if (¿Hay stock suficiente?) then (si)
    :Calcular total;
  else (no)
    :Mostrar productos sin stock;
    |Cliente|
    :Elegir opción;
  endif
else (no)
  :Solicitar corrección de datos;
  |Cliente|
  note right: regresa a "Registrar pedido"
  :Registrar pedido;
  note right: Formulario de pedido
  |Sistema|
  :Validar datos del pedido;
endif

|Cliente|
:Elegir método de pago;
if (¿Tipo de pago?) then (pago digital)
  |Sistema|
  :Generar QR / datos bancarios;
  |Cliente|
  :Realizar pago;
else (pago en efectivo)
  |Sistema|
  :Marcar pedido como pendiente de cobro;
endif

|Sistema|
:Registrar pedido;
:Asignar número de pedido;

|Pasarela de pago|
:Procesar transacción;
if (¿Pago confirmado?) then (si)
  note right: continúa flujo
else (no)
  :Notificar fallo de pago;
  note left: [evento mensaje] Fallo de pago\nnotificado al Cliente
  |Cliente|
  note left: Evento: Fallo de pago
  if (¿Reintentar?) then (si)
    |Cliente|
    :Realizar pago;
  else (no)
    |Pasarela de pago|
    :Cancelar pago;
    :Notificar cliente;
    note left: [evento mensaje]\nnotificado al Cliente
    stop
    note right: Pago fallido
  endif
endif

|Encargado|
:Recolectar productos;
if (¿Stock real coincide?) then (si)
  :Cambiar estado a listo para entrega;
  :Asignar repartidor;
else (no)
  :Notificar inconsistencia;
  |Sistema|
  :Consultar inventario;
  note right: regresa al flujo de inventario
endif

|Repartidor|
:Aceptar pedido;
:Recoger pedido;
:Entregar pedido;
if (¿Entrega exitosa?) then (si)
  if (¿Pago era efectivo?) then (si)
    :Cobrar cliente;
    :Registrar pago;
  else (no)
    note right: pago digital ya procesado
  endif
  |Encargado|
  :Actualizar inventario;
  :Registrar venta;
  :Generar comprobante;
  :Cerrar delivery;
  :Notificar cliente;
  note left: [evento mensaje]\nPedido recibido exitosamente
  stop
  note right: Pedido completado
else (no)
  if (¿Problema detectado?) then (si)
    if (problema) then (cliente no está)
      :Cliente no está;
      :Intentar contactar cliente;
      note right: Evento tiempo: 10-15 min
      :Retornar pedido;
    else (cliente rechaza)
      :Cliente rechaza;
      :Registrar motivo;
      :Retornar pedido;
    endif
    :Registrar incidencia;
    |Encargado|
    :Marcar estado "fallido";
    :Notificar al cliente;
    note left: [evento mensaje] Entrega fallida\nnotificado al Cliente
    stop
    note right: Entrega fallida
  else (no)
    note right: sin problema → actualizar inventario
    |Encargado|
    :Actualizar inventario;
    note right: conector "no" → regresa\na Actualizar inventario
  endif
endif

@enduml

2. ACTIVIDADES

@startuml DiagramaActividadesDonPepe

|Cliente|
start
:Realizar pedido;

|Sistema|
:Mostrar productos;

|Cliente|
:Ver productos;
:Seleccionar producto;
:Elegir cantidad de productos;

|Sistema|
:Verificar disponibilidad del producto;
if (¿producto disponible?\n(cantidad > 0)) then (si)
  if (¿cantidad disponible para el cliente?\n(cantidadCliente <= cantidadDisponible)) then (si)
    |Cliente|
    :Añadir al pedido;
    if (¿seleccionar más productos?) then (si)
      |Cliente|
      stop
      note right: conector 1\n(vuelve a "Seleccionar producto")
    else (no)
      if (¿confirmar pedido?) then (si)
        |Sistema|
        :Ingresar datos de envío;
        if (¿datos válidos?) then (si)
          :Calcular subtotal;
        else (no)
          :Pedir datos al usuario nuevamente;
          note right: conector 5\n(reintenta datos de envío)
        endif
      else (no)
        |Cliente|
        stop
        note right: conector 3\n(fin sin confirmar)
      endif
    endif
  else (no)
    |Cliente|
    :Modificar pedido;
    note right: conector 2\n(lleva a fin de flujo modificación)
  endif
else (no)
  |Sistema|
  :Mostrar aviso de no disponible;
  note right: conector 1\n(retorna al flujo principal)
endif

|Cliente|
:Elegir método de pago;

|Sistema|
fork
  -> efectivo;
  :Pagar mediante efectivo;
  :Ingresar corte de pago;
fork again
  -> QR;
  :Pagar mediante QR;
  :Generar QR;
  :Realizar pago;
fork again
  -> transferencia bancaria;
  :Pagar mediante transferencia bancaria;
  :Ingresar datos del destinatario;
  :Confirmar transferencia;
  :Ingresar monto y concepto;
  :Realizar pago;
end fork

|Sistema|
:Verificar pago;
if (¿pago confirmado?) then (si)
  :Marca como "Pagado";
else (no)
  :Notificar al cliente;
  |Cliente|
  :Recibir notificación;
  if (¿reintentar pago?) then (si)
    note right: conector 4\n(reintenta pago)
  else (no)
    note right: conector 3\n(retorna al flujo de pedido)
  endif
endif

|Sistema|
:Confirmar pedido;

fork
  :Actualizar estado del pedido\n(En preparación);
fork again
  :Asignar número de seguimiento;
fork again
  :Enviar detalle del pedido al personal;
end fork

|Encargado de ventas|
:Recibir pedido;
if (¿diferencia entre inventario\ndigital y real?) then (si)
  :Activar alerta;
  :Notificar al cliente;
  note right: conector 1\n(reinicia el proceso de selección)
else (no)
  :Actualizar estado del pedido\n(Listo para entregar);
  :Asignar repartidor;
  :Registrar repartidor;
  :Registrar datos de envío\n(hora de salida y dirección);
  :Actualizar estado del pedido\n(En camino);
endif

|Delivery|
:Recibir pedido;
:Ir a la dirección de entrega;
:Llegar a la dirección;
if (¿el cliente se encuentra\nen la dirección?) then (si)
  if (¿inconveniente con la\nentrega del pedido?) then (si)
    :Registrar motivo;
    note right: conector 3\n(regresa a flujo de encargado)
  else (no)
    if (¿pedido pagado anteriormente\ncon QR / transferencia?) then (no)
      :Recibir dinero en efectivo;
      :Entregar pedido;
    else (si)
      :Entregar pedido;
    endif
    :Marcar como entregado;
  endif
else (no)
  :Contactar al cliente;
  if (¿hay respuesta?) then (si)
    :Coordinar la entrega con el cliente;
    note right: regresa a decisión\n¿inconveniente con la entrega?
  else (no)
    :Reportar;
    note right: conector 3\n(regresa a flujo de encargado)
  endif
endif

|Sistema|
fork
  :Actualizar stock;
fork again
  :Registrar venta completada;
fork again
  :Asociar comprobante de pago;
  :Notificar al cliente;
fork again
  :Cierre de registro de delivery;
end fork

stop

@enduml

3. DIAGRAMA DE CLASES

@startuml DiagramaClases_DonPepe

skinparam classBackgroundColor #EEF4FF
skinparam classBorderColor #555555
skinparam arrowColor #444444
skinparam classHeaderBackgroundColor #C5D8F5

title Diagrama de Clases — Sistema Minimarket Tienda Ya

package "Personas" {
  class Cliente {
    +idCliente: int
    +nombre: String
    +telefono: String
    +direccion: String
    +registrarCliente(): void
    +actualizarDatos(): void
    +realizarPedido(): Pedido
    +confirmarRecepcion(): void
  }

  class Repartidor {
    +idRepartidor: int
    +nombre: String
    +telefono: String
    +estadoDisponible: boolean
    +aceptarEntrega(): void
    +entregarPedido(): void
    +registrarCobroEfectivo(): void
    +reportarIncidencia(): void
  }
}

package "Operaciones" {
  class Pedido {
    +idPedido: int
    +fechaHora: DateTime
    +direccionEntrega: String
    +estado: String
    +total: decimal
    +agregarProducto(producto: Producto, cantidad: int): void
    +quitarProducto(producto: Producto): void
    +calcularTotal(): decimal
    +confirmarPedido(): void
    +actualizarEstado(estado: String): void
  }

  class Venta {
    +idVenta: int
    +fechaVenta: DateTime
    +montoTotal: decimal
    +estadoVenta: String
    +comprobante: String
    +registrarVenta(): void
    +generarComprobante(): void
    +asociarPago(pago: Pago): void
    +completarVenta(): void
  }

  class Pago {
    +idPago: int
    +metodo: String
    +monto: decimal
    +estadoPago: String
    +fechaPago: DateTime
    +procesarPago(): void
    +confirmarPago(): void
    +marcarPendiente(): void
    +registrarCobro(): void
  }
}

package "Inventario" {
  class Producto {
    +idProducto: int
    +nombre: String
    +categoria: String
    +precio: decimal
    +stock: int
    +fechaVencimiento: Date
    +verificarDisponibilidad(cantidad: int): boolean
    +actualizarStock(cantidad: int): void
    +obtenerPrecio(): decimal
    +marcarVencido(): void
  }
}

' Relaciones
Cliente "1" --> "0..*" Pedido : realiza
Pedido "1" --> "1..*" Producto : contiene
Pedido "1" --> "1" Pago : registra
Pedido "0..*" --> "0..1" Repartidor : asignado a
Pedido "1" --> "1" Venta : genera
Venta "1" --> "1" Pago : respaldada por

@enduml