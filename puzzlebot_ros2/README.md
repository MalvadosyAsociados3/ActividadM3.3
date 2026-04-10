# puzzlebot_ros2 — TE3003B Módulo 3 · Actividad M3.3

ROS 2 Humble workspace for autonomous SLAM mapping and Nav2 navigation of the
**Puzzlebot** (differential-drive, 360° LiDAR) inside a Gazebo Classic maze.

---

## Integrantes

- Luis Adrián Uribe Cruz (LaucoTec)
- Grant Nathaniel Keegan (gnkeegan)
- Diego Gerardo Sánchez Moreno (DiegoSanMo6011)
- Héctor Gúmaro Guzmán Reyes (HectorGuzm4n)

---

## Packages

| Package | Purpose |
|---|---|
| `puzzlebot_description` | URDF/Xacro model, meshes, RViz description config |
| `puzzlebot_gazebo` | Gazebo Classic world, spawn launch |
| `puzzlebot_navigation2` | SLAM Toolbox + Nav2 configs, maps, launch files |

---

## Estructura del workspace

```
puzzlebot_ws/
└── src/
    └── puzzlebot_ros2/
        ├── puzzlebot_description/
        │   ├── launch/
        │   ├── meshes/
        │   ├── rviz/
        │   └── urdf/
        ├── puzzlebot_gazebo/
        │   ├── config/
        │   ├── launch/
        │   └── worlds/
        ├── puzzlebot_navigation2/
        │   ├── config/
        │   ├── launch/
        │   ├── maps/
        │   └── rviz/
        └── README.md
```

---

## Tutorial paso a paso para correr el proyecto

### Prerequisitos — Instalar dependencias

Asegúrate de tener ROS 2 Humble instalado y luego ejecuta:

```bash
sudo apt install ros-humble-slam-toolbox \
  ros-humble-nav2-bringup \
  ros-humble-navigation2 \
  ros-humble-teleop-twist-keyboard \
  ros-humble-gazebo-ros-pkgs \
  ros-humble-gazebo-ros2-control
```

---

### Paso 1 — Clonar y compilar el workspace

```bash
# Clonar el repositorio
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src
git clone https://github.com/MalvadosyAsociados3/Actividad-M3.3.git puzzlebot_ros2

# Compilar
cd ~/ros2_ws
colcon build --symlink-install
source install/setup.bash
```

> **Tip:** Añade `source ~/ros2_ws/install/setup.bash` a tu `~/.bashrc` para no tener que hacerlo en cada terminal nueva:
> ```bash
> echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
> ```

---

### Paso 2 — Verificar el modelo del robot (opcional)

Para ver el modelo 3D del Puzzlebot en RViz sin necesidad de simulación:

```bash
source ~/ros2_ws/install/setup.bash
ros2 launch puzzlebot_description puzzlebot_description.launch.py
```

Deberías ver el robot con sus ruedas, LiDAR y todos los frames TF correctos en RViz.

---

### Opción A — Generar un nuevo mapa con SLAM

Usa esta opción si quieres mapear el entorno desde cero.

**Terminal 1 — Lanzar simulación + SLAM Toolbox + RViz:**

```bash
source ~/ros2_ws/install/setup.bash
ros2 launch puzzlebot_navigation2 slam.launch.py
```

Esto abre Gazebo con el laberinto, el Puzzlebot y RViz mostrando el mapa construyéndose en tiempo real.

**Terminal 2 — Teleoperar el robot para explorar el laberinto:**

```bash
source ~/ros2_ws/install/setup.bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

Controles del teclado:

| Tecla | Acción |
|---|---|
| `i` | Avanzar |
| `,` | Retroceder |
| `j` | Girar izquierda |
| `l` | Girar derecha |
| `k` | Detener |
| `q` / `z` | Aumentar / reducir velocidad |

Buenas prácticas durante el mapeo:
- Muévete **lento y constante** para que el LiDAR registre bien
- Cubre **todas las zonas** del laberinto sin dejar huecos
- Evita **giros bruscos** (generan errores de odometría)
- Vuelve al **punto de inicio** al terminar para activar el loop closure

**Terminal 3 — Guardar el mapa cuando estés satisfecho:**

```bash
source ~/ros2_ws/install/setup.bash
ros2 run nav2_map_server map_saver_cli -f ~/ros2_ws/src/puzzlebot_ros2/puzzlebot_navigation2/maps/map_maze
```

Esto genera dos archivos:
- `map_maze.pgm` — imagen del mapa (grilla de ocupación)
- `map_maze.yaml` — metadatos del mapa (resolución, origen, umbrales)

---

### Opción B — Navegación autónoma con el mapa ya existente

El proyecto ya incluye un mapa generado. Solo necesitas una terminal:

```bash
source ~/ros2_ws/install/setup.bash
ros2 launch puzzlebot_navigation2 nav2.launch.py
```

Esto lanza Gazebo + Nav2 completo + RViz. Una vez abierto, en RViz:

1. Haz clic en **"2D Pose Estimate"** y marca la posición inicial aproximada del robot sobre el mapa
2. Espera a que la **nube de partículas AMCL** converja alrededor del robot
3. Haz clic en **"Nav2 Goal"** (o **"2D Goal Pose"**) y marca el destino deseado
4. El robot calcula la ruta y navega solo evitando obstáculos

Si quieres usar un mapa personalizado distinto al que viene por defecto:

```bash
ros2 launch puzzlebot_navigation2 nav2.launch.py map:=/ruta/a/tu/mapa.yaml
```

---

## TF Tree

```
map
 └── odom                  ← AMCL (navegación) / SLAM Toolbox (mapeo)
      └── base_footprint   ← libgazebo_ros_diff_drive.so
           └── base_link   ← robot_state_publisher (joint fijo en URDF)
                ├── wheel_r_link
                ├── wheel_l_link
                ├── caster_link
                └── lidar_link
```

---

## Topics principales

| Topic | Tipo | Publisher |
|---|---|---|
| `/scan` | `sensor_msgs/LaserScan` | Plugin LiDAR de Gazebo |
| `/odom` | `nav_msgs/Odometry` | Plugin diff-drive de Gazebo |
| `/cmd_vel` | `geometry_msgs/Twist` | Nav2 controller / teleop |
| `/map` | `nav_msgs/OccupancyGrid` | SLAM Toolbox / map_server |
| `/tf` | `tf2_msgs/TFMessage` | Múltiples nodos |

---

## Especificaciones del robot

| Parámetro | Valor |
|---|---|
| Huella (footprint) | ~0.18 m × 0.15 m |
| Separación de ruedas | 0.19 m |
| Diámetro de ruedas | 0.10 m |
| Velocidad lineal máxima | 0.22 m/s |
| Velocidad angular máxima | 1.0 rad/s |
| Rango LiDAR | 0.12 – 5.0 m (360°) |

---

## Solución de problemas comunes

| Problema | Solución |
|---|---|
| `Package not found` | Ejecuta `source ~/ros2_ws/install/setup.bash` |
| Gazebo no abre | Ejecuta `killall gzserver gzclient` y reintenta |
| El mapa no aparece en RViz | Verifica que el Fixed Frame sea `map` |
| El robot no responde al teleop | Confirma que el topic sea `/cmd_vel` |
| SLAM muy lento | Reduce `map_update_interval` en `config/slam_toolbox.yaml` |
| AMCL no converge | Ajusta mejor la pose inicial con "2D Pose Estimate" |
| Robot choca con paredes | Ajusta `inflation_radius` en `config/nav2_params.yaml` |
