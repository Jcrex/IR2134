# Informe de avances — Open-RMF Escola Eng Tèc Industrial UPC (Ejercicio 2)

Réplica del flujo del ejercicio 1 para el nuevo edificio (3 plantas). El catálogo completo de
errores y soluciones está en `../ejercicio1/INFORME-avances-RMF.md` (sigue siendo válido: las
mismas causas y arreglos aplican aquí). Este documento recoge las **rutas/nombres propios** del
ejercicio 2.

> **Convención de rutas:** dentro del contenedor el workspace es `/opt/rmf_ws`
> (= host `~/uji/IR2134/rmf_ws`). El ejercicio está en `/opt/rmf_ws/Ejercicios/ejercicio2`.
> En cada terminal: `rmfenv`.

## Nombres del ejercicio 2

| Concepto | Valor |
|---|---|
| Edificio | Escola Eng Tèc Industrial UPC |
| `name` del building.yaml | `industrial` |
| Meta-paquete | `src/industrial_upc/` |
| Paquetes | `industrial_upc_maps`, `industrial_upc_config`, `industrial_upc_simulation`, `industrial_upc_fleet_adapter` |
| Assets de robots | se **reutiliza** `psychology_uv_assets` (no se duplica) |
| `map_name` (simulación) | `industrial` |
| Carpeta de modelos del mundo | `industrial_world` |
| Niveles previstos | `soterrani_1`, `soterrani_2`, `baixa` |
| `initial_map` | `baixa` |
| Planos | `Plànols-20260426/{planta_soterrani_1,planta_soterrani_2,planta_baixa}.png` |
| Puertos fleet_manager | tinyRobot 22011 · deliveryRobot 22012 · cleanRobot 22013 |

## Estado actual

- ✅ Paquetes ROS preparados (maps, config, simulation, fleet_adapter) y planos colocados.
- ✅ Planta baixa (`baixa`) dibujada en traffic-editor; world + nav_graphs generados y copiados.
- ✅ Flota `tinyRobot` (×2) operativa: ambos en `/fleet_states` y patrulla validada.
- ✅ Flotas `deliveryRobot` y `cleanRobot` desactivadas en el launch (sin robots en planta baixa, ver 3.2).
- ⏳ Pendiente: nombrar waypoints destino en traffic-editor (ahora solo los 2 chargers tienen nombre).
- ⏳ Pendiente: dar nombre válido a las puertas `null`/`l` y regenerar (ver 3.4 prevención).
- ⏳ Pendiente: resto de plantas (`soterrani_1`, `soterrani_2`) + ascensor, y flotas delivery/clean.

## 1. Tras terminar el mapa en traffic-editor

### 1.1. Regenerar mundo y grafos (dentro de `Ejercicios/ejercicio2`)
```bash
cd /opt/rmf_ws/Ejercicios/ejercicio2

# 1) Mundo de Gazebo
ros2 run rmf_building_map_tools building_map_generator gazebo \
  industrial.building.yaml industrial.world industrial_world

# 2) Grafos de navegación
ros2 run rmf_building_map_tools building_map_generator nav \
  industrial.building.yaml nav_graphs

# 3) (opcional) Previsualizar en Gazebo antes de la sim completa
ros2 run rmf_building_map_tools building_map_model_downloader industrial.building.yaml -e ./models
export GZ_SIM_RESOURCE_PATH=`pwd`/industrial_world:`pwd`/models:/rmf_demos_ws/install/rmf_demos_assets/share/rmf_demos_assets/models
gz sim -r -v 4 industrial.world
```

### 1.2. Copiar los artefactos al paquete de mapas (DENTRO del contenedor, como root)
```bash
DEST=/opt/rmf_ws/src/industrial_upc/industrial_upc_maps/maps/industrial
SRC=/opt/rmf_ws/Ejercicios/ejercicio2

cp $SRC/industrial.building.yaml $DEST/
cp $SRC/industrial.world          $DEST/
rm -rf $DEST/industrial_world && cp -r $SRC/industrial_world $DEST/
rm -rf $DEST/nav_graphs       && cp -r $SRC/nav_graphs       $DEST/
```
> ⚠️ Copia SIEMPRE los 4 artefactos (building.yaml, world, industrial_world/, nav_graphs/).
> El fallo más típico es regenerar el nav graph y copiar solo el building.yaml → grafo viejo.
> Los planos `Plànols-20260426/` ya están en `$DEST` (no hace falta recopiarlos).

### 1.3. Ajustar los configs de flota a los nombres reales del mapa
En `src/industrial_upc/industrial_upc_config/config/industrial_upc_sim/*.yaml`, sustituir todos
los `# AJUSTAR tras el mapa`:
- `start.map_name` → nivel real (`soterrani_1` / `soterrani_2` / `baixa`).
- `charger` / `start.waypoint` → waypoint real con `is_charger`.
- (cleanRobot) `action_paths.clean.<zona>` → la clave debe ser un **waypoint real** del nav graph.

