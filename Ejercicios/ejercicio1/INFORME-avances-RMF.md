# Informe de avances — Open-RMF Facultat de Psicologia UV

Documento de referencia (y chuleta de examen) con todo el flujo de trabajo, comandos, errores
encontrados y su solución. Todo está probado en el contenedor Docker `rmf_demos:jazzy-rmf-latest`.

> **Convención de rutas:** dentro del contenedor el workspace es `/opt/rmf_ws` (= tu carpeta del host
> `~/uji/IR2134/rmf_ws`). El ejercicio está en `/opt/rmf_ws/Ejercicios/ejercicio1`.
> En cada terminal: `rmfenv` (sourcea Jazzy + rmf_demos + tu workspace).

---

## 1. Compilar y pasar los archivos de traffic-editor al workspace (sin errores)

### 1.1. Flujo completo
```
traffic-editor → guardas .building.yaml
                      │
                      ├─ regeneras .world      (building_map_generator gazebo)
                      ├─ regeneras nav_graphs   (building_map_generator nav)
                      │
                      └─ copias los 3 artefactos al paquete de mapas → relanzas
```

### 1.2. Regenerar mundo y grafos (dentro de `Ejercicios/ejercicio1`)
```bash
cd /opt/rmf_ws/Ejercicios/ejercicio1

# 1) Mundo de Gazebo (modelos 3D de paredes/suelos + robots + puertas)
ros2 run rmf_building_map_tools building_map_generator gazebo \
  psychology.building.yaml psychology.world psychology_world

# 2) Grafos de navegación (un .yaml por graph_idx; varias plantas van en el mismo si comparten graph_idx)
ros2 run rmf_building_map_tools building_map_generator nav \
  psychology.building.yaml nav_graphs

# 3) (opcional) Previsualizar el mundo en Gazebo antes de la sim completa
ros2 run rmf_building_map_tools building_map_model_downloader psychology.building.yaml -e ./models
export GZ_SIM_RESOURCE_PATH=`pwd`/psychology_world:`pwd`/models:/rmf_demos_ws/install/rmf_demos_assets/share/rmf_demos_assets/models
gz sim -r -v 4 psychology.world
```

### 1.3. Copiar los artefactos al paquete de mapas
El paquete `psychology_uv_maps` **solo instala** la carpeta `maps/` (`install(DIRECTORY maps ...)`),
NO regenera nada. Hay que copiar **los tres artefactos** (no solo el building.yaml).

> ⚠️ **Hazlo DENTRO del contenedor**: la carpeta `psychology_world/` la genera root, y desde el host
> (usuario normal) no se puede borrar/sobrescribir → "Permiso denegado".

```bash
DEST=/opt/rmf_ws/src/psychology_uv/psychology_uv_maps/maps/psychology
SRC=/opt/rmf_ws/Ejercicios/ejercicio1

cp $SRC/psychology.building.yaml $DEST/
cp $SRC/psychology.world          $DEST/
rm -rf $DEST/psychology_world && cp -r $SRC/psychology_world $DEST/
rm -rf $DEST/nav_graphs       && cp -r $SRC/nav_graphs       $DEST/
```

### 1.4. Compilar y recargar
```bash
cd /opt/rmf_ws
colcon build --packages-select psychology_uv_maps   # con --symlink-install la 1ª vez
rmfenv
```

### 1.5. REGLAS DE ORO (lo que evita el 90% de los "no se actualiza")
- **Copia SIEMPRE los 3 artefactos** (`building.yaml`, `psychology.world`, `nav_graphs/`). El fallo más
  típico es regenerar el nav graph pero copiar solo el building.yaml → sigue el grafo viejo.
- El `psychology.world` solo hace falta recopiarlo si cambiaste **paredes/puertas/suelos**. Si solo
  añadiste **carriles/waypoints** (que son navegación), el `.world` no cambia → basta `nav_graphs/`.
- Con **`--symlink-install`**, el `install/` apunta por symlink al `src/`. Por eso, tras la primera
  compilación, **sobrescribir archivos en `src/` no necesita `colcon build`**: basta **relanzar**.
  Solo necesitas recompilar si **añades un archivo nuevo** que no existía al compilar.
- Quién lee qué:
  - El **visualizador del grafo (RViz/dashboard)** lo publica el **fleet adapter** desde `nav_graphs/0.yaml`.
  - El **building map server** lee el `building.yaml` (`config_file`).
  - El **fleet adapter** planifica con `nav_graphs/<graph_idx>.yaml`.

