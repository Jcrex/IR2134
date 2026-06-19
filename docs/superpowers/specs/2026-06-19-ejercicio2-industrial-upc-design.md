# Diseño — Ejercicio 2: Escola Eng Tèc Industrial UPC (Open-RMF)

Fecha: 2026-06-19

## Objetivo

Dejar preparado **todo lo ajeno a traffic-editor** para el ejercicio 2 (edificio Escola
Eng Tèc Industrial UPC, 3 plantas), replicando la estructura del ejercicio 1
(`psychology_uv`). Cuando el usuario termine de dibujar el mapa en traffic-editor, solo
deberá regenerar `.world`/`nav_graphs` y copiarlos al paquete de mapas.

## Reparto de responsabilidades

- **Usuario (traffic-editor):** `industrial.building.yaml` → de él se generan, por
  comando, `industrial.world` y `nav_graphs/`.
- **Claude (este trabajo):** estructura de paquetes ROS, launch files, configs de flota,
  código del fleet adapter, rviz, carpeta de trabajo y planos colocados.

## Estructura nueva de paquetes: `src/industrial_upc/`

```
industrial_upc/
  industrial_upc_maps/          (ament_cmake)  → maps/industrial/ (planos copiados; building/world/nav los pone el usuario)
  industrial_upc_config/        (ament_cmake)  → launch + 3 configs de flota + rviz
  industrial_upc_simulation/    (ament_cmake)  → launch de Gazebo
  industrial_upc_fleet_adapter/ (ament_python) → fleet_adapter / fleet_manager / manage_lane (copia genérica)
```

Se **reutiliza** `psychology_uv_assets` (modelos de robots); no se duplica.

## Convenciones

- Prefijo de paquetes: `industrial_upc_*`.
- `map_name` RMF (arg de simulación): `industrial`.
- Carpeta de modelos del mundo generada: `industrial_world`.
- Niveles (placeholders; los fija el usuario en traffic-editor): `soterrani_1`,
  `soterrani_2`, `baixa` (según planos `Plànols-20260426/`).
- `initial_map` por defecto: `baixa`.
- Puertos de fleet_manager: tinyRobot 22011, deliveryRobot 22012, cleanRobot 22013.

## Flotas (las 3, como ejercicio 1)

- `tinyRobot` (patrulla), `deliveryRobot` (entrega), `cleanRobot` (limpieza con
  `actions: ["clean"]` + `action_paths.clean`).
- Los campos dependientes del mapa (`map_name`, `waypoint` de charger/arranque, coords de
  `action_paths`) quedan con placeholders comentados `# AJUSTAR tras el mapa`.

## Carpeta de trabajo `Ejercicios/ejercicio2/`

- Los planos ya están (`Plànols-20260426/`). Es donde se creará el
  `industrial.building.yaml` y se regenerarán world/nav antes de copiarlos a `src/`.
- Se añade un `INFORME-avances-RMF.md` adaptado con los comandos ya apuntando a
  `industrial_upc` / `industrial`.

## Fuera de alcance (depende del mapa del usuario)

`industrial.building.yaml`, `industrial.world`, `nav_graphs/*.yaml`: se generan desde el
mapa. Se dejan los huecos y los comandos exactos documentados.

## Proceso de trabajo

Commit por cada avance (preferencia del usuario): un commit por paquete/unidad coherente.
