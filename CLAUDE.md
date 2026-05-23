# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Descripción del proyecto

Workspace de ROS 2 para un sistema de gestión de flotas de robots (RMF - Robotics Middleware Framework) aplicado al edificio de la Facultad de Psicología de la Universitat de València (UV). Utiliza Open-RMF para coordinación de tráfico, asignación de tareas y simulación con Gazebo.

## Comandos de desarrollo

### Compilar todo el workspace
```bash
cd ~/uji/IR2134/rmf_ws
colcon build --symlink-install
source install/setup.bash
```

### Compilar un paquete individual
```bash
colcon build --symlink-install --packages-select <nombre_paquete>
# Ejemplo: colcon build --symlink-install --packages-select psychology_uv_fleet_adapter
```

### Lanzar la simulación completa (Gazebo + RMF)
```bash
source install/setup.bash
ros2 launch psychology_uv_simulation psychology_uv_sim.launch.xml
```

### Lanzar solo la infraestructura RMF (sin simulación)
```bash
ros2 launch psychology_uv_config psychology_uv_sim.launch.xml
```

### Generar mundo desde building.yaml
```bash
ros2 run rmf_building_map_tools building_map_generator gazebo \
  src/psychology_uv/psychology_uv_maps/maps/psychology/psychology.building.yaml \
  src/psychology_uv/psychology_uv_maps/maps/psychology/psychology.world \
  src/psychology_uv/psychology_uv_maps/maps/psychology/psychology_world
```

### Generar grafo de navegación
```bash
ros2 run rmf_building_map_tools building_map_generator nav \
  src/psychology_uv/psychology_uv_maps/maps/psychology/psychology.building.yaml \
  src/psychology_uv/psychology_uv_maps/maps/psychology/nav_graphs
```

### Tests (fleet adapter)
```bash
colcon test --packages-select psychology_uv_fleet_adapter
colcon test-result --verbose
```

### API del fleet manager (documentación interactiva)
Con la simulación corriendo, visitar `http://127.0.0.1:22011/docs` para la documentación FastAPI.

## Arquitectura

El workspace contiene un meta-paquete `psychology_uv` con 5 sub-paquetes ROS 2:

- **psychology_uv_config** (ament_cmake): Launch files principales y configuraciones YAML de flota. Orquesta el lanzamiento de todos los nodos RMF (traffic schedule, task dispatcher, visualización, supervisores de puertas/ascensores) y los fleet adapters. Depende de paquetes externos: `psychology_tasks`, `psychology_panel`.
- **psychology_uv_fleet_adapter** (ament_python): Implementación del fleet adapter y fleet manager basados en REST API (FastAPI/uvicorn). Contiene 3 ejecutables: `fleet_adapter`, `fleet_manager`, `manage_lane`.
- **psychology_uv_maps** (ament_cmake): Contiene el mapa del edificio (`psychology.building.yaml`), mundo de simulación (`.world`), modelos 3D generados, y grafos de navegación.
- **psychology_uv_simulation** (ament_cmake): Launch files para Gazebo (ros_gz_sim), configurando el mundo y el bridge de reloj `/clock`.
- **psychology_uv_assets** (ament_cmake): Modelos SDF de robots (TinyRobot, DeliveryRobot, CleanerBotA/E, Caddy, HospitalRobot) y dispositivos (TeleportDispenser, TeleportIngestor).

### Flujo de datos del fleet adapter

```
RMF Task Dispatcher → Fleet Adapter (rmf_easy_full_control)
                          ↕ REST API (HTTP)
                      Fleet Manager (FastAPI en puerto configurable)
                          ↕ ROS 2 topics
                      Robot simulado en Gazebo (PathRequest, RobotState)
```

El **fleet manager** (`fleet_manager.py`) expone endpoints REST bajo `/open-rmf/psychology_uv_fm/` (status, navigate, stop, start_activity, toggle_teleop, toggle_attach) y se comunica con los robots vía topics ROS 2 (`robot_path_requests`, `robot_state`, `robot_mode_requests`).

El **fleet adapter** (`fleet_adapter.py`) usa `rmf_adapter.easy_full_control` para integrar la flota con el planificador de RMF, consultando al fleet manager vía `RobotClientAPI.py`.

### Configuración de flota

Dos niveles de configuración YAML:
- `psychology_uv_fleet_adapter/config.yaml`: Configuración base de flota (perfil de robot, límites cinemáticos, batería, capacidades de tareas, conexión al fleet manager).
- `psychology_uv_config/config/psychology_uv_sim/tinyRobot_config.yaml`: Configuración por robot individual (posición inicial, charger, frecuencia de actualización), referencia la configuración base de flota.

### Mapa del edificio

El mapa se define en `psychology.building.yaml` (formato `rmf_building_map_tools`). Desde este archivo se generan:
1. El mundo `.world` para Gazebo con los modelos 3D de paredes/suelos.
2. Los grafos de navegación en `nav_graphs/` usados por los fleet adapters para planificación de rutas.

El nivel del mapa se llama `uv_ej` y el mapa de RMF se identifica como `psychology`.