---

## 2. Comandos para generar tareas / acciones desde terminal

> **SIEMPRE con `--use_sim_time`** en simulación. Sin él, la tarea se agenda en tiempo real (2026) y
> el robot nunca arranca.

### 2.1. Patrulla (patrol / loop)
```bash
# Patrulla por waypoints (cualquier robot libre de la flota):
ros2 run rmf_demos_tasks dispatch_patrol -p LIP1 aula-este -n 1 --use_sim_time

# Recorrido por varios waypoints, repetido n veces:
ros2 run rmf_demos_tasks dispatch_patrol -p LIP1 sala-grabacion AUR LON004CI aula-este -n 2 --use_sim_time

# Asignar a un robot CONCRETO (-F flota, -R robot):
ros2 run rmf_demos_tasks dispatch_patrol -F tinyRobot -R tinyRobot1 -p aula-este -n 1 --use_sim_time
```

### 2.2. Limpieza (clean)
```bash
# OJO: el nombre tras -cs debe ser un WAYPOINT real del nav graph
# Y a la vez una clave en action_paths.clean del config de la flota limpiadora.
ros2 run rmf_demos_tasks dispatch_clean -cs aula-este --use_sim_time
ros2 run rmf_demos_tasks dispatch_clean -cs sala-grabacion --use_sim_time
```

### 2.3. Entrega (delivery)
```bash
# -p pickup_wp  -ph nombre_dispenser   -d dropoff_wp  -dh nombre_ingestor
ros2 run rmf_demos_tasks dispatch_delivery -p AUR -ph coke_dispenser -d sala-grabacion -dh coke_ingestor --use_sim_time
```
Requiere: waypoint pickup con `pickup_dispenser`, waypoint dropoff con `dropoff_ingestor`
(nombres **distintos**), y modelos `TeleportDispenser`/`TeleportIngestor` en el `.world`.

### 2.4. Gestión y diagnóstico
```bash
# Cancelar una tarea:
ros2 run rmf_demos_tasks cancel_task -id <id_de_la_tarea>

# Alarma de incendio (todos a la salida):
ros2 topic pub -1 /fire_alarm_trigger std_msgs/Bool '{data: true}'

# DIAGNÓSTICO (esencial): por qué una tarea se acepta o falla, al instante:
ros2 topic echo /task_api_responses

# Ver flotas vivas / nodos / estado:
ros2 topic echo /fleet_states --once
ros2 node list | grep fleet
```

---

## 3. Errores encontrados y cómo resolverlos (¡para el examen!)

### 3.1. `Error Code 34: The value <ambient>128 192 210 0.1</ambient> is invalid`
- **Causa:** colores SDF en rango 0–255 (deben ser floats **0.0–1.0**). El modelo `RobotPlaceholder`
  de los assets los tenía mal y rompía la carga del `.world` entero.
- **Solución:** normalizar los colores (`128/255 = 0.502`, etc.) en el `model.sdf`, o eliminar del
  `building.yaml` los muebles `RobotPlaceholder` colocados por error en traffic-editor.

### 3.2. `Non-unique name[null] detected ... model with name[null] already exists`
- **Causa:** varias puertas con `name: [1, "null"]` (sin nombrar en traffic-editor) → todas se llaman
  igual y colisionan al generar el mundo.
- **Solución:** dar a cada puerta un **nombre único** (p. ej. `puerta172`, `puerta173`…) en traffic-editor,
  o por script en el `building.yaml`, y regenerar.

### 3.3. `Unable to find uri[model://deliveryRobot]` / `model://cleanRobot`
- **Causa:** el `spawn_robot_type` no coincide con el nombre real del modelo de los assets.
- **Solución:** usar los nombres EXACTOS de las carpetas de modelos: `DeliveryRobot`, `CleanerBotA`/`CleanerBotE`,
  `TinyRobot` (mayúsculas correctas).

### 3.4. `Unable to resolve uri[model://]` (model:// vacío)
- **Causa:** un vértice con `spawn_robot_name`/`spawn_robot_type` **vacíos** (p. ej. un punto de espera)
  genera un robot fantasma con URI vacía.
- **Solución:** quitarle los campos `spawn_robot_*` vacíos a ese vértice (deja `is_charger`/`is_parking_spot`
  si los necesitas).

