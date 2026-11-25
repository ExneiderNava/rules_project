# Guía de Examen 4: Exportación de Reporte Detallado a Formato .txt

Esta guía paso a paso permite implementar la funcionalidad de **exportación de reportes detallados a archivos de texto plano (.txt)** en el sistema CarMotors SGT, alineada con la arquitectura de capas, utilitarios de formateo (`Formato`) y servicios E/S con Java NIO (`Files`, `BufferedWriter`) existentes en el proyecto.

---

## 1. Mapeo de Arquitectura y Clases Involucradas

Para cumplir con el examen sin romper el diseño del proyecto ni repetir código, utilizaremos las siguientes clases:

* **Configuración (`ConfiguracionSistema.java`):** Defines la ruta estándar del directorio de almacenamiento de reportes (`./reportes/`).
* **Vista / Formateador (`ReporteVista.java` y `Formato.java`):** Construye la lista de líneas (`List<String>`) estructurando cabeceras, tablas alineadas y pies de página.
* **Servicio de Persistencia E/S (`ReporteServicio.java` / `FacturacionServicio.java`):** Maneja la creación física del archivo `.txt` en disco mediante `Files.newBufferedWriter`.
* **Controlador (`ReporteController.java`):** Conecta la interacción del menú con el servicio y confirma la exportación al usuario.

---

## 2. Paso a Paso de Implementación

### Paso 1: Configurar el Directorio de Almacenamiento (`ConfiguracionSistema.java`)

Abre `com.carmotors.config.ConfiguracionSistema` y añade la constante para la carpeta de reportes:

```java
package com.carmotors.config;

import java.nio.file.Path;

public class ConfiguracionSistema {
    public static final Path CARPETA_FACTURAS = Path.of("facturas");
    // NUEVA CONSTANTE PARA EXAMEN 4:
    public static final Path CARPETA_REPORTES = Path.of("reportes");
}
```

---

### Paso 2: Construir el Generador de Texto Plano (`ReporteVista.java`)

Abre `com.carmotors.vista.ReporteVista` y agrega el método encargado de estructurar las líneas del reporte impreso en formato `.txt`:

```java
package com.carmotors.vista;

import com.carmotors.modelo.OrdenServicio;
import java.util.ArrayList;
import java.util.List;

public class ReporteVista {

    public List<String> generarLineasReporteDetallado(OrdenServicio orden, String nombreCliente, String placaVehiculo) {
        List<String> lineas = new ArrayList<>();
        
        lineas.add("==============================================================");
        lineas.add("              TALLER AUTOMOTRIZ CARMOTORS                     ");
        lineas.add("           REPORTE DETALLADO DE ORDEN DE SERVICIO             ");
        lineas.add("==============================================================");
        lineas.add("Número de Orden   : #" + orden.getId());
        lineas.add("Estado Actual     : " + orden.getEstado().getNombre());
        lineas.add("Tipo Mantenimiento: " + orden.getTipoMantenimiento().getNombre());
        lineas.add("Fecha Recepción   : " + Formato.fechaHora(orden.getFechaRecepcion()));
        lineas.add("Fecha Entrega     : " + Formato.fechaHora(orden.getFechaEntrega()));
        lineas.add("--------------------------------------------------------------");
        lineas.add("DATOS DEL CLIENTE Y VEHÍCULO");
        lineas.add("  Cliente         : " + nombreCliente);
        lineas.add("  Placa Vehículo  : " + placaVehiculo);
        lineas.add("--------------------------------------------------------------");
        lineas.add("DESCRIPCIÓN DEL TRABAJO");
        lineas.add("  " + orden.getDescripcionProblema());
        lineas.add("--------------------------------------------------------------");
        lineas.add("COSTOS DEL SERVICIO");
        lineas.add("  Mano de Obra    : " + Formato.pesos(orden.getCostoManoObra()));
        lineas.add("==============================================================");
        lineas.add("Reporte generado automáticamente por CarMotors SGT.");
        
        return lineas;
    }
}
```

---

### Paso 3: Lógica de Escritura Física en Disco (`ReporteServicio.java`)

Abre `com.carmotors.servicio.ReporteServicio` e implementa el método de exportación utilizando la API Java NIO (`java.nio.file`):

