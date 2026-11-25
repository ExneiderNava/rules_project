# Guía Paso a Paso — Examen 2: Aplicación de Hilos y Resolución de Condición de Carrera (Race Condition)

---

## 📍 Contexto del Proyecto CarMotors SGT y Objetivo del Examen

En el proyecto **CarMotors SGT**, el inventario de repuestos es un recurso mutable compartido. Múltiples mecánicos o cajeros pueden intentar reservar o deducir unidades de un mismo repuesto de forma simultánea.

Currently, `InventarioServicio` y `RepuestoDAOJdbc` leen la cantidad actual de stock en memoria, calculan la nueva cantidad y ejecutan una sentencia SQL `UPDATE`. Sin control de concurrencia, si dos o más hilos ejecutan esta secuencia al mismo milisegundo, se produce una **Condición de Carrera (*Race Condition*)** y una **Pérdida de Actualización (*Lost Update*)**, corrompiendo la consistencia del inventario.

El objetivo de este examen es:
1. Crear una prueba simulada multihilo usando `Thread` y `Runnable`.
2. Demostrar el fallo de consistencia (fuga de datos por falta de sincronización).
3. Resolver el problema aplicando la palabra clave `synchronized` en la sección crítica.
4. Coordinar la finalización de los hilos usando `join()` antes de validar el resultado final en la base de datos.

---

## 🛠️ Estructura de Clases Reales en el Proyecto

* **Paquete Modelo:** `com.carmotors.modelo.Repuesto`
* **Paquete DAO:** `com.carmotors.dao.jdbc.RepuestoDAOJdbc`
* **Paquete Servicio:** `com.carmotors.servicio.InventarioServicio`
* **Paquete Controlador:** `com.carmotors.controlador.InventarioController`
* **Paquete Vista:** `com.carmotors.vista.InventarioVista`

---

## 📑 Paso 1: Entender la Condición de Carrera en Java

### Anatomía del Problema (Sin Sincronizar)
La operación de descuento de stock no es indivisible en hardware:
1. **Lectura:** El Hilo 1 lee `cantidad_stock = 10`.
2. **Intercalación:** El Hilo 2 lee `cantidad_stock = 10` antes de que el Hilo 1 guarde su cambio.
3. **Cálculo local:** Ambos hilos restan 1 unidad (`10 - 1 = 9`).
4. **Escritura:** Ambos hilos ejecutan `UPDATE repuestos SET cantidad_stock = 9`.
5. **Resultado:** Se realizaron 2 ventas, pero el inventario bajó de 10 a 9 en lugar de los **8 reales**.

---

## 💻 Paso 2: Modificar `InventarioServicio.java` para Sincronizar la Sección Crítica

Abre el archivo `com/carmotors/servicio/InventarioServicio.java` y agrega los métodos de prueba (uno sin sincronizar para evidenciar el fallo y otro sincronizado con `synchronized`):

```java
package com.carmotors.servicio;

import com.carmotors.dao.RepuestoDAO;
import com.carmotors.modelo.Repuesto;

public class InventarioServicio {

    private final RepuestoDAO repuestoDAO;

    public InventarioServicio(RepuestoDAO repuestoDAO) {
        this.repuestoDAO = repuestoDAO;
    }

    // =========================================================================
    // 1. MÉTODO SIN SINCRONIZAR (Genera Condición de Carrera)
    // =========================================================================
    public void deducirSinSincronizar(int repuestoId, int cantidad) {
        Repuesto repuesto = repuestoDAO.buscarPorId(repuestoId);
        if (repuesto != null && repuesto.getCantidadStock() >= cantidad) {
            // Simulamos una pequeña latencia de I/O para forzar la intercalación de hilos
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            int nuevoStock = repuesto.getCantidadStock() - cantidad;
            repuestoDAO.actualizarCantidades(repuestoId, nuevoStock, repuesto.getCantidadReservada());
        }
    }

    // =========================================================================
    // 2. MÉTODO SINCRONIZADO (Resuelve la Condición de Carrera)
    // =========================================================================
    // La palabra clave 'synchronized' garantiza exclusión mutua: un solo hilo
    // entra a la sección crítica a la vez adquiriendo el monitor de la instancia.
    public synchronized void deducirSincronizado(int repuestoId, int cantidad) {
        Repuesto repuesto = repuestoDAO.buscarPorId(repuestoId);
        if (repuesto != null && repuesto.getCantidadStock() >= cantidad) {
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            int nuevoStock = repuesto.getCantidadStock() - cantidad;
            repuestoDAO.actualizarCantidades(repuestoId, nuevoStock, repuesto.getCantidadReservada());
        }
    }
}
```

