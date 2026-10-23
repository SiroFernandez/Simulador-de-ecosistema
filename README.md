# Simulador de ecosistema

Simulador de un ecosistema en Java en el que **ovejas** y **lobos** viven sobre un mapa dividido en regiones: se mueven, comen, se reproducen, huyen y cazan según su estado. Se puede ejecutar por línea de comandos (leyendo y escribiendo JSON) o con una **interfaz gráfica en Swing** donde se ve el mapa en directo y se pueden cambiar las regiones.

Proyecto académico de la asignatura *Tecnología de la Programación II* del Grado en Ingeniería Informática (Universidad Complutense de Madrid), realizado en pareja.

## Qué simula

### Animales
Todos los animales tienen energía, edad, deseo de apareamiento, velocidad y un rango de visión. En cada paso de la simulación pasan por uno de estos estados:

| Estado | Significado |
|---|---|
| `NORMAL` | Se mueve por el mapa y busca comida |
| `MATE` | Busca pareja para reproducirse |
| `HUNGER` | Tiene poca energía y busca comida (los lobos, una presa) |
| `DANGER` | Huye de un depredador (solo las ovejas) |
| `DEAD` | Muere y desaparece del ecosistema |

- **Oveja** (`sheep`, herbívora): come de las regiones y huye de los lobos.
- **Lobo** (`wolf`, carnívoro): persigue y caza a otros animales dentro de su rango de visión.

Cuando dos animales se aparean, la cría hereda la especie y una media de las características de sus padres (energía, velocidad y rango de visión), con una pequeña variación aleatoria.

### Estrategias de selección
Para elegir pareja, presa o peligro, cada animal usa una estrategia configurable:

- `first`: el primero que encuentra.
- `closest`: el más cercano.
- `youngest`: el más joven.

### Regiones
El mapa es una cuadrícula de regiones, y cada región reparte comida a los animales que hay en ella:

- `default`: región normal.
- `dynamic`: región con una reserva de comida que se regenera con el tiempo (parámetros `food` y `factor`). Cuantos más herbívoros hay en la región, menos comida recibe cada uno.

## Interfaz gráfica

- **Mapa** con los animales de cada especie en un color, el tiempo de simulación y el número de animales por especie. Con `h` se muestra la ayuda y con `s` se filtran los animales por estado.
- **Ventana de mapa** independiente y barra de estado.
- **Tablas** con información de las especies y de las regiones.
- **Diálogo para cambiar regiones**: se elige el tipo de región, el rango de filas y columnas y sus parámetros.
- **Barra de control**: abrir un archivo de configuración, ejecutar, detener, elegir el número de pasos y el retardo entre ellos.
- **Tres modos de ejecución**: `NORMAL` (en el hilo de la interfaz), `THREAD` (en un hilo propio) y `WORKER` (con `SwingWorker`).

## Arquitectura

El proyecto sigue el patrón **MVC** y organiza el código en paquetes:

```
src/simulator/
├── launcher/    Main: lectura de argumentos y arranque (batch o GUI)
├── control/     Controller: carga los datos y ejecuta la simulación
├── model/       Simulator, Animal (Sheep, Wolf), Region, RegionManager, estrategias...
├── factories/   Factorías y builders que crean objetos a partir de JSON
├── view/        Interfaz gráfica en Swing
└── misc/        Utilidades (Vector2D, números aleatorios...)
```

Patrones de diseño utilizados:

- **Factory / Builder**: los animales, las regiones y las estrategias se crean a partir de JSON mediante `BuilderBasedFactory` y un builder por tipo.
- **Strategy**: `SelectionStrategy` con sus implementaciones `SelectFirst`, `SelectClosest` y `SelectYoungest`.
- **Observer**: `Simulator` notifica los cambios (animales nuevos, regiones, avances y reinicios) a los observadores de la vista (`EcoSysObserver`).
- **Herencia y polimorfismo**: `Animal` es una clase abstracta de la que heredan `Sheep` y `Wolf`.

## Requisitos

- Java 17 o superior (el proyecto está configurado con JDK 18).
- Las librerías [org.json](https://github.com/stleary/JSON-java) y [Apache Commons CLI](https://commons.apache.org/proper/commons-cli/), que hay que añadir al classpath.
- Los iconos de la barra de control se cargan desde `resources/icons/` (`open.png`, `viewer.png`, `regions.png`, `run.png`, `stop.png`...), que debe estar junto al directorio desde el que se lanza la aplicación.

## Ejecución

Compilar (en Windows, usa `;` en lugar de `:` en el classpath):

```bash
javac -cp "lib/*" -d out $(find src -name "*.java")
```

Lanzar la aplicación:

```bash
java -cp "out:lib/*" simulator.launcher.Main [opciones]
```

| Opción | Descripción |
|---|---|
| `-m`, `--mode` | `gui` (por defecto) o `batch` |
| `-i`, `--input` | Archivo JSON de configuración (obligatorio en modo `batch`) |
| `-o`, `--output` | Archivo de salida con el estado inicial y final (obligatorio en modo `batch`) |
| `-t`, `--time` | Duración de la simulación en segundos (por defecto, 10) |
| `-dt`, `--delta-time` | Tiempo real por paso de simulación (por defecto, 0.03) |
| `-sv`, `--simple-viewer` | Muestra una ventana simple del mapa en modo `batch` |
| `-h`, `--help` | Muestra la ayuda |

Ejemplos:

```bash
# Interfaz gráfica, cargando una configuración
java -cp "out:lib/*" simulator.launcher.Main -m gui -i config.json

# Modo batch: 20 segundos de simulación y salida en JSON
java -cp "out:lib/*" simulator.launcher.Main -m batch -i config.json -o salida.json -t 20
```

## Formato del archivo de configuración

```json
{
  "width": 800,
  "height": 600,
  "cols": 20,
  "rows": 15,
  "regions": [
    { "row": [0, 14], "col": [0, 19], "spec": { "type": "default" } },
    {
      "row": [3, 5],
      "col": [3, 8],
      "spec": { "type": "dynamic", "data": { "food": 100.0, "factor": 2.0 } }
    }
  ],
  "animals": [
    {
      "amount": 20,
      "spec": {
        "type": "sheep",
        "data": {
          "mate_strategy": { "type": "first" },
          "danger_strategy": { "type": "closest" },
          "pos": { "x_range": [100, 300], "y_range": [100, 300] }
        }
      }
    },
    {
      "amount": 5,
      "spec": {
        "type": "wolf",
        "data": {
          "mate_strategy": { "type": "youngest" },
          "hunt_strategy": { "type": "closest" },
          "pos": { "x_range": [400, 700], "y_range": [300, 500] }
        }
      }
    }
  ]
}
```

- `regions` es opcional. Los rangos de `row` y `col` son inclusivos.
- `amount` es el número de animales que se crean con esa especificación.
- Si no se indican estrategias, se usa `first`. `pos` es opcional y define el rango en el que se coloca cada animal al azar.
