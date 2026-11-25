# Guía de Resolución: Examen 1 — Reporte de Servicios Completados con Fecha de Recepción y Entrega

Esta guía contiene el paso a paso exacto, los archivos a modificar, las consultas SQL y el código Java necesario para resolver el **Examen 1** en el repositorio del proyecto **CarMotors SGT**.

---

## 🎯 Objetivo del Examen 1
1. Asegurar que las órdenes de servicio tengan registradas la **`fecha_recepcion`** (al crearse) y la **`fecha_entrega`** (al completarse o entregarse).
2. Crear una nueva opción de consulta/reporte que filtre únicamente los **servicios en estado COMPLETADO** (o entregados).
3. Mostrar en la consola un reporte tabulado que desglose cada servicio completado con su ID, vehículo/placa, tipo de mantenimiento, **`fecha_recepcion`**, **`fecha_entrega`** y costo total.
4. Procesar y ordenar el reporte utilizando **Java Stream API** (`.stream()`, `.filter()`, `.sorted()`).

---

## 📂 Archivos Involucrados en el Proyecto

| Capa | Archivo | Modificación |
| :--- | :--- | :--- |
| **Base de Datos / DAO** | `src/main/java/com/carmotors/dao/jdbc/OrdenServicioDAOJdbc.java` | Actualizar `fecha_entrega` con `NOW()` al pasar a `COMPLETADO`. |
| **Modelo / DTO** | `src/main/java/com/carmotors/modelo/ResumenServicioCompletado.java` *(Nuevo)* | Objeto DTO para transportar los datos del reporte. |
| **DAO Reportes** | `src/main/java/com/carmotors/dao/ReporteDAO.java`<br>`src/main/java/com/carmotors/dao/jdbc/ReporteDAOJdbc.java` | Agregar la consulta SQL y mapeo de servicios completados. |
| **Servicio** | `src/main/java/com/carmotors/servicio/ReporteServicio.java` | Agregar método con Stream API para ordenar/filtrar los servicios completados. |
| **Vista / Controlador** | `src/main/java/com/carmotors/vista/ReporteVista.java`<br>`src/main/java/com/carmotors/controlador/ReporteController.java` | Desplegar la tabla en consola y registrar la nueva opción en el menú. |

---

## 🛠️ Paso a Paso para la Solución

### Paso 1: Garantizar la asignación de `fecha_entrega` al completar la orden

Ubicación: **`src/main/java/com/carmotors/dao/jdbc/OrdenServicioDAOJdbc.java`**

En el método `actualizarEstado(int id, EstadoServicio estado)`, cuando el estado cambia a `COMPLETADO` (o `ENTREGADO`), debemos actualizar también el campo `fecha_entrega` con la fecha y hora actual (`NOW()` en MySQL):

```java
@Override
public boolean actualizarEstado(int id, EstadoServicio estado) {
    // Si el estado es COMPLETADO (id 3) o ENTREGADO (id 5), registramos fecha_entrega = NOW()
    String sql;
    if (estado == EstadoServicio.COMPLETADO || estado == EstadoServicio.ENTREGADO) {
        sql = "UPDATE servicios SET estado_id = ?, fecha_entrega = NOW() WHERE id = ?";
    } else {
        sql = "UPDATE servicios SET estado_id = ? WHERE id = ?";
    }

    try (PreparedStatement ps = conexion.prepareStatement(sql)) {
        ps.setInt(1, estado.getId());
        ps.setInt(2, id);
        return ps.executeUpdate() > 0;
    } catch (SQLException e) {
        throw new ExcepcionDatos("Error al actualizar estado de la orden #" + id, e);
    }
}
```

---

### Paso 2: Crear el DTO `ResumenServicioCompletado.java`

Ubicación: **`src/main/java/com/carmotors/modelo/ResumenServicioCompletado.java`** *(Crear este archivo en la carpeta `modelo`)*

```java
package com.carmotors.modelo;

public class ResumenServicioCompletado {
    private final int id;
    private final String placaVehiculo;
    private final String tipoMantenimiento;
    private final String fechaRecepcion;
    private final String fechaEntrega;
    private final double total;

    public ResumenServicioCompletado(int id, String placaVehiculo, String tipoMantenimiento, 
                                     String fechaRecepcion, String fechaEntrega, double total) {
        this.id = id;
        this.placaVehiculo = placaVehiculo;
        this.tipoMantenimiento = tipoMantenimiento;
        this.fechaRecepcion = fechaRecepcion != null ? fechaRecepcion : "N/A";
        this.fechaEntrega = fechaEntrega != null ? fechaEntrega : "Pendiente";
        this.total = total;
    }

    public int getId() { return id; }
    public String getPlacaVehiculo() { return placaVehiculo; }
    public String getTipoMantenimiento() { return tipoMantenimiento; }
    public String getFechaRecepcion() { return fechaRecepcion; }
    public String getFechaEntrega() { return fechaEntrega; }
    public double getTotal() { return total; }
}
```

---

### Paso 3: Agregar la consulta en el DAO de Reportes