---

## 💻 Paso 3: Crear la Tarea Runnable para los Hilos

Dentro de `com.carmotors.controlador.InventarioController` o como una clase interna/lambda, implementaremos la tarea `Runnable` que ejecutarán los hilos concurrentes:

```java
// Tarea que ejecutará cada hilo
int repuestoIdPrueba = 1; // ID de repuesto existente con stock suficiente
int hilosTotales = 10;
int deduccionesPorHilo = 5; // Cada hilo restará 5 unidades
```

---

## 💻 Paso 4: Implementar la Simulación Multihilo con `start()` y `join()`

En `com/carmotors/controlador/InventarioController.java`, agrega el método que coordina la prueba y utiliza `join()` como barrera de sincronización para esperar a que todos los hilos terminen antes de consultar la base de datos:

```java
package com.carmotors.controlador;

import com.carmotors.servicio.InventarioServicio;
import com.carmotors.modelo.Repuesto;
import com.carmotors.dao.RepuestoDAO;
import java.util.ArrayList;
import java.util.List;

public class InventarioController extends ControladorBase {

    private final InventarioServicio inventarioServicio;
    private final RepuestoDAO repuestoDAO;

    public InventarioController(InventarioServicio inventarioServicio, RepuestoDAO repuestoDAO) {
        this.inventarioServicio = inventarioServicio;
        this.repuestoDAO = repuestoDAO;
    }

    public void simularConcurrencia() {
        int repuestoId = 1; // Asegurarse de usar un repuesto con suficiente stock (ej. 100)
        Repuesto inicial = repuestoDAO.buscarPorId(repuestoId);
        
        if (inicial == null) {
            System.out.println("[ERROR] Repuesto no encontrado para la prueba.");
            return;
        }

        int stockInicial = inicial.getCantidadStock();
        int numHilos = 10;
        int cantidadPorOperacion = 1;

        System.out.println("\n==============================================================");
        System.out.println("   SIMULACIÓN DE CONCURRENCIA - CONDICIÓN DE CARRERA (RACE CONDITION)");
        System.out.println("==============================================================");
        System.out.println("Stock Inicial en BD: " + stockInicial);
        System.out.println("Hilos concurrentes: " + numHilos);
        System.out.println("Operaciones por hilo: 1 descuento de " + cantidadPorOperacion + " unidad.");
        System.out.println("Stock Final Esperado: " + (stockInicial - numHilos));

        // ---------------------------------------------------------------------
        // PRUEBA 1: SIN SINCRONIZACIÓN (EVIDENCIA EL FALLO)
        // ---------------------------------------------------------------------
        System.out.println("\n---> Ejecutando SIN synchronized...");
        List<Thread> hilosSinSinc = new ArrayList<>();
        
        for (int i = 0; i < numHilos; i++) {
            Thread t = new Thread(() -> {
                inventarioServicio.deducirSinSincronizar(repuestoId, cantidadPorOperacion);
            }, "Hilo-SinSinc-" + i);
            hilosSinSinc.add(t);
            t.start(); // Inicia el hilo en un nuevo flujo de ejecución
        }

        // BARRERA DE SINCRONIZACIÓN: join()
        // El hilo principal (main) se bloquea hasta que TODOS los hilos secundarios terminen.
        for (Thread t : hilosSinSinc) {
            try {
                t.join(); 
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }

        Repuesto enFallo = repuestoDAO.buscarPorId(repuestoId);
        System.out.println("Stock Real en BD (Sin Synchronized): " + enFallo.getCantidadStock());
        if (enFallo.getCantidadStock() != (stockInicial - numHilos)) {
            System.out.println("⚠️  ¡CONDICIÓN DE CARRERA DETECTADA! Fuga de datos / Pérdida de actualización.");
        }

        // Restablecemos el stock para la segunda prueba
        repuestoDAO.actualizarCantidades(repuestoId, stockInicial, inicial.getCantidadReservada());

        // ---------------------------------------------------------------------
        // PRUEBA 2: CON SYNCHRONIZED (CORRIGE EL FALLO)
        // ---------------------------------------------------------------------
        System.out.println("\n---> Ejecutando CON synchronized...");
        List<Thread> hilosConSinc = new ArrayList<>();
        
        for (int i = 0; i < numHilos; i++) {
            Thread t = new Thread(() -> {
                inventarioServicio.deducirSincronizado(repuestoId, cantidadPorOperacion);
            }, "Hilo-ConSinc-" + i);
            hilosConSinc.add(t);
            t.start();
        }

        // Esperamos a que terminen los hilos protegidos
        for (Thread t : hilosConSinc) {
            try {
                t.join();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }

        Repuesto corregido = repuestoDAO.buscarPorId(repuestoId);
        System.out.println("Stock Real en BD (Con Synchronized): " + corregido.getCantidadStock());
        if (corregido.getCantidadStock() == (stockInicial - numHilos)) {
            System.out.println("✅ ¡PRUEBA EXITOSA! Sincronización correcta, consistencia garantizada.");
        }
        System.out.println("==============================================================\n");
    }
}
```

