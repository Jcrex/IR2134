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
- ✅ Configs de las 3 flotas con placeholders comentados `# AJUSTAR tras el mapa`.
- ⏳ `industrial.building.yaml` en edición en traffic-editor (de momento un nivel `L0` → `planta_baixa.png`).
- ⏳ Pendiente generar `industrial.world` y `nav_graphs/` y copiarlos a `industrial_upc_maps`.
- ⏳ Pendiente: ajustar en los configs `map_name`/`waypoint`/`action_paths` a los nombres reales del mapa.

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

## 3. Recordatorios clave (ver detalle en el informe del ejercicio 1)

- Plantas apiladas → dar `elevation` positiva distinta a cada nivel en el building.yaml.
- Ascensor entre plantas → vértice dentro del footprint del ascensor en CADA planta + carriles.
  Comprobar `grep -c 'lift: lift1' nav_graphs/0.yaml` → debe dar 2.
- Limpieza → `actions: ["clean"]` (no basta `task_capabilities.clean`).
- Puertas/robots con nombre único; `spawn_robot_type` con el nombre EXACTO del modelo de assets.
- Segfault de `liblift.so` en imagen jazzy → recompilar `rmf_simulation` desde fuente (ver ej.1 §3.6).