### 3.5. Las dos plantas apiladas (ascensor de recorrido 0)
- **Causa:** ambos niveles con `elevation: 0` → plantas en el mismo Z.
- **Solución:** dar a la planta superior una `elevation` positiva (p. ej. `8.0`) en el `building.yaml`.

### 3.6. ⭐ Segmentation fault en `liblift.so` (`LiftPlugin::PreUpdate → Each<AxisAlignedBox, Pose>`)
- **Síntoma:** Gazebo abre el mundo con ascensor y se cierra en <1 s. La config del ascensor es correcta
  (comparada con el hotel demo, que sí funciona).
- **Causa:** **bug del plugin preinstalado** en la imagen Docker (no es tu mapa).
- **Solución:** recompilar los plugins de `rmf_simulation` desde el fuente para tapar el binario malo:
  ```bash
  cd /opt/rmf_ws/src && git clone https://github.com/open-rmf/rmf_simulation.git
  cd /opt/rmf_ws
  rm -rf build/rmf_building_sim_gz_plugins build/rmf_robot_sim_gz_plugins build/rmf_robot_sim_common \
         install/rmf_building_sim_gz_plugins install/rmf_robot_sim_gz_plugins install/rmf_robot_sim_common log/latest
  colcon build --packages-select rmf_robot_sim_common rmf_robot_sim_gz_plugins rmf_building_sim_gz_plugins
  source install/setup.bash
  echo $GZ_SIM_SYSTEM_PLUGIN_PATH | tr ':' '\n' | grep building_sim   # debe salir /opt/rmf_ws/install/... primero
  ```
- **NO** lo arregla cambiar el motor de físicas (dartsim/bullet).

### 3.7. "No se actualiza el cambio" (artefacto obsoleto)
- **Síntoma:** editas en traffic-editor pero RViz/dashboard siguen mostrando lo viejo.
- **Causa:** regeneraste en `Ejercicios/` pero **no copiaste todos los artefactos** a `src/` (típico: falta
  `nav_graphs/0.yaml`).
- **Solución:** copiar los 3 artefactos (sección 1.3) y **relanzar** (con symlink-install no hace falta
  recompilar). Verificación útil:
  ```bash
  grep -c 'lift: lift1' .../nav_graphs/0.yaml   # 2 = ascensor conectado en ambas plantas
  ```

### 3.8. `No path found for robot ... Unable to find a path` (cruce de plantas)
- **Causa:** el ascensor no está conectado en el grafo: falta un **waypoint dentro de la cabina en una de
  las plantas**. Con un solo vértice de ascensor el grafo queda partido.
- **Solución:** en traffic-editor, añadir un vértice **dentro del footprint del ascensor en CADA planta**
  (mismas x,y), en el `graph_idx` de la flota, con carriles que lo conecten al pasillo. Regenerar nav.
  Comprobar `grep -c 'lift: lift1' 0.yaml` → debe dar 2.

### 3.9. `{"detail":"Fleet not configured to perform this action"}` (limpieza)
- **Causa:** la flota no **declara** la acción `clean`. `task_capabilities.clean: True` NO basta.
- **Solución:** añadir al config de la flota, como hermano de `task_capabilities`:
  ```yaml
  actions: ["clean"]
  ```

### 3.10. `waypoint name for Place [X] cannot be found in the navigation graph` (limpieza)
- **Causa:** `dispatch_clean -cs X` usa `X` como **waypoint del go_to_place** Y como clave de
  `action_paths` a la vez. Si `X` no es un waypoint real, falla.
- **Solución:** las claves de `action_paths.clean.<X>` deben ser **waypoints reales** del nav graph
  (p. ej. `aula-este`, `sala-grabacion`), no nombres inventados (`clean_aula_este` ❌).

### 3.11. Conflicto de puerto del fleet_manager
- **Causa:** dos flotas con el mismo `fleet_manager.port`.
- **Solución:** **puerto único por flota** (tinyRobot 22011, deliveryRobot 22012, cleanRobot 22013).

### 3.12. `Permiso denegado` al copiar `psychology_world/`
- **Causa:** archivos generados por root; el host no puede borrarlos.
- **Solución:** hacer la copia **dentro del contenedor** (donde eres root).

### 3.13. Ruido `Requesting new schedule update because update timed out`
- **No es un error.** Es ruido normal de RMF (el `rmf_traffic_schedule` responde con `Sending remedial
  update`). Ignóralo.