#### 3.1 Actualizar la Interfaz `ReporteDAO.java`
Ubicación: **`src/main/java/com/carmotors/dao/ReporteDAO.java`**

Agregar la firma del método:
```java
List<ResumenServicioCompletado> serviciosCompletados();
```

#### 3.2 Implementar en `ReporteDAOJdbc.java`
Ubicación: **`src/main/java/com/carmotors/dao/jdbc/ReporteDAOJdbc.java`**

Implementar la consulta SQL que une las tablas `servicios`, `vehiculos` y `mantenimiento_tipos`, filtrando únicamente por estado `COMPLETADO` (o id de estado `= 3`):

```java
@Override
public List<ResumenServicioCompletado> serviciosCompletados() {
    String sql = """
        SELECT s.id, 
               v.placa, 
               m.nombre AS tipo_mantenimiento, 
               s.fecha_recepcion, 
               s.fecha_entrega,
               (s.costo_mano_obra + COALESCE((SELECT SUM(sr.cantidad_usada * r.precio_venta) 
                                              FROM servicios_repuestos sr 
                                              JOIN repuestos r ON sr.repuesto_id = r.id 
                                              WHERE sr.servicio_id = s.id), 0)) AS total
        FROM servicios s
        JOIN vehiculos v ON s.vehiculo_id = v.id
        JOIN mantenimiento_tipos m ON s.mantenimiento_tipo_id = m.id
        WHERE s.estado_id IN (3, 4, 5) -- 3: Completado, 4: Facturado, 5: Entregado
        ORDER BY s.fecha_entrega DESC
        """;

    List<ResumenServicioCompletado> resultado = new ArrayList<>();
    try (PreparedStatement ps = conexion.prepareStatement(sql);
         ResultSet rs = ps.executeQuery()) {
        while (rs.next()) {
            resultado.add(new ResumenServicioCompletado(
                rs.getInt("id"),
                rs.getString("placa"),
                rs.getString("tipo_mantenimiento"),
                rs.getString("fecha_recepcion"),
                rs.getString("fecha_entrega"),
                rs.getDouble("total")
            ));
        }
    } catch (SQLException e) {
        throw new ExcepcionDatos("Error al obtener reporte de servicios completados", e);
    }
    return resultado;
}
```

---

### Paso 4: Agregar el método de negocio en `ReporteServicio.java`

Ubicación: **`src/main/java/com/carmotors/servicio/ReporteServicio.java`**

Utilizar **Java Stream API** para procesar, filtrar campos nulos y ordenar la lista devuelta por el DAO:

```java
public List<ResumenServicioCompletado> serviciosCompletados() {
    return reporteDAO.serviciosCompletados().stream()
        .filter(s -> s.getFechaEntrega() != null && !s.getFechaEntrega().equals("Pendiente"))
        .sorted(Comparator.comparing(ResumenServicioCompletado::getFechaEntrega).reversed())
        .toList();
}
```

---

### Paso 5: Desplegar en la Vista y el Controlador

#### 5.1 Método en `ReporteVista.java`
Ubicación: **`src/main/java/com/carmotors/vista/ReporteVista.java`**

Agregar el método que da formato tabulado a la respuesta:

```java
public void mostrarServiciosCompletados(List<ResumenServicioCompletado> lista) {
    List<String> lineas = new ArrayList<>();
    for (ResumenServicioCompletado s : lista) {
        lineas.add(String.format("Orden #%-4d | Vehículo: %-8s | Tipo: %-12s | Recepción: %s | Entrega: %s | Total: %s",
            s.getId(),
            s.getPlacaVehiculo(),
            s.getTipoMantenimiento(),
            s.getFechaRecepcion(),
            s.getFechaEntrega(),
            Formato.moneda(s.getTotal())
        ));
    }
    imprimir("SERVICIOS COMPLETADOS (CON FECHA RECEPCIÓN Y ENTREGA)", lineas, "No hay servicios completados registrados.");
}
```

#### 5.2 Registrar la Opción en `ReporteVista.java` y `ReporteController.java`

En **`ReporteVista.java`**, agregar la opción al mapa de menú:
```java
OPCIONES.put("7", "Servicios completados (Fechas de Recepción y Entrega)");
```

En **`ReporteController.java`**, registrar la acción en el constructor:
```java
acciones.put("7", this::serviciosCompletados);
```

Y agregar el método privado en **`ReporteController.java`**:
```java
private void serviciosCompletados() {
    vista.mostrarServiciosCompletados(reporteServicio.serviciosCompletados());
}
```

---

## 🧪 Verificación y Prueba de Funcionamiento

1. Inicia la aplicación y crea una orden de servicio (quedará con `fecha_recepcion` actual).
2. Pasa la orden a **`EN_PROCESO`** y luego a **`COMPLETADO`** (la consulta actualizará `fecha_entrega` con `NOW()`).
3. Ve a la opción **`6. Reportes gerenciales`** -> **`7. Servicios completados`**.
4. Verifica que la consola imprima correctamente la orden con ambas fechas visibles.
