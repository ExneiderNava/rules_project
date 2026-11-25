# Guía de Examen 5: Aplicación del Patrón Observer para Notificaciones al Cliente

El patrón de diseño conductual **Observer** permite establecer una relación de dependencia uno-a-muchos entre objetos, de modo que cuando el objeto principal (el Sujeto) cambia de estado, todos sus dependientes (los Observadores) son notificados y actualizados automáticamente. En el proyecto CarMotors SGT, este patrón se encuentra implementado para alertas de stock bajo mediante `ObservadorInventario` y `PublicadorInventario`.

Esta guía detalla la extensión del patrón para notificar al cliente cada vez que su Orden de Servicio cambia de estado (por ejemplo, de `PENDIENTE` a `EN_PROCESO`, o de `EN_PROCESO` a `COMPLETADO`), garantizando un acoplamiento mínimo entre la lógica de negocio y los canales de notificación.

---

## 1. Análisis de Arquitectura del Proyecto

El proyecto almacena los patrones conductuales en el paquete `com.carmotors.servicio.observador`. Para mantener coherencia estructural con las clases existentes (`ObservadorInventario`, `PublicadorInventario`, `AlertaStockObservador`), se adopta la misma arquitectura:

```text
com.carmotors.servicio.observador/
├── ObservadorOrden.java               <-- Interfaz Observadora (Nueva)
├── PublicadorOrden.java                 <-- Sujeto / Publicador Singleton (Nuevo)
└── ClienteNotificacionObservador.java  <-- Observador Concreto para Clientes (Nuevo)
```

Dado que la entidad `OrdenServicio` solo almacena `vehiculoId`, la capa de servicio consulta la relación `VehiculoDAO` -> `ClienteDAO` para personalizar el mensaje enviando nombre, teléfono y correo electrónico del cliente.

---

## 2. Paso 1: Definir la Interfaz `ObservadorOrden`

Crea la interfaz `ObservadorOrden.java` dentro del paquete `com.carmotors.servicio.observador`. Esta interfaz declara el contrato de notificación que implementará cualquier observador de órdenes.

```java
package com.carmotors.servicio.observador;

import com.carmotors.modelo.Cliente;
import com.carmotors.modelo.EstadoServicio;
import com.carmotors.modelo.OrdenServicio;

public interface ObservadorOrden {
    void onCambioEstado(OrdenServicio orden, Cliente cliente, EstadoServicio estadoAnterior, EstadoServicio estadoNuevo);
}
```

---

## 3. Paso 2: Crear el Sujeto `PublicadorOrden` (Singleton Thread-Safe)

Crea la clase `PublicadorOrden.java` en el paquete `com.carmotors.servicio.observador`. Administra la lista de observadores suscritos e implementa el patrón **Singleton** para garantizar un único punto de publicación centralizado.

```java
package com.carmotors.servicio.observador;

import com.carmotors.modelo.Cliente;
import com.carmotors.modelo.EstadoServicio;
import com.carmotors.modelo.OrdenServicio;

import java.util.ArrayList;
import java.util.List;

public class PublicadorOrden {

    private static final PublicadorOrden INSTANCIA = new PublicadorOrden();
    private final List<ObservadorOrden> observadores = new ArrayList<>();

    private PublicadorOrden() {
    }

    public static PublicadorOrden getInstancia() {
        return INSTANCIA;
    }

    public synchronized void suscribir(ObservadorOrden observador) {
        if (observador != null && !observadores.contains(observador)) {
            observadores.add(observador);
        }
    }

    public synchronized void retirar(ObservadorOrden observador) {
        observadores.remove(observador);
    }

    public synchronized void notificarCambioEstado(OrdenServicio orden, Cliente cliente, EstadoServicio estadoAnterior, EstadoServicio estadoNuevo) {
        for (ObservadorOrden obs : observadores) {
            obs.onCambioEstado(orden, cliente, estadoAnterior, estadoNuevo);
        }
    }
}
```

---

## 4. Paso 3: Crear el Observador Concreto `ClienteNotificacionObservador`

Crea la clase `ClienteNotificacionObservador.java` en `com.carmotors.servicio.observador`. Inyecta un objeto `Consumer<String> salida` para desacoplar el canal de transmisión (consola, log o servicio externo) de la lógica del observador.