---

## 💻 Paso 5: Registrar la Opción en la Vista y Menú

### En `com/carmotors/vista/InventarioVista.java`:
Agrega la opción al menú de opciones de inventario:

```java
public static final List<String> OPCIONES = List.of(
    "1. Listar repuestos",
    "2. Ingresar lote (entrada de mercancía)",
    // ... opciones existentes ...
    "14. Simular prueba de hilos y condición de carrera (Race Condition)"
);
```

### En `com/carmotors/controlador/InventarioController.java`:
En el constructor, registra la opción `"14"`:

```java
registrar("14", this::simularConcurrencia);
```

---

## 🎓 Guía de Sustentación Oral para el Profesor

Si el profesor te pregunta sobre este punto en el examen, debes responder con esta terminología técnica:

1. **¿Qué es una Condición de Carrera (*Race Condition*)?**
   * *"Ocurre cuando múltiples hilos acceden y modifican un mismo estado compartido en memoria o base de datos de forma simultánea, y el resultado final depende del orden no determinista en que la CPU intercala sus ejecuciones."*

2. **¿Por qué `cantidad++` o `stock - cantidad` no es atómico?**
   * *"Porque en la CPU se descompone en 3 pasos: 1. Lectura desde memoria al registro local, 2. Modificación en registro local, 3. Escritura de regreso a la memoria principal. Si dos hilos leen antes de que uno escriba, se produce la pérdida de actualización (*Lost Update*)."*

3. **¿Qué función cumple `synchronized`?**
   * *"Adquiere el monitor intrínseco (cerrojo) del objeto. Garantiza la **exclusión mutua**: solo un hilo a la vez puede ejecutar el bloque protegido. Los demás hilos quedan en estado `BLOCKED` esperando que el monitor se libere."*

4. **¿Para qué se usa `t.join()`?**
   * *"Pausa la ejecución del hilo principal (`main`) hasta que el hilo `t` complete su ejecución (estado `TERMINATED`). Actúa como una barrera de sincronización para poder consultar e inspeccionar el resultado final en la base de datos de forma confiable."*

5. **Diferencia entre `t.start()` y `t.run()`:**
   * *"`t.start()` solicita a la JVM crear un nuevo hilo de ejecución nativo independiente. `t.run()` simplemente ejecuta el método en el hilo actual (`main`), por lo que no crea ninguna concurrencia real."*
