# Gazebo Model for the Uvify IFO-S with PX4 Software-in-the-Loop
[TOC] 
![./docs/gazebo_vs_real_flight.gif](./docs/gazebo_vs_real_flight.gif) 
![./docs/ifo_visual.png](./docs/ifo_visual.png) 

This repo is a collection of ROS packages, Gazebo models, Gazebo plugins, as well as the `PX4-Autopilot` source code contained as a git submodule. It does not contain the Gazebo simulator itself, or ROS. These are all dependencies that must be installed as per the instructions below.

Currently, this simulator is intended for Ubuntu 18.04 with ROS Melodic

The repos `ifo_hardware` and `ifo_gazebo` (i.e. this repo) are analogies/replacements of each other, which take care of setting up and providing the interface to the real and simulated IFO quadcopter, respectively. As such, `ifo_hardware` should only ever be cloned on the IFO's Jetson Nano, whilst `ifo_gazebo` should be cloned on your local computer. Any code that is part of a general autonomy stack should be located in other repos and cloned on both the real hardware and the local simulating computer.



## Folder structure

- `./docs/` : A directory for any files related to documentation.
- `./cpr_office_gazebo/` : A copy of the Clearpath Robotics' "Office World".
- `./ifo_description/` : Folder containing CAD meshes and URDF/Xacro files that describe the actual gazebo model
- `./ifo_gazebo/` : Gazebo-specific config files, launch files, and a ROS node that emulates a motion capture system VRPN server.
- `./PX4-Autopilot/` : A submodule containing the PX4 source code.
- `./realsense_gazebo_plugin/` : A submodule of a realsense plugin made by PAL Robotics.

## TODO
1. Add bottom-facing 1D Lidar and camera

## Pre-requisites

1. Gazebo + ROS Melodic (follow standard install instructions for the `desktop-full` option [here](http://wiki.ros.org/melodic/Installation/Ubuntu))
2. Python catkin tools (installable with `sudo apt-get install python3-catkin-tools`)

# Getting Started
This repo is a collection of ROS packages that contains everything you need to create a simulated version of the Uvify IFO-S quadcopter in Gazebo with ROS. 

The `PX4-Autopilot` repo, which contains the software of the Uvify IFO-S flight controller, is stored as submodule of this repo. This allows the PX4 directory to be fixed relative to the gazebo model, which simplifies the simulator start-up procedure since the user no longer needs to specify the location of PX4 source code. Moreover, this gives us control of the exact version of `PX4-Autopilot`, meaning this repo will not break if the PX4 developers push a new commit to the `PX-Autopilot` repo. The `PX4-Autopilot` repo has lots of submodules itself, so it is critical to clone this repo with the `--recursive` flag. Start by creating the folder `~/catkin_ws/src/`. Then

    cd ~/catkin_ws/src
    git clone https://bitbucket.org/decargroup/ifo_gazebo.git --recursive

Feel free to change the above URL to one appropriate for SSH-keys. Alternatively, you can clone this repo regularly and then run `git submodule update --init --recursive`. 

Next, we must blacklist the `PX4-Autopilot` directory in the catkin tools, so that the `catkin build` command does not compile the PX4 source code (since it fails). PX4 must be built manually. While inside your catkin workspac, add the `px4` package to the blacklist,

    catkin config --blacklist px4

You can now run 

    catkin build

and test to see that all the packages get built successfully, with the px4 package being skipped. 

>**Note:  We prefer `catkin build` instead of `catkin_make` to build a ROS workspace, but `catkin_make` should also be possible. Check out the [documentation for catkin](https://catkin-tools.readthedocs.io/en/latest/verbs/catkin_build.html).**

### Building the PX4-Autopilot Source Code
To compile PX4 manually, we must first install the toolchain. Thankfully PX4 provides an install script which installs everything required. The PX4 documentation cites that their installation scripts, are intended for a _clean_ Ubuntu 18.04 installation. It might still work for a non-fresh installation though. 

    bash ~/catkin_ws/src/PX4-Autopilot/Tools/setup/ubuntu.sh

Reboot your computer when complete. Technically, at this point, your computer is also outfitted with a development environment to modify the PX4 source code itself, and do some basic software-in-the-loop (SITL) testing in simulation.  Now, build the PX4 code. 

    cd ~/catkin_ws/src/ifo_gazebo/PX4-Autopilot
    make px4_sitl gazebo

This can take a long time, and can fail multiple times with a C++ compiler failure. The reason of failure is apparently due to your computer running out of RAM. Nevertheless, the build process gets a little further every time, so the solution is just to keep running `make px4_sitl gazebo` until it succeeds. Once successful, you should see the PX4 app start up in the terminal and the Gazebo GUI launching. You should only need to do this build step once, provided that you never modify the PX4 source code.

See PX4's [Ubuntu Development Environment instructions](https://docs.px4.io/master/en/dev_setup/dev_env_linux_ubuntu.html) for reference.

### Installing ROS, Gazebo, MAVROS
 Next, the PX4 documentation conveniently provides an install script which will install all the required software for ROS/Gazebo simulation including ROS Melodic, Gazebo 9, MAVROS and more

    cd ~
    wget https://raw.githubusercontent.com/PX4/Devguide/master/build_scripts/ubuntu_sim_ros_melodic.sh
    bash ubuntu_sim_ros_melodic.sh

See PX4's [ROS/Gazebo installation instructions](https://docs.px4.io/master/en/dev_setup/dev_env_linux_ubuntu.html#rosgazebo) for reference.

Note that this will create some additional folders under `~/catkin_ws/src/` such as `~/catkin_ws/src/mavros/`. If you have no need to modify the source code of these packages, feel free to delete these folders so they arn't compiled every time with `catkin build`. If you do this, install MAVROS dependencies with

    sudo apt-get install ros-melodic-mavros ros-melodic-mavros-extras

As per the official [MAVROS installation instructions](https://github.com/mavlink/mavros/tree/master/mavros#installation), we also need to install some geographic dataset dependency for proper reference frame conversions.

``` bash
cd ~
wget https://raw.githubusercontent.com/mavlink/mavros/master/mavros/scripts/install_geographiclib_datasets.sh
./install_geographiclib_datasets.sh
rm install_geographiclib_datasets.sh     # Delete the file once done.
```

### Launching the simulator
Finally, we need to source the usual setup script in `catkin_ws/devel/setup.bash`, but also a custom script `setup_ifo_gazebo.bash` located in this repo, which manually specifies the paths to relevant dependencies inside the PX4 source using environment variables. These two scripts need to be run for every new terminal. Alternatively, adding them to `~/.bashrc` will automatically execute them with every new terminal

``` bash
echo "source ~/catkin_ws/devel/setup.bash" >> ~/.bashrc
echo "source ~/catkin_ws/src/ifo_gazebo/setup_ifo_gazebo.bash suppress" >> ~/.bashrc
```
Restart your terminal. Then, you should be ready to fire up the simulator

    roslaunch ifo_gazebo ifo_empty_world_1.launch

The PX4 app should be running in the terminal, and the Gazebo GUI should have started, displaying a single quadcopter located at the origin. You can run `rostopic list` and you should see a large list of topics. You can see it in action by typing

    commander takeoff
    
in the same terminal you used to type the previous command (the one with all the PX4 printout). Watch the quadcopter take off and immediately land. You can also open a second terminal and type `rostopic echo /mavros/local_position/pose` to view the state estimate in real time.