```java
package com.carmotors.servicio.observador;

import com.carmotors.modelo.Cliente;
import com.carmotors.modelo.EstadoServicio;
import com.carmotors.modelo.OrdenServicio;

import java.util.function.Consumer;

public class ClienteNotificacionObservador implements ObservadorOrden {

    private final Consumer<String> salida;

    public ClienteNotificacionObservador(Consumer<String> salida) {
        this.salida = salida;
    }

    @Override
    public void onCambioEstado(OrdenServicio orden, Cliente cliente, EstadoServicio estadoAnterior, EstadoServicio estadoNuevo) {
        String mensaje = String.format(
            "[NOTIFICACIÓN CLIENTE] Estimado(a) %s, su vehículo con orden #%d ha cambiado de estado: %s -> %s. Email: %s | Teléfono: %s",
            cliente != null ? cliente.getNombre() : "Cliente",
            orden.getId(),
            estadoAnterior != null ? estadoAnterior.getNombre() : "N/A",
            estadoNuevo.getNombre(),
            cliente != null ? cliente.getEmail() : "N/A",
            cliente != null ? cliente.getTelefono() : "N/A"
        );
        salida.accept(mensaje);
    }
}
```

---

## 5. Paso 4: Disparar el Evento en `OrdenServicioServicio`

En la clase `com.carmotors.servicio.OrdenServicioServicio`, recupera los datos del cliente a través del vehículo y notifica a `PublicadorOrden` al completar una transición de estado en la orden.

```java
// Ejemplo de integración en los métodos de transición de OrdenServicioServicio.java
public void cambiarEstado(int servicioId, EstadoServicio nuevoEstado) {
    OrdenServicio orden = ordenDAO.buscarPorId(servicioId);
    if (orden == null) {
        throw new IllegalArgumentException("Orden de servicio no encontrada.");
    }

    EstadoServicio estadoAnterior = orden.getEstado();
    ordenDAO.actualizarEstado(servicioId, nuevoEstado);

    // Consulta relacional para obtener los datos de contacto del cliente
    Vehiculo vehiculo = vehiculoDAO.buscarPorId(orden.getVehiculoId());
    Cliente cliente = (vehiculo != null) ? clienteDAO.buscarPorId(vehiculo.getClienteId()) : null;

    // Notificación a todos los observadores suscritos
    PublicadorOrden.getInstancia().notificarCambioEstado(orden, cliente, estadoAnterior, nuevoEstado);
}
```

---

## 6. Paso 5: Suscribir el Observador en `MenuPrincipalController`

Para activar las notificaciones en tiempo de ejecución, suscribe el nuevo observador dentro del constructor de `com.carmotors.controlador.MenuPrincipalController`.

```java
// Dentro del constructor de MenuPrincipalController.java
public MenuPrincipalController(...) {
    // Suscripción de alertas de inventario (existente)
    PublicadorInventario.getInstancia().suscribir(new AlertaStockObservador(consola::mostrarAlerta));

    // Suscripción de notificaciones de estado de órdenes al cliente (NUEVA)
    PublicadorOrden.getInstancia().suscribir(new ClienteNotificacionObservador(consola::mostrarAlerta));
}
```

---

## 7. Matriz de Sustentación Técnica para Evaluación

| Pregunta del Evaluador | Respuesta Técnica Sugerida |
| :--- | :--- |
| **¿Qué problema resuelve el patrón Observer en este examen?** | Resuelve el acoplamiento directo entre la lógica de negocio de órdenes (`OrdenServicioServicio`) y los múltiples canales de notificación (consola, email, SMS). |
| **¿Por qué se inyecta `Consumer<String>` en el observador?** | Cumple el principio de Inversión de Dependencias (D de SOLID), permitiendo redirigir los mensajes sin depender directamente de la consola ni de clases concretas de vista. |
| **Diferencia entre el modelo Push y Pull en Observer** | En el modelo **Push** (aplicado aquí), el Sujeto envía todos los objetos del evento (`orden`, `cliente`, `estados`) directamente al observador. En el modelo **Pull**, solo notifica el evento y el observador consulta los atributos que requiere. |