```java
package com.carmotors.servicio;

import com.carmotors.config.ConfiguracionSistema;
import java.io.BufferedWriter;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardOpenOption;
import java.util.List;

public class ReporteServicio {

    public String exportarReporteTxt(String nombreArchivo, List<String> lineas) throws IOException {
        Path carpeta = ConfiguracionSistema.CARPETA_REPORTES;
        
        // 1. Asegurar la existencia del directorio objetivo
        Files.createDirectories(carpeta);
        
        // 2. Definir ruta absoluta/relativa del archivo .txt
        Path rutaArchivo = carpeta.resolve(nombreArchivo + ".txt");
        
        // 3. Escribir contenido con BufferedWriter y UTF-8
        try (BufferedWriter escritor = Files.newBufferedWriter(
                rutaArchivo,
                StandardCharsets.UTF_8,
                StandardOpenOption.CREATE,
                StandardOpenOption.TRUNCATE_EXISTING,
                StandardOpenOption.WRITE)) {
            
            for (String linea : lineas) {
                escritor.write(linea);
                escritor.newLine();
            }
        }
        
        return rutaArchivo.toAbsolutePath().toString();
    }
}
```

---

### Paso 4: Integración en Vista y Controlador (`ReporteController.java`)

1. **Añadir la opción en `ReporteVista.java`:**
```java
// En ReporteVista.java, registrar la opción en el mapa de OPCIONES:
OPCIONES.put("7", "Exportar reporte detallado a archivo .txt");
```

2. **Registrar la acción en `ReporteController.java`:**
```java
package com.carmotors.controlador;

import com.carmotors.modelo.OrdenServicio;
import com.carmotors.servicio.OrdenServicioServicio;
import com.carmotors.servicio.ReporteServicio;
import com.carmotors.vista.ReporteVista;
import com.carmotors.vista.VistaConsola;

import java.util.List;

public class ReporteController extends ControladorBase {

    private final ReporteServicio reporteServicio;
    private final OrdenServicioServicio ordenServicio;
    private final ReporteVista vista;

    public ReporteController(VistaConsola consola, ReporteVista vista, ReporteServicio reporteServicio, OrdenServicioServicio ordenServicio) {
        super(consola);
        this.vista = vista;
        this.reporteServicio = reporteServicio;
        this.ordenServicio = ordenServicio;

        // Registro de la opción de exportación .txt
        registrarAccion("7", this::exportarReporteTxt);
    }

    private void exportarReporteTxt() {
        int ordenId = consola.pedirEntero("Ingrese el ID de la orden a exportar: ");
        try {
            OrdenServicio orden = ordenServicio.buscarPorId(ordenId);
            if (orden == null) {
                consola.mostrarError("La orden #" + ordenId + " no existe.");
                return;
            }

            // Construir texto formateado desde la vista
            List<String> lineas = vista.generarLineasReporteDetallado(orden, "Cliente Registrado", "XYZ-123");
            
            // Guardar en disco mediante el servicio
            String nombreArchivo = "REPORTE_ORDEN_" + orden.getId();
            String rutaCompleta = reporteServicio.exportarReporteTxt(nombreArchivo, lineas);

            consola.mostrarExito("Reporte exportado exitosamente en:\n" + rutaCompleta);
        } catch (Exception e) {
            consola.mostrarError("Error al exportar el reporte: " + e.getMessage());
        }
    }
}
```

---

## 3. Matriz de Preguntas de Sustentación Oral para el Profesor

| Pregunta del Evaluador | Respuesta Técnica Explicativa |
| :--- | :--- |
| **1. ¿Por qué se utiliza `Files.newBufferedWriter` en lugar de `FileWriter` clásico?** | `Files.newBufferedWriter` pertenece a Java NIO.2. Ofrece mejor rendimiento con buffer en memoria, soporte explícito de codificación de caracteres (`StandardCharsets.UTF_8`) y opciones de apertura avanzadas (`TRUNCATE_EXISTING`, `CREATE`). |
| **2. ¿Qué ventaja ofrece el bloque `try-with-resources` en la escritura de archivos?** | Garantiza la liberación automática de recursos E/S (cierre del *stream* de escritura) sin importar si la ejecución finaliza con éxito o si se lanza una excepción `IOException`, evitando fugas de memoria (*file descriptor leaks*). |
| **3. ¿Por qué la construcción del texto está en la Vista y la escritura en el Servicio?** | Cumple con el principio de **Responsabilidad Única (SRP)** del patrón MVC. La vista sabe cómo formatear y presentar la información, mientras que la capa de servicio gestiona las reglas del sistema de archivos E/S. |
| **4. ¿Cómo se manejan los caracteres especiales y acentos en el archivo generado?** | Se especifica explícitamente `StandardCharsets.UTF_8` en el `BufferedWriter`, lo que asegura la compatibilidad multiplataforma (Windows, Linux, macOS) de tildes y caracteres especiales. |