### 1.4. Compilar y lanzar
```bash
cd /opt/rmf_ws
colcon build --symlink-install --packages-select \
  industrial_upc_maps industrial_upc_config industrial_upc_simulation industrial_upc_fleet_adapter
rmfenv
ros2 launch industrial_upc_simulation industrial_upc_sim.launch.xml

# Verificar (otra terminal): deben salir las 3 flotas
ros2 topic echo /fleet_states --once
ros2 node list | grep fleet    # 3 fleet_adapter + 3 fleet_manager
```
> Con `--symlink-install`, tras la 1ª compilación basta **relanzar** al sobrescribir archivos
> existentes; solo recompila si **añades archivos nuevos**.

## 2. Comandos de tareas (siempre con `--use_sim_time`)

```bash
# Patrulla
ros2 run rmf_demos_tasks dispatch_patrol -p <waypoint> -n 1 --use_sim_time
# A un robot concreto
ros2 run rmf_demos_tasks dispatch_patrol -F tinyRobot -R tinyRobot1 -p <waypoint> -n 1 --use_sim_time
# Limpieza (la zona = waypoint real y clave de action_paths.clean)
ros2 run rmf_demos_tasks dispatch_clean -cs <zona> --use_sim_time
# Entrega
ros2 run rmf_demos_tasks dispatch_delivery -p <pickup_wp> -ph <dispenser> -d <dropoff_wp> -dh <ingestor> --use_sim_time
# Diagnóstico
ros2 topic echo /task_api_responses
```

## 3. Errores encontrados en el ejercicio 2

### 3.1. `Cannot find a waypoint named [X_charger] ... We will not add the robot to the fleet`
- **Síntoma:** los robots aparecen en Gazebo pero NO en `/fleet_states` y no se mueven. El adapter
  repite "Cannot find a waypoint named [tinyRobot1_charger] in the navigation graph".
- **Causa:** desajuste de nombres entre el `charger`/`waypoint` del config y el nombre real del
  vértice en el nav graph. En el mapa los chargers se llamaban `tinyRobot_charger1`/`2` pero el
  config pedía `tinyRobot1_charger`/`2` (número en otra posición). RMF busca el waypoint por nombre
  EXACTO.
- **Solución:** alinear ambos. O renombrar el vértice en traffic-editor, o (lo que hicimos) poner en
  `tinyRobot_config.yaml` el nombre real: `charger`/`waypoint` = `tinyRobot_charger1` / `tinyRobot_charger2`.
  Con `--symlink-install` basta relanzar; no hace falta recompilar.

### 3.2. `Other error for <robot> in get_data: 'robot_name'` (en bucle)
- **Causa:** el launch arranca flotas (deliveryRobot, cleanRobot) cuyos robots NO existen en el mapa.
  El fleet manager no encuentra su estado y escupe el error sin parar.
- **Solución:** comentar en `industrial_upc_sim.launch.xml` los `<group>` de las flotas sin robots.
  Reactivarlas cuando el building.yaml incluya esos robots.

### 3.3. `Connection to ws://localhost:8000/_internal failed` (en bucle)
- **Causa:** se pasó `server_uri:="ws://localhost:8000/_internal"` pero no hay api server escuchando
  (o se cayó, ver 3.4). La simulación funciona igual sin el dashboard web.
- **Solución:** lanzar sin `server_uri`, o arrancar el api server / panel.

### 3.4. ⭐ El api server cae al arrancar: `ValidationError ... for DoorState ... Application startup failed`
- **Síntoma:** el api server, aunque "está lanzado de base", aborta el arranque tras
  `loading states from database...` con un `pydantic ValidationError` sobre un `DoorState`
  (p. ej. `puerta24`) al que le faltan `door_time`/`door_name`/`current_mode`. Como consecuencia,
  los adapters no pueden conectar (ver 3.3) y aparecen errores `Event loop is closed`.
- **Causa:** el api server persiste el estado de RMF en SQLite (`rmf_api_config.py` →
  `sqlite://<dir>/run/db.sqlite3`). En el arranque carga y valida TODOS los estados guardados; un
  registro viejo/incompleto (heredado de otro edificio, p. ej. `puerta24` del ejercicio 1) ya no
  pasa la validación del esquema actual y aborta. Suele originarse por puertas mal nombradas.
- **Solución:** resetear la BD (solo guarda historial de runtime; es seguro borrarla). En el contenedor:
  ```bash
  # parar el api server (Ctrl+C) y:
  rm -f /opt/rmf_ws/run/db.sqlite3
  rm -rf /opt/rmf_ws/run/cache
  # relanzar; se recrea vacía
  ```
- **Prevención:** dar nombre único y válido a TODAS las puertas en traffic-editor (evitar puertas
  `null` o de un carácter como `l`), regenerar y recopiar.

## 4. Recordatorios clave (ver detalle en el informe del ejercicio 1)

- El `charger`/`waypoint` del config debe coincidir EXACTO con el nombre del vértice en el nav graph (3.1).
- Plantas apiladas → dar `elevation` positiva distinta a cada nivel en el building.yaml.
- Ascensor entre plantas → vértice dentro del footprint del ascensor en CADA planta + carriles.
  Comprobar `grep -c 'lift: lift1' nav_graphs/0.yaml` → debe dar 2.
- Limpieza → `actions: ["clean"]` (no basta `task_capabilities.clean`).
- Puertas/robots con nombre único; `spawn_robot_type` con el nombre EXACTO del modelo de assets.
- Segfault de `liblift.so` en imagen jazzy → recompilar `rmf_simulation` desde fuente (ver ej.1 §3.6).