---

## 4. Cómo agregar robots (flotas) a RMF

Cada flota = **1 archivo de config** + **1 bloque `<include>`** en el launch. El `fleet_adapter.launch.xml`
es genérico y se reutiliza; solo cambias `config_file` y `nav_graph_file`.

### 4.1. Crear el config de la flota
`psychology_uv_config/config/psychology_uv_sim/<flota>_config.yaml` (copia el de tinyRobot y adapta):

| Campo | Qué cambiar |
|---|---|
| `rmf_fleet.name` | nombre de la flota (coincide con el adapter) |
| `profile.footprint/vicinity` | tamaño real del robot (DeliveryRobot ~0.5/0.7; tinyRobot 0.3/0.5) |
| `battery_system`/`mechanical_system` | parámetros del modelo |
| `task_capabilities` | `loop`/`delivery` (booleanos) |
| `actions: ["clean"]` | **solo flotas de limpieza** (declara la acción) |
| `robots` | nombres, `charger` y `map_name`/`waypoint` de arranque (deben coincidir con el `building.yaml`) |
| `fleet_manager.port` | **único por flota** (22011, 22012, 22013…) |
| `fleet_manager.action_paths.clean.<zona>` | **solo limpieza**: `map_name`, `path` ([x,y,yaw]) y `finish_waypoint` |

Ejemplo mínimo de declaración de limpieza:
```yaml
rmf_fleet:
  ...
  task_capabilities:
    loop: True
    delivery: False
  actions: ["clean"]
  ...
fleet_manager:
  port: 22013
  action_paths:
    clean:
      aula-este:                  # = waypoint real del nav graph
        map_name: "planta_1"
        path: [ [158.316, -57.539, 0.0], [159.816, -57.539, 1.57], ... ]
        finish_waypoint: "aula-este"
```

### 4.2. Añadir el `<include>` en `psychology_uv_config/launch/psychology_uv_sim.launch.xml`
```xml
<group>
  <include file="$(find-pkg-share psychology_uv_fleet_adapter)/launch/fleet_adapter.launch.xml">
    <arg name="use_sim_time" value="$(var use_sim_time)" />
    <arg name="nav_graph_file"
         value="$(find-pkg-share psychology_uv_maps)/maps/psychology/nav_graphs/0.yaml" />
    <arg name="config_file"
         value="$(find-pkg-share psychology_uv_config)/config/psychology_uv_sim/cleanRobot_config.yaml" />
  </include>
</group>
```

### 4.3. Nav graph de cada flota
- **Simple:** todas las flotas usan `nav_graphs/0.yaml` (si todos los carriles están en `graph_idx 0`).
- **De libro:** cada flota su grafo (`graph_idx 1`, `2`…) dibujando carriles separados en traffic-editor →
  `nav_graphs/1.yaml`, `2.yaml`.

### 4.4. Compilar y verificar
```bash
cd /opt/rmf_ws
colcon build --packages-select psychology_uv_config   # symlink-install: solo si añadiste archivos nuevos
rmfenv
ros2 launch psychology_uv_simulation psychology_uv_sim.launch.xml

# En otra terminal: deben salir las 3 flotas
ros2 topic echo /fleet_states --once
ros2 node list | grep fleet    # 3 fleet_adapter + 3 fleet_manager
```

### 4.5. Recordatorios clave
- Los robots deben existir en el `.world` (vía `spawn_robot_name`/`spawn_robot_type` en el `building.yaml`).
- Que el robot **aparezca** en el dashboard requiere su **fleet adapter** (no basta con que esté en Gazebo).
- El `charger`, `map_name` y `waypoint` del config deben coincidir EXACTOS con el `building.yaml`.
- Para que **suba/baje de planta**, el ascensor debe estar conectado en el grafo de esa flota (ver 3.8).

---

## Resumen del estado del proyecto
- ✅ Planta baja + planta 2, ascensor funcionando (tras recompilar `rmf_simulation`).
- ✅ 3 flotas en RMF: `tinyRobot` (×2), `deliveryRobot` (×2), `cleanRobot` (×2).
- ✅ Tareas validadas: patrulla (cruce de plantas por ascensor) y limpieza.
- ⏳ Pendiente: entrega real (renombrar dispenser/ingestor `coke` → distintos + añadir modelos Teleport),
  y opcionalmente separar grafos por flota (`graph_idx` 1 y 2).
