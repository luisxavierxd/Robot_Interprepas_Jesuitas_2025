# Robot Interprepas Jesuitas 2025

Workspace de ROS 2 para el robot móvil diferencial que se usó en el Interprepas Jesuitas 2025. Hace teleoperación con joystick, mapeo con SLAM, navegación autónoma con Nav2 y seguimiento de pelota. Se puede usar en el robot real o simularlo en Gazebo.

El proyecto parte del tutorial [Articulated Robotics](https://articulatedrobotics.xyz/) de Josh Newans. Lo que se cambió para este robot fueron las dimensiones del chasis, los parámetros de las ruedas y el LIDAR (un LDRobot en lugar del RPLIDAR original).

---

## Estructura del repositorio

Cada carpeta es un **submódulo de git** que apunta a un commit fijo:

| Carpeta | Origen | Para qué sirve |
|---|---|---|
| [`articubot_one`](https://github.com/luisxavierxd/articubot_one) | Fork propio | Paquete principal: descripción URDF del robot, launch files y configuración (ros2_control, SLAM, Nav2, twist_mux, joystick, ball tracker). |
| [`diffdrive_arduino`](https://github.com/joshnewans/diffdrive_arduino) | joshnewans | Plugin de hardware de `ros2_control` (`diffdrive_arduino/DiffDriveArduino`) que se comunica por serial con el Arduino de los motores. |
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
| Chasis (largo × ancho × alto) | 0.335 × 0.265 × 0.138 m |
| Radio de rueda | 0.033 m |
| Separación entre ruedas | 0.297 m |
| LIDAR respecto al chasis | x = 0.122 m, z = 0.212 m |

## Requisitos

- Ubuntu 22.04 + **ROS 2 Humble**
- Paquetes:

```bash
sudo apt install ros-humble-xacro ros-humble-ros2-control ros-humble-ros2-controllers \
  ros-humble-gazebo-ros-pkgs ros-humble-gazebo-ros2-control \
  ros-humble-twist-mux ros-humble-joy ros-humble-teleop-twist-joy \
  ros-humble-slam-toolbox ros-humble-navigation2 ros-humble-nav2-bringup \
  ros-humble-nav2-lifecycle-manager libudev-dev
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

# LIDAR LDRobot
ros2 launch ldlidar_node ldlidar_bringup.launch.py

# Joystick (en el robot o en la laptop)
ros2 launch articubot_one joystick.launch.py
```

En el joystick hay que mantener presionado el **botón 6** (enable) para moverse; el **botón 7** activa el modo turbo. El eje 1 controla el avance y el eje 0 el giro.

### Mapeo (SLAM Toolbox)

```bash
ros2 launch articubot_one online_async_launch.py use_sim_time:=false   # true en simulación
rviz2 -d src/Robot_Interprepas_Jesuitas_2025/articubot_one/config/map.rviz
```

### Navegación (Nav2)

```bash
ros2 launch articubot_one navigation_launch.py use_sim_time:=false
# Con un mapa ya guardado:
ros2 launch articubot_one localization_launch.py map:=<ruta/al/mapa.yaml> use_sim_time:=false
```

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

## Notas y problemas conocidos

- **Frame del LIDAR:** el driver `ldlidar_node` publica por defecto con `frame_id: ldlidar_link`, pero el URDF del robot define el LIDAR como `laser_frame`. Si SLAM o RViz no muestran el scan, cambia `frame_id` a `laser_frame` en `ldrobot-lidar-ros2/ldlidar_node/params/ldlidar.yaml`.
- **Puerto del LIDAR:** el driver LDRobot busca `/dev/ldlidar`. Para crear ese alias instala la regla udev de `ldrobot-lidar-ros2/rules/` o cambia `serial_port` en el mismo YAML.
- `launch/rplidar.launch.py` sigue configurado para el RPLIDAR original. Con el LDRobot usa `ldlidar_bringup.launch.py`.
- Si cambias de motores o ruedas, ajusta `enc_counts_per_rev` en `description/ros2_control.xacro` y `wheel_radius` / `wheel_separation` en `config/my_controllers.yaml`.

## Créditos

- [Articulated Robotics](https://github.com/joshnewans), de Josh Newans: `articubot_one`, `diffdrive_arduino`, `serial_motor_demo` y `joy_tester`.
- [Myzhar](https://github.com/Myzhar/ldrobot-lidar-ros2): driver de los LIDAR LDRobot.

## Licencia

[Apache 2.0](LICENSE). Cada submódulo conserva su propia licencia.
