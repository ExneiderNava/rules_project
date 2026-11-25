# Guía de Examen #3: Ajuste de Consulta SQL Existente de acuerdo a una nueva condición

Esta guía proporciona la solución paso a paso para el **Examen 3**, estructurada y adaptada al 100% a la arquitectura real de **CarMotors SGT** (`com.carmotors.dao`, `com.carmotors.dao.jdbc`, `com.carmotors.servicio`, `com.carmotors.controlador`).

---

## 1. Diagnóstico del Problema y Justificación Técnica

### Estado Actual en el Proyecto:
En el código base actual de CarMotors:
1. `OrdenServicioDAOJdbc.java` posee únicamente consultas SQL genéricas sin condiciones de filtrado complejas:
   - `listar()` -> `SELECT id, vehiculo_id, mantenimiento_tipo_id, fecha_recepcion, fecha_entrega, descripcion_problema, costo_mano_obra, estado_id FROM servicios ORDER BY fecha_recepcion, id` (trae la totalidad de filas de la tabla).
   - `buscarPorId(id)` -> `... WHERE id = ?`
   - `listarPorVehiculo(vehiculoId)` -> `... WHERE vehiculo_id = ?`
2. En `OrdenServicioServicio.java`, el método `listarPorEstado(EstadoServicio estado)` ejecuta `ordenDAO.listar()` extrayendo **todos los registros de la base de datos a memoria Heap de Java** y luego aplica un filtro secundario con Streams (`.stream().filter(o -> o.getEstado() == estado).toList()`).

### Diagnóstico de Ingeniería:
- **Ineficiencia de Red y Memoria:** Transferir cientos o miles de filas desde MySQL a la JVM para descartar la gran mayoría en memoria Java es una mala práctica de rendimiento (*anti-patrón de filtrado en memoria*).
- **Ajuste Requerido para el Examen:** Mover la condición de filtrado directamente a la consulta SQL mediante la cláusula **`WHERE`** con **`PreparedStatement`** (garantizando protección contra inyección SQL).

---

## 2. Paso a Paso para Resolver el Examen 3

### Paso 1: Actualizar la Interfaz DAO (`OrdenServicioDAO.java`)

Abre el archivo `src/main/java/com/carmotors/dao/OrdenServicioDAO.java` y declara los métodos de búsqueda con las nuevas condiciones de filtrado SQL:

```java
package com.carmotors.dao;

import com.carmotors.modelo.EstadoServicio;
import com.carmotors.modelo.OrdenServicio;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

public interface OrdenServicioDAO {
    void guardar(OrdenServicio orden);
    Optional<OrdenServicio> buscarPorId(int id);
    List<OrdenServicio> listar();
    List<OrdenServicio> listarPorVehiculo(int vehiculoId);
    
    // === NUEVOS MÉTODOS PARA EL EXAMEN 3 ===
    // Condición 1: Filtrado directo en SQL por estado_id
    List<OrdenServicio> listarPorEstado(EstadoServicio estado);

    // Condición 2: Filtrado directo en SQL por rango de fechas (fecha_recepcion)
    List<OrdenServicio> listarPorRangoFechas(LocalDateTime inicio, LocalDateTime fin);
    
    void actualizarEstado(int id, EstadoServicio estado);
}
```

---

### Paso 2: Implementar la Consulta SQL Parametrizada (`OrdenServicioDAOJdbc.java`)

Abre `src/main/java/com/carmotors/dao/jdbc/OrdenServicioDAOJdbc.java` e implementa los métodos ajustando la sentencia SQL con la cláusula `WHERE`:

```java
package com.carmotors.dao.jdbc;

import com.carmotors.dao.OrdenServicioDAO;
import com.carmotors.modelo.EstadoServicio;
import com.carmotors.modelo.OrdenServicio;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Timestamp;
import java.time.LocalDateTime;
import java.util.List;

public class OrdenServicioDAOJdbc implements OrdenServicioDAO {
    
    private static final String COLUMNAS = 
        "SELECT id, vehiculo_id, mantenimiento_tipo_id, fecha_recepcion, fecha_entrega, descripcion_problema, costo_mano_obra, estado_id FROM servicios";

    // =========================================================================
    // CONDICIÓN 1: Ajuste de consulta SQL por ESTADO (WHERE estado_id = ?)
    // =========================================================================
    @Override
    public List<OrdenServicio> listarPorEstado(EstadoServicio estado) {
        // Nueva condición SQL integrada directamente en el motor relacional
        String sql = COLUMNAS + " WHERE estado_id = ? ORDER BY fecha_recepcion DESC, id DESC";
        
        try (Connection con = ConexionFactory.getConexion();
             PreparedStatement ps = con.prepareStatement(sql)) {
            
            // Asignación del parámetro posicional de forma segura
            ps.setInt(1, estado.getId());
            
            return todas(ps); // Utiliza el método helper 'todas()' que procesa el ResultSet
        } catch (SQLException e) {
            throw new RuntimeException("Error al consultar órdenes por estado en BD: " + e.getMessage(), e);
        }
    }

    // =========================================================================
    // CONDICIÓN 2: Ajuste de consulta SQL por RANGO DE FECHAS (BETWEEN ? AND ?)
    // =========================================================================
    @Override
    public List<OrdenServicio> listarPorRangoFechas(LocalDateTime inicio, LocalDateTime fin) {
        // Nueva condición SQL con cláusula BETWEEN
        String sql = COLUMNAS + " WHERE fecha_recepcion BETWEEN ? AND ? ORDER BY fecha_recepcion DESC";
        
        try (Connection con = ConexionFactory.getConexion();
             PreparedStatement ps = con.prepareStatement(sql)) {
            
            ps.setTimestamp(1, Timestamp.valueOf(inicio));
            ps.setTimestamp(2, Timestamp.valueOf(fin));
            
            return todas(ps);
        } catch (SQLException e) {
            throw new RuntimeException("Error al consultar órdenes por rango de fechas en BD: " + e.getMessage(), e);
        }
    }
}
```

---

### Paso 3: Refactorizar la Capa de Servicio (`OrdenServicioServicio.java`)

Abre `src/main/java/com/carmotors/servicio/OrdenServicioServicio.java` y refactoriza el método `listarPorEstado` para eliminar el filtro en memoria y delegar la consulta ajustada directamente a MySQL:

```java
// REFACTORIZACIÓN EN OrdenServicioServicio.java

// ANTES (Ineficiente: Traía todo a memoria Java y filtraba con Streams):
// public List<OrdenServicio> listarPorEstado(EstadoServicio estado) {
//     return ordenDAO.listar().stream().filter(orden -> orden.getEstado() == estado).toList();
// }

// AHORA (Optimizado: Delegación directa a la consulta SQL con WHERE):
public List<OrdenServicio> listarPorEstado(EstadoServicio estado) {
    if (estado == null) {
        throw new IllegalArgumentException("El estado no puede ser nulo.");
    }
    return ordenDAO.listarPorEstado(estado);
}

// NUEVO MÉTODO DE SERVICIO PARA FILTRAR POR RANGO DE FECHAS:
public List<OrdenServicio> listarPorRangoFechas(LocalDateTime inicio, LocalDateTime fin) {
    if (inicio == null || fin == null || inicio.isAfter(fin)) {
        throw new IllegalArgumentException("El rango de fechas proporcionado es inválido.");
    }
    return ordenDAO.listarPorRangoFechas(inicio, fin);
}
```

---

### Paso 4: Actualización en el Controlador (`OrdenServicioController.java`)

En `src/main/java/com/carmotors/controlador/OrdenServicioController.java`, la opción `"3"` (*Seguimiento: órdenes por estado*) llamará directamente al método refactorizado:

```java
private void listarPorEstado() {
    EstadoServicio estado = pedirEstado();
    if (estado != null) {
        // Invoca al servicio que ahora ejecuta el filtro parametrizado en MySQL
        List<OrdenServicio> ordenes = servicio.listarPorEstado(estado);
        vista.mostrarOrdenes(ordenes);
    }
}
```

---

## 3. Preguntas de Examen y Sustentación Oral ante el Profesor

| Pregunta Probable del Profesor | Respuesta Breve y Concisa |
| :--- | :--- |
| **¿Por qué es superior filtrar en la consulta SQL (`WHERE`) que usar `.stream().filter()` en Java?** | *"Porque filtrar en SQL reduce dramáticamente el tráfico de red entre la base de datos y la aplicación, minimizando la memoria asignada en el Heap de la JVM. MySQL aprovecha los índices relacionales para resolver la búsqueda en milisegundos."* |
| **¿Cómo garantizas la seguridad ante ataques de SQL Injection al ajustar consultas?** | *"Utilizando siempre `PreparedStatement` con marcadores posicionales `?`. La sentencia se envía precompilada al motor de base de datos y los valores de los parámetros se tratan estrictamente como literales de datos, nunca como sintaxis SQL ejecutable."* |
| **¿Qué diferencia hay entre `executeQuery()` y `executeUpdate()` en JDBC?** | *"`executeQuery()` se utiliza para sentencias de lectura (`SELECT`) devolviendo un objeto `ResultSet`, mientras que `executeUpdate()` se aplica a modificaciones (`INSERT`, `UPDATE`, `DELETE`) devolviendo un número entero con las filas afectadas."* |
