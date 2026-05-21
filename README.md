# TP_PATTERN_DESING

### Preguntas orientadoras

**Singleton**
> ¿Por qué un logger global es buen candidato para Singleton?

Un logger global es un excelente candidato para el patrón Singleton porque, se necesita una única instancia compartida en toda la aplicación.
En nuestro caso la usamos para el logs en `main`, y los observers `ConsolePrinter` y `AlertObserver`.
Basicamente, entraliza el manejo de logs.

>  ¿Qué problemas puede traer en aplicaciones multihilo?

En un entorno concurrente pueden aparecer dos problemas principales:

1. Creación múltiple de instancias

Si dos hilos ejecutan getInstance() al mismo tiempo, ambos podrían crear instancias distintas.

2. Escritura concurrente

Varios hilos podrían escribir simultáneamente, mezclando mensajes o generando inconsistencias.

> ¿Cómo se resolvería?

Se resolveria haciendo que el método `getInstance()` sea synchronized, o poniedo un bloqueo antes de generar la instancia del Logger.
En nuestro caso pusimos un semaforo, que solo permite el ingreso de un hilo a la consulta del `instance == null`.

**Strategy**

> ¿Qué deberías hacer (modificar o crear) si quisieras agregar un nuevo medio de transporte?

Para agregar un nuevo medio de transporte, no es necesario modificar las clases existentes, simplemente se crea una nueva estrategia que debe implementar la interface **TransportStrategy**.
Y para implementarla en runtime se la seteamos a `TransportMonitor` con `setStrategy(TransportStrategy strategy)`.

> ¿Qué principio SOLID describe esa propiedad del diseño?

Principio SOLID involucrado: `Open/Closed Principle (OCP)`

Las clases deben estar abiertas para extensión, pero cerradas para modificación.

El sistema puede extenderse con nuevas estrategias sin alterar el código existente.


**Observer** 
> ¿Qué pasa si un observer tarda mucho en procesar la notificación? 

Si un observer tarda mucho en procesar una notificación, el Subject queda bloqueado hasta que dicho observer termine.

> ¿Cómo se desacoplaría la velocidad del subject de la del observer?

Realizando notificaciones asíncronas, creando un hilo por observer.

4. **Integración**
> `ConsolePrinter` y `AlertObserver` usan el mismo logger pero de forma distinta. ¿Por qué eso es posible sin que se conozcan entre sí?

Porque cada clase recibe una referencia al Logger (la cual es siempre la misma instancia) y utiliza sus métodos sin importar quién más lo use.
Y pueden ser notificados los dos observer del mismo modo porque ambos dependen de una abstracción común (`interface TransportObserver`), e implemetan `onUpdate(TransportSnapshot transportSnapshot)`.

---

## Evidencia en ejecucion.

Al iniciar el programa mostramos un menu de control, el cual indica como interactuar para cambiar de strategy en runtime.

![](img/menu-de-control.png)

A medida que corre vamos cambiando de strategy.

![](img/switch-strategy.png)

Tambien, podemos cambiar los umbrales de costo y ETA, los setemos con valores aleatorimente bajos, para poder verificar el `logWarning` y `logError` de `AlertObserver` 

Cambio umbral de Costo:
![](img/cambio-umbral-cost.png)

Cambio umbral de ETA:
![alt text](img/cambio-umbral-ETA.png)


## Evidencia en codigo.
```java
// Evidencia de Singleton en la clase Logger

public class Logger {
    // 1. Instancia estática privada
    private static Logger instance = null;
    
    // 2. Constructor privado (evita que se instancie con "new" desde afuera)
    private Logger(){
        System.out.println("------Iniciando Logger------");
    }
    
    // 3. Método estático de acceso (Lazy Initialization con semáforo)
    public static Logger getInstance(){
        try {
            semaphore.acquire();
            if (instance == null){
                instance = new Logger();
            }
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        semaphore.release();
        return instance;
    }
    // ... métodos de log ...
}
```
```java
// Interfaz común para todas las estrategias

public interface TransportStrategy {
    public String getName();
    public double getCost();
    // ...
}

// Implementación concreta de una estrategia (Ej: Bicicleta)
public class Bicicleta implements TransportStrategy {
    public Bicicleta(double distance){ 
        // Lógica específica de cálculo de costo y ETA para Bicicleta
    }
    // ... sobreescritura de métodos ...
}

// El Contexto (TransportMonitor) cambiando la estrategia dinámicamente
public class TransportMonitor implements Subject {
    public TransportStrategy transportStrategy;

    public void setStrategy(TransportStrategy strategy){
        transportStrategy = strategy; // Se inyecta la nueva estrategia
        // ... actualización del snapshot ...
    }
}
```
```java
// El Subject manejando la lista de Observers (TransportMonitor)
public class TransportMonitor implements Subject {
    public ArrayList<TransportObserver> observers = new ArrayList<>();

    @Override
    public void subcribe(TransportObserver observer) {
        observers.add(observer);
    }

    @Override
    public void start(int intervalMs) {
        recalcular(intervalMs);
        // Notificación a todos los suscriptores
        for (int i=0; i < observers.size(); i++){
            TransportObserver observer = observers.get(i);
            observer.onUpdate(transportSnapshot);
        }
    }
}

// Un Observer reaccionando a la notificación (AlertObserver)
public class AlertObserver implements TransportObserver {
    @Override
    public void onUpdate(TransportSnapshot transportSnapshot) {
        // Lógica que reacciona de forma independiente al evento
        if(transportSnapshot.getCost() > umbralCost){
            logger.logWarning("Se está superando el presupuesto $$$");
        }
    }
}
```
