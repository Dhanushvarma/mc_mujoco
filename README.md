# mujoco interface for mc-rtc

![Screenshot from 2022-08-12 17-09-16](https://user-images.githubusercontent.com/16384313/188832982-1a1263e4-ce33-4dc7-804a-589d151670b3.png)

## Usage

First, install the required apt packages:

```sh
$ sudo apt install libxrandr-dev libxinerama-dev libxcursor-dev libxi-dev libglew-dev
```

Then, execute the following command to install mujoco (the version in [`MUJOCO_VERSION`](MUJOCO_VERSION), downloaded to `$HOME/.mujoco/mujoco-<version>`), and build `mc_mujoco`:

```sh
$ git clone --recursive git@github.com:rohanpsingh/mc_mujoco.git
$ cd mc_mujoco
$ mkdir build && cd build
$ cmake .. -DCMAKE_BUILD_TYPE=RelWithDebInfo
$ make
$ make install
```

If mujoco is already installed, you can add the following   option to specify the path :

```sh
$ cmake .. -DCMAKE_BUILD_TYPE=RelWithDebInfo -DMUJOCO_ROOT_DIR=$HOME/.mujoco/mujoco-3.15.0
```

mc_mujoco reads its user configuration (`mc_mujoco.yaml`, robot files) from `~/.config/mc_rtc/mc_mujoco` by default. Pass `-DMC_MUJOCO_USER_DESTINATION=<folder>` to use another folder.

Add the following line to your `~/.bashrc`:

```
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${HOME}/.mujoco/mujoco-3.15.0/lib:${HOME}/.mujoco/mujoco-3.15.0/bin
```

Then, to run the interface:

```sh
$ mc_mujoco
```

## Docker

Assuming [Docker](https://docs.docker.com/engine/install/) (Compose v2) has been installed:

```sh
$ git clone --recursive git@github.com:rohanpsingh/mc_mujoco.git
$ cd mc_mujoco/docker
$ make run
```

This builds the environment image, compiles mc_mujoco, and drops you into a shell. Then:

```sh
$ mc_mujoco
```

See [docker/README.md](docker/README.md) for details on development workflow, using the image for your own mc_rtc controllers, running CI locally, and version management.

## Examples

Example controllers live under [`examples/`](examples/). Not built by default (enable with `-DBUILD_EXAMPLES=ON`). See [`examples/README.md`](examples/README.md) for the convention and how to activate a sample's config inside the container.

- **[neck_policy](examples/neck_policy/)** — Minimal libtorch NN policy that drives JVRC1's neck yaw joint. Good starting point for running NN policies in mc_mujoco.
- **[grasp-fsm](examples/grasp-fsm/)** *(submodule)* — FSM controller that grasps an object on a table using BaselineWalkingController. Pulled from [grasp-fsm-sample-controller](https://github.com/rohanpsingh/grasp-fsm-sample-controller).

---

#### To load additional objects in the scene

There are several steps needed to be followed:

- First, create your object's XML file under `mc_mujoco/robots`. For example, [longtable.xml](robots/longtable.xml)
- Then, create a simple config file with the `xmlModelPath` attribute. For example, [longtable.in.yaml](robots/longtable.in.yaml)
- Then, install your object by adding it to the end of `mc_mujoco/robots/CMakeLists.txt`. For example, add [setup_env_object(box object)](robots/CMakeLists.txt#L15)

Your object is now created and installed. Next you want to tell `mc-mujoco` to load it and place it at the desired pose.
Find `~/.config/mc_rtc/mc_mujoco/mc_mujoco.yaml` (create it manually if needed) and paste the following:

```yaml
objects:
  box1:
    module: "box"
    init_pos:
      translation: [3.1, 0, 0.9]
      rotation: [0, 0, 0]
  box2:
    module: "box"
    init_pos:
      translation: [3.7, 0.2, 0.9]
      rotation: [0, 0, 0]
```

---

#### To load MuJoCo plugins

To load [MuJoCo plugins](https://mujoco.readthedocs.io/en/latest/programming/extension.html#engine-plugins), specify the paths to the directories containing the plugin libraries in the `PluginPaths` entry in `~/.config/mc_rtc/mc_mujoco/mc_mujoco.yaml`.

```yaml
PluginPaths: ["<path-to-plugins>"]
```

They can also be given in the mc_rtc configuration (e.g. the `--mc-config` file), under `MuJoCo`. An entry is a folder
of plugin libraries or one library file, and a relative path is relative to the working directory:

```yaml
MuJoCo:
  PluginPaths: ["<path-to-plugins>", "<path-to/libmy_plugin.so>"]
```

The same `MuJoCo` section can hold `objects`, with the same entries as `mc_mujoco.yaml`. They are added to those of
`mc_mujoco.yaml`, and one with the same name as an object there replaces it. An object whose root body is a mocap body
is moved by `{object}::SetPosW` like a free object.

mc_rtc merges the `MuJoCo` section from all its configuration files before `mc_mujoco` reads it: the installed
`etc/mc_rtc.yaml`, the files in `MC_RTC_CONTROLLER_CONFIG`, `~/.config/mc_rtc/mc_rtc.yaml` (unless
`LoadUserConfiguration: false`), then the `--mc-config` file. A later `PluginPaths` list replaces an earlier one, while
`objects` are merged key by key, inside each object too.

List each plugin library once. Two different copies of one plugin, for example an installed folder in `mc_mujoco.yaml`
and a build-tree library in the run configuration, make MuJoCo stop the process when the second one registers. The same
file listed twice is harmless.

#### Plugin sensors

Declare a plugin sensor inline in the `<sensor>` section of the robot's MJCF, with its configuration as `<config>`
children:

```xml
<extension>
  <plugin plugin="my_plugin"/>
</extension>
<sensor>
  <plugin name="tof" plugin="my_plugin" objtype="site" objname="tof_site">
    <config key="resolution" value="64"/>
  </plugin>
</sensor>
```

When the models are merged, only the attributes of `<extension><plugin>` are kept, so a named `<instance>` declared
there is dropped, and a sensor's `instance=` attribute is not prefixed.

The data of every MuJoCo plugin sensor attached to a robot is copied to the controller's datastore after each
simulation step, as a `std::vector<double>` named `{robot}::{sensor}`, where `{sensor}` is the sensor's name in the
robot's MJCF. The entry appears after the first simulation step, at start and after a reset. The state of the sensor's
plugin instance can be read and written with the `{robot}::{sensor}::GetPluginState` and
`{robot}::{sensor}::SetPluginState` callbacks below, for example to send the plugin commands. The datastore checks the
call's types, so use these exact ones:

```cpp
auto & ds = controller.datastore();
std::vector<double> state;
ds.call<bool, std::vector<double> &>("jvrc1::tof::GetPluginState", state);
ds.call<bool, const std::vector<double> &>("jvrc1::tof::SetPluginState", state);
```

- The state belongs to the plugin instance: two sensors that share one instance read and write the same state.
- A sensor named like a callback of its robot (e.g. `SetPosW`) makes the datastore throw.
- A reset of the simulation runs `mj_resetData`, which calls each plugin's reset but leaves `plugin_state` as it was (a
  new `mjData` also starts with whatever was in memory there). A plugin that takes commands through its state should
  clear it in its reset, and the controller sends its commands again after a reset.
- The callbacks are made again after the controller's `reset()` and its plugins' `reset()` have run: use them in
  `run()`, a plugin's `before()` or an FSM state, or check `ds.has()` first.
- The callbacks lock the simulation mutex, so do not call them from code that already holds it (the rendering thread).
- A plugin instance without state gets no callbacks, and a plugin sensor without a name is not published.

---

#### GUI: Mouse Interaction

An object is selected by left-double-click. The user can then apply forces and torques on the selected object by holding `Ctrl` key and dragging the left-mouse-button for torques and right-mouse-button for forces.

## Datastore callbacks

The following callbacks are available for the controller when running inside `mc_mujoco`

| Signature                                                                           | Description                                                                                                        |
| ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `{robot}::SetPosW(const sva::PTransformd &)`                                         | Set the position of the given `robot`. Also works with objects                                                     |
| `{robot}::SetPDGains(const std::vector<double> & p, const std::vector<double> & d)` | Set the PD gains for the actuation of the given `robot`. `p` and `d` must follow the robot's reference joint order |
| `{robot}::SetPDGainsByName(const std::string & jn, double p, double d)`             | Set the PD gains for a given joint `jn` in `robot`                                                                 |
| `{robot}::GetPDGains(std::vector<double> & p, std::vector<double> & d)`             | Get the current PD gains for `robot` actuation                                                                     |
| `{robot}::GetPDGainsByName(const std::string & jn, double & p, double & d)`         | Get the current PD gains for a given joint `jn` in `robot`                                                         |
| `{robot}::{sensor}::SetPluginState(const std::vector<double> & state)`             | Write the state of a plugin sensor's plugin instance; `state` must have the instance's state size                   |
| `{robot}::{sensor}::GetPluginState(std::vector<double> & state)`                    | Read the state of a plugin sensor's plugin instance                                                                 |

## Citation

```
@inproceedings{singh2023mc,
  title={mc-mujoco: Simulating Articulated Robots with FSM Controllers in MuJoCo},
  author={Singh, Rohan P and Gergondet, Pierre and Kanehiro, Fumio},
  booktitle={2023 IEEE/SICE International Symposium on System Integration (SII)},
  pages={1--5},
  year={2023},
  organization={IEEE}
}
```

### Credits

This package includes code from:

- [imgui v1.84.2](https://github.com/ocornut/imgui/)
- [implot v0.11](https://github.com/epezent/implot)
- [ImGuizmo](https://github.com/CedricGuillemet/ImGuizmo)
- [pugixml v1.11.4](https://github.com/zeux/pugixml)
- [Roboto font](https://github.com/googlefonts/roboto)
