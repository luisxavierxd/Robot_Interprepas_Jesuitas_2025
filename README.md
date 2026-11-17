# Robot Interprepas Jesuitas 2025

Workspace de ROS 2 para el robot móvil diferencial que se usó en el Interprepas Jesuitas 2025. Hace teleoperación con joystick, mapeo con SLAM, navegación autónoma con Nav2 y recorre una ruta preprogramada de waypoints para recolectar objetos. Se puede usar en el robot real o simularlo en Gazebo.

El proyecto parte del tutorial [Articulated Robotics](https://articulatedrobotics.xyz/) de Josh Newans. Lo que se cambió para este robot fueron las dimensiones del chasis, los parámetros de las ruedas y el LIDAR (un LDRobot en lugar del RPLIDAR original).

---

## Estructura del repositorio

Cada carpeta es un **submódulo de git** que apunta a un commit fijo:

| Carpeta | Origen | Para qué sirve |
|---|---|---|
| [`articubot_one`](https://github.com/luisxavierxd/articubot_one) | Fork propio | Paquete principal: descripción URDF del robot, launch files y configuración (ros2_control, SLAM, Nav2, twist_mux, joystick, ball tracker). |
| [`diffdrive_arduino`](https://github.com/luisxavierxd/diffdrive_arduino) | Fork propio (adaptado a Humble) | Plugin de hardware de `ros2_control` (`diffdrive_arduino/DiffDriveArduino`) que se comunica por serial con el Arduino de los motores. |
| [`serial`](https://github.com/joshnewans/serial) | joshnewans | Librería serial (C++) que usa `diffdrive_arduino`. |
| [`serial_motor_demo`](https://github.com/joshnewans/serial_motor_demo) | joshnewans | GUI y driver para probar los motores por serial sin `ros2_control`. |
| [`joy_tester`](https://github.com/joshnewans/joy_tester) | joshnewans | GUI para ver ejes y botones del joystick. |
| [`ldrobot-lidar-ros2`](https://github.com/Myzhar/ldrobot-lidar-ros2) | Myzhar | Driver ROS 2 para los LIDAR LDRobot (LD06 / LD19 / STL27L). |

## Hardware

- **Computadora:** Raspberry Pi 4 (la ruta serial del LIDAR en `rplidar.launch.py` es la del bus PCIe de la Pi 4).
- **Control de motores:** Arduino con firmware [ROSArduinoBridge](https://github.com/joshnewans/ros_arduino_bridge) en `/dev/ttyUSB0` a 57600 baud.
- **Motores:** 2 motores DC con encoder (3436 cuentas por vuelta), tracción diferencial con una rueda loca.
- **LIDAR:** LDRobot (antes RPLIDAR).
- **Joystick:** cualquier gamepad que reconozca `joy`.

### Parámetros físicos (`articubot_one/description/robot_core.xacro`, `config/my_controllers.yaml`)

| Parámetro | Valor |
|---|---|
| Chasis (largo × ancho × alto) | 0.146 × 0.156 × 0.145 m |
| Radio de rueda | 0.0286 m |
| Separación entre ruedas | 0.156 m |
| LIDAR respecto al chasis | x = 0.122 m, z = 0.212 m |

## Requisitos

- Ubuntu 22.04 + **ROS 2 Humble**
- Las dependencias están declaradas en `articubot_one/package.xml`, así que `rosdep` las instala todas (paso de abajo).

### Desarrollo en Windows (WSL + Docker)

El proyecto necesita Ubuntu 22.04 + Humble. Si tu WSL trae otra versión (por ejemplo Ubuntu 24.04 / Jazzy), puedes compilar dentro de un contenedor `ros:humble`:

```bash
docker run -it --name humble_ws -v "$PWD:/ws/src/repo" ros:humble bash
# dentro del contenedor:
apt-get update && rosdep update
cd /ws && rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install && source install/setup.bash
```

## Instalación

```bash
mkdir -p ~/robot_ws/src && cd ~/robot_ws/src
git clone --recurse-submodules https://github.com/luisxavierxd/Robot_Interprepas_Jesuitas_2025.git
cd ~/robot_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
source install/setup.bash
```

Si ya clonaste el repo sin `--recurse-submodules`, corre:

```bash
git submodule update --init --recursive
```

## Uso

Todos los comandos asumen que ya hiciste `source install/setup.bash`.

### Simulación (Gazebo)

```bash
ros2 launch articubot_one launch_sim.launch.py world:=./src/Robot_Interprepas_Jesuitas_2025/articubot_one/worlds/obstacles.world
```

### Robot real

```bash
# En la Raspberry Pi: robot_state_publisher, ros2_control (diff_cont + joint_broad) y twist_mux
ros2 launch articubot_one launch_robot.launch.py

# LIDAR LDRobot: publica /scan en el frame laser_frame y se activa solo (lifecycle)
ros2 launch articubot_one ldlidar.launch.py            # serial_port:=/dev/ttyUSB1 si no usas la regla udev

# Joystick (en el robot o en la laptop)
ros2 launch articubot_one joystick.launch.py
```

En el joystick hay que mantener presionado el **botón 6** (enable) para moverse; el **botón 7** activa el modo turbo. El eje 1 controla el avance y el eje 0 el giro.

### Mapeo (SLAM Toolbox)

```bash
ros2 launch articubot_one online_async_launch.py use_sim_time:=false   # true en simulación
rviz2 -d src/Robot_Interprepas_Jesuitas_2025/articubot_one/config/map.rviz
# Guardar el mapa para Nav2:
ros2 run nav2_map_server map_saver_cli -f src/Robot_Interprepas_Jesuitas_2025/articubot_one/maps/map
```

### Ruta autónoma (recolección)

```bash
# 1. Mapa + localización (map_server + AMCL). Por defecto carga articubot_one/maps/map.yaml
ros2 launch articubot_one localization_launch.py
# 2. Nav2 (planner, controller, behavior_server, bt_navigator, waypoint_follower)
ros2 launch articubot_one navigation_launch.py
# 3. Dar la pose inicial con "2D Pose Estimate" en RViz y lanzar la ruta
ros2 run articubot_one follow_waypoints.py
```

`follow_waypoints.py` lee `config/waypoints.yaml` (54 poses en el marco `map`) y se lo manda a Nav2 con la acción `FollowWaypoints`. En cada punto se detiene 1.5 s (`waypoint_pause_duration` en `nav2_params.yaml`). Parámetros:

| Parámetro | Default | Descripción |
|---|---|---|
| `waypoints_file` | `config/waypoints.yaml` | Archivo de la ruta |
| `set_initial_pose` | `false` | Usa el primer waypoint como pose inicial de AMCL (si el robot arranca ahí) |
| `localizer` | `amcl` | Usa `none` si la localización viene de slam_toolbox |
| `loop` | `false` | Repite la ruta sin parar |

```bash
ros2 run articubot_one follow_waypoints.py --ros-args -p set_initial_pose:=true
```

Para solo ver el mapa guardado: `ros2 launch articubot_one map_simple.py`.

La velocidad máxima de Nav2 está limitada a 0.13 m/s lineal y 0.5 rad/s angular (`FollowPath` en `nav2_params.yaml`).

### Seguimiento de pelota

Hace falta el paquete [`ball_tracker`](https://github.com/joshnewans/ball_tracker) y una cámara (`camera.launch.py`).

```bash
ros2 launch articubot_one ball_tracker.launch.py sim_mode:=false
```

### Probar hardware por separado

```bash
ros2 run serial_motor_demo driver --ros-args -p serial_port:=/dev/ttyUSB0 -p baud_rate:=57600 -p loop_rate:=30 -p encoder_cpr:=3436
ros2 run serial_motor_demo gui
ros2 run joy_tester test_joy
```

## Prioridad de comandos de velocidad (`twist_mux`)

`twist_mux` recibe varias fuentes de `cmd_vel` y manda la de mayor prioridad a `/diff_cont/cmd_vel_unstamped`:

| Fuente | Tópico | Prioridad |
|---|---|---|
| Joystick | `cmd_vel_joy` | 100 (máxima) |
| Ball tracker | `cmd_vel_tracker` | 20 |
| Nav2 | `cmd_vel` | 10 |

El joystick siempre tiene prioridad sobre la navegación autónoma, así que sirve para tomar el control en cualquier momento.

## Notas

- **Puertos seriales:** el Arduino (`/dev/ttyUSB0`) y el LIDAR son adaptadores USB-serial, y el orden en que aparecen como `ttyUSB*` puede cambiar al reiniciar. La regla `ldrobot-lidar-ros2/rules/ldlidar.rules` crea `/dev/ldlidar` para chips CP210x (`sudo cp ldrobot-lidar-ros2/rules/ldlidar.rules /etc/udev/rules.d/ && sudo udevadm control --reload && sudo udevadm trigger`). Si el Arduino también usa CP210x, mejor usa rutas `/dev/serial/by-path/...` para los dos.
- **Mapa:** `maps/map.pgm` se dibujó a mano (GIMP): 400 × 200 px a 0.0061 m/px, o sea 2.44 × 1.22 m. Su `origin` es la esquina inferior izquierda de la cancha `[0, 0, 0]`. **No** pongas ahí la pose de salida del robot: esa se da con `2D Pose Estimate` o con `set_initial_pose:=true`. Los waypoints (x 0.15–2.29, y 0.10–1.12) están en ese marco.
- **Costmaps:** `robot_radius` es 0.08 m (la media anchura del chasis) y `inflation_radius` 0.12 m. Varios waypoints quedan a ~8–9 cm de una pared, y con un radio mayor Nav2 no puede planear hacia ellos. Con estos valores se planean los 53 tramos de la ruta.
- `launch/rplidar.launch.py` es para el RPLIDAR original. Con el LDRobot usa `ldlidar.launch.py`.
- Si cambias de motores o ruedas, ajusta `enc_counts_per_rev` en `description/ros2_control.xacro` y mantén `wheel_radius` / `wheel_separation` iguales en `robot_core.xacro` y `my_controllers.yaml`.

## Créditos

- [Articulated Robotics](https://github.com/joshnewans), de Josh Newans: `articubot_one`, `diffdrive_arduino`, `serial_motor_demo` y `joy_tester`.
- [Myzhar](https://github.com/Myzhar/ldrobot-lidar-ros2): driver de los LIDAR LDRobot.

## Licencia

[Apache 2.0](LICENSE). Cada submódulo conserva su propia licencia.
