# maps/industrial

Artefactos del mapa del edificio Escola Eng Tèc Industrial UPC.

## Contenido

- `Plànols-20260426/` — planos PNG de las plantas (referenciados por el `building.yaml`).
- **(pendiente)** `industrial.building.yaml` — lo genera el usuario en traffic-editor.
- **(pendiente)** `industrial.world` — generado desde el building.yaml.
- **(pendiente)** `industrial_world/` — modelos 3D del mundo (generados).
- **(pendiente)** `nav_graphs/` — grafos de navegación (generados).

## Cómo rellenar los pendientes

Desde `Ejercicios/ejercicio2` (donde está el building.yaml en edición), dentro del contenedor:

```bash
# 1) Mundo de Gazebo
ros2 run rmf_building_map_tools building_map_generator gazebo \
  industrial.building.yaml industrial.world industrial_world

# 2) Grafos de navegación
ros2 run rmf_building_map_tools building_map_generator nav \
  industrial.building.yaml nav_graphs

# 3) Copiar los 4 artefactos a este paquete
DEST=/opt/rmf_ws/src/industrial_upc/industrial_upc_maps/maps/industrial
cp industrial.building.yaml $DEST/
cp industrial.world          $DEST/
rm -rf $DEST/industrial_world && cp -r industrial_world $DEST/
rm -rf $DEST/nav_graphs       && cp -r nav_graphs       $DEST/
```

Ver `Ejercicios/ejercicio2/INFORME-avances-RMF.md` para el flujo completo.
