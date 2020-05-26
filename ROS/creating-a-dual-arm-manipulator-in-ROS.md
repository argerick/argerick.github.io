Title: Creating a dual-arm manipulator in ROS
Date: 2017-02-18 07:31
Author: Erick Vieyra
Category: ROS
Tags: ROS, RVIZ, tutorial, URDF, xacro
Slug: creating-a-dual-arm-manipulator-in-ROS
Header_Cover: images/cover.png
OG_Image: images/creating-a-dual-arm-manipulator-in-ROS_dual-arm.png
Status: published

In this blog post we will create a ROS package that contains the URDF model for a dual-arm robot, as well as the necessary configuration files for executing the package and visualizing it in *RVIZ*.
We will go through a lot of the basic features and tools of ROS, like *RVIZ*, URDF files (and *xacro*), *launch* files, package managing, etc.

![dual_arm]({static}/images/creating-a-dual-arm-manipulator-in-ROS_dual-arm.png)

## Creating a package

ROS encapsulates independent functions, methods, robot models, etc, in packages. A package can have dependencies for other packages (indicated when they are created) but packages cannot contain other packages *inside* them. ROS packages have certain characteristics that we will review in the following paragraphs.

### Structure of a package

The most simple ROS package that can exist has the next structure:

```
my_package/
  CMakeLists.txt
  package.xml
```

The name of the package and the root directory must be the same. A ROS package must have a file that *catkin* can compile named `CMakeLists.txt`, and a file named `package.xml` containing information about the package.

Our ROS package will have a slightly more complicated structure given that we want to include an URDF robot description within it.
This is the structure that our package will have:

```
enesbot_dualarm/
   CMakeLists.txt
   package.xml
   launch/
       display.launch
   urdf/
       enesbot_dualarm.xacro
   meshes/
       n0.stl
       n1.stl
       n2.stl
       nb.stl
   rviz/
       conf.rviz
```

The structure is still simplified and only showcases the important contents of the package.
The `launch/` directory will contain the *launch files* (one, for now) that we will use every time we execute the package using ``roslaunch``.
``urdf/`` will contain the *xacro* file that defines our robot (which later gets expanded into modular *URDF* files).
The folder called ``meshes/`` keeps the 3D models that our robot will use.
The folder called ``rviz/`` will contain the *RVIZ* configuration file that we will generate and use when visualizing the robot.

### Execution and dependencies

We can create our package by opening a terminal and navigating to `catkin_ws/src/` in our [previously created ROS workspace](http://enesbot.me/installing-ros) and executing the following command:

``` shell
catkin_create_pkg enesbot_dualarm rospy roscpp std_msgs
```

The command ``catkin_create_pkg`` creates a package with the name ``enesbot\_dualarm`` with ``rospy``, ``roscpp`` and
``std\_msgs`` as dependencies (which are basic dependencies for what we want our package to do).

Executing the command also produces the basic folder structure that we talked about before. We can add the remaining folders and compile the package by returning to the root of our workspace in `catkin_ws` and executing `catkin_make`:

``` shell
catkin_make
```

After compiling a ROS package, we are able to execute it using *roslaunch*, among many other things.

## URDF with xacro

*Xacro* is a very simple language that allows us to create URDF files using macros that can contain simple instructions and basic math.
The main advantage of using *xacro* is that we can take advantage of the iterative nature of robot links by defining them as macros that get repeated with different parameters throughout the robot. Using this approach saves time, increases readability, and is less error-prone.

### Creating a macro

Let's create a macro that generates the code for two links of the robot (united by a joint) that will become the basic building block of each of the robot arms.
To simplify the macro, we will input all the parameters that it needs, plus the ones that indicate its relationship with the other robot links (which link is the *parent*, which ones are the *children*, etc). We will also specify the type of joint (axis position), and the type of 3D model that represents the joint, etc.

``` xml
<xacro:macro name="segment" params="n parent child cad_type xyz origin rpy">

  <joint name="j${n}" type="continuous">
    <parent link="${parent}"/>
    <child link="${child}"/>
    <origin xyz = "${origin}" rpy="${rpy}" />
    <axis xyz="${xyz}" />
  </joint>

  <link name="${child}">
    <visual>
      <geometry>
        <mesh filename="package://enesbot_dualarm/meshes/n${cad_type}.stl"/>
      </geometry>
      <xacro:if value="${cad_type == 2}">
        <material name = "white">
  				<color rgba="1 1 1 1"/>
  			</material>
      </xacro:if>
    </visual>
  </link>

</xacro:macro>
```

Lets analyze the macro part by part:

We start the file with ``<xacro:macro>`` to indicate that what follows is a macro. the arguments ``name`` and ``params`` help us to define the name and the parameters that we will pass to the macro in order to produce code.
Let's explain how each of the parameters affects the way the macro expands:

1.  ``n``: indicates the name of the segment (1, 2, ... *n*).
2.  ``parent``: the segment's parent.
3.  ``child``: the segment's child.
4.  ``cad_type``: the type of 3D model to use (0,1 or 2 that are segment A, segment B or end effector).
5.  ``xyz``: axis of the joint.
6.  ``origin``: origin (or displacement) of the current segment regarding the last one.
7.  ``rpy``: rotation of the current segment regarding the last one.

Our macro works in the following way:
We define a joint with the name ``j${n}`` (where `${x}` is a parameter, so *j1*, *j2*, etc) and then we define its relationship with the rest of the manipulator (the ``parent`` and ``child`` parameters).
The remaining parameters are used *as-is* to specify the features of the joint we are creating.

Afterwards we create a *link* (with the same name as the ``child`` of our joint). Note that we assign a 3D model depending on the value in `cad_type`, so the possible values are ``n0.stl`` (1), ``n1.stl`` (2), or ``n2.stl`` (3).  

Lastly, we use a conditional to check if the segment's `cad_type` is two (which is the end effector) ``xacro:if value="${cad_type == 2}``; If the segment is the end effector, we paint it white instead.

### Meshes

The 3D models employed for this robot were created using [OpenSCAD](http://www.openscad.org) and later imported as ``.stl`` files.
The needed ``.stl`` files are included in the package and they look like this:

![cads]({static}/images/creating-a-dual-arm-manipulator-in-ROS_cads.png)

Many good how-to's and tutorials for OpenSCAD can be found in the [OpenSCAD User Manual](https://en.wikibooks.org/wiki/OpenSCAD_User_Manual).

### The complete *xacro* file

We add the rest of the file contents to produce a complete URDF (after expanding it with the macro). We place a *link* representing the robot's base (trunk) as shown in the following file:

``` xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro" name ="enesbot_dualarm">

  <xacro:macro name="segment" params="n parent child cad_type xyz origin rpy">

    <joint name="j${n}" type="continuous">
      <parent link="${parent}"/>
      <child link="${child}"/>
      <origin xyz = "${origin}" rpy="${rpy}" />
      <axis xyz="${xyz}" />
    </joint>

    <link name="${child}">
      <visual>
        <geometry>
          <mesh filename="package://enesbot_dualarm/meshes/n${cad_type}.stl"/>
        </geometry>
        <xacro:if value="${cad_type == 2}">
          <material name = "white">
    				<color rgba="1 1 1 1"/>
    			</material>
        </xacro:if>
      </visual>
    </link>

  </xacro:macro>

	<link name="root">
		<visual>
			<geometry>
				<mesh filename="package://enesbot_dualarm/meshes/nb.stl"/>
			</geometry>
		</visual>
	</link>

  <xacro:segment n="r1" parent="root" child="r1"   cad_type="1" xyz="1 0 0" origin="0 -0.15 0.4" rpy="0 -0.523599 -0.523599" />
  <xacro:segment n="r2" parent="r1"  child="r2"   cad_type="0" xyz="0 0 1" origin="0.1 0 0" rpy="0 0 0" />
  <xacro:segment n="r3" parent="r2"  child="r3"   cad_type="1" xyz="1 0 0" origin="0.1 0 0" rpy="0 0 0" />
  <xacro:segment n="r4" parent="r3"  child="r4"   cad_type="0" xyz="0 0 1" origin="0.1 0 0" rpy="0 0 0" />
  <xacro:segment n="r5" parent="r4"  child="r5"   cad_type="1" xyz="1 0 0" origin="0.1 0 0" rpy="0 0 0" />
  <xacro:segment n="r6" parent="r5"  child="r6"   cad_type="0" xyz="0 0 1" origin="0.1 0 0" rpy="0 0 0" />
  <xacro:segment n="r7" parent="r6"  child="r7"   cad_type="2" xyz="1 0 0" origin="0.1 0 0" rpy="0 0 0" />

  <xacro:segment n="l1" parent="root" child="l1"   cad_type="1" xyz="1 0 0" origin="0 0.15 0.4" rpy="0 -0.523599 0.523599" />
  <xacro:segment n="l2" parent="l1"  child="l2"   cad_type="0" xyz="0 0 1" origin="0.1 0 0" rpy="0 0 0" />
  <xacro:segment n="l3" parent="l2"  child="l3"   cad_type="1" xyz="1 0 0" origin="0.1 0 0" rpy="0 0 0" />
  <xacro:segment n="l4" parent="l3"  child="l4"   cad_type="0" xyz="0 0 1" origin="0.1 0 0" rpy="0 0 0" />
  <xacro:segment n="l5" parent="l4"  child="l5"   cad_type="1" xyz="1 0 0" origin="0.1 0 0" rpy="0 0 0" />
  <xacro:segment n="l6" parent="l5"  child="l6"   cad_type="0" xyz="0 0 1" origin="0.1 0 0" rpy="0 0 0" />
  <xacro:segment n="l7" parent="l6"  child="l7"   cad_type="2" xyz="1 0 0" origin="0.1 0 0" rpy="0 0 0" />

</robot>

```

To use the *xacro* file, we need to execute it with the tool of the same name ( `xacro` ) just as it is shown in the next command:

``` shell
rosrun xacro xacro enesbot_dualarm.xacro
```

`xacro` expands a xacro file into its corresponding URDF code in the terminal. A very common way of expanding a *xacro* into a file and checking its integrity with ``check_urdf`` is by using the two following commands:

``` shell
rosrun xacro xacro enesbot_dualarm.xacro > exp_enesbot_dualarm.urdf --inorder
check_urdf exp_enesbot_dualarm.urdf
```

## The launch file

We can use a launch file to execute several ROS nodes (processes) that implement the functionality that we need for displaying our robot.
Visualizing a robot in ROS requires ``rviz``, and to be able to control the value of its joints (to move it), we need at least 2 other nodes: ``joint_state_publisher`` and ``robot_state_publisher``.
Let's create a launch file that lets us execute these three nodes with its corresponding arguments in a single command!

### How does a launch file works

Launch files are *xml* files that contain ROS commands. In their most basic form, they contain three types of elements:

-   ``arg``: Elements that we declare ourselves, like the URDF of the robot.
-   ``param``: Elements defined by ROS and used by their nodes.
-   ``node``: processes either part of ROS or written by the developer, like *RVIZ*.

First we will write the a command that loads as arguments our *xacro* file and our *RVIZ* configuration file (more on this last file in a moment).

The parameter ``robot_description`` will take the expanded output from `xacro`.
Since we want to *manually* move the joints of our robot, we will assign the option `True` to the `use_gui` option from the ``joint_state_publisher`` node.  
Lastly, we will execute three nodes: the ``joint_state_publisher``, the `robot_state_publisher` and `rviz`.

Note that `rviz` is executed with the `required=True` option. This means that if the user closes this node, all the processes should terminate their execution too (since we cannot visualize the robot anymore).

### Configuring RVIZ

We mentioned a configuration file for *RVIZ* in the last section. To get it, we can execute *RVIZ* and create one using the following steps:  
1. Run `roscore` in a terminal.
```` shell
roscore
````
2. In another terminal, run the following command to start *RVIZ*:
``` shell
rosrun rviz rviz
```

3. Add views, modify or change the environment as needed. For example it is useful for this example to add the *Robot Model* and *TF* panels from the `Panels` menu.

4. Save the configuration using the `File` menu in the desired location.

![rviz_settings]({static}/images/creating-a-dual-arm-manipulator-in-ROS_rviz-settings.png)

### The launch file

The final version of our launch file will look like this:

```` xml
<launch>

	<arg name = "model" default = "$(find enesbot_dualarm)/urdf/n.xacro"/>
	<arg name="rvizconfig" default="$(find enesbot_dualarm)/rviz/conf.rviz" />
	<param name = "robot_description" command="$(find xacro)/xacro $(arg model) --inorder"/>
	<param name="use_gui" value="true"/>
	<node name="joint_state_publisher" pkg="joint_state_publisher" type="joint_state_publisher" />
	<node name="robot_state_publisher" pkg="robot_state_publisher" type="state_publisher" />
	<node name="rviz" pkg="rviz" type="rviz" required="true" args="-d $(arg rvizconfig)"/>

</launch>

````

Place it in the ``enesbot_dualarm/launch/`` directory.

## Executing the package

Once we have all the files ready, we can try to build our workspace and execute the package to test the manipulator:

Navigate to the root of the workspace and run:
```` shell
catkin_make
````
Then, launch the package using our newly created launch file using the ``roslaunch``:

``` shell
roslaunch enesbot_dualarm display.launch
```

You can use the GUI control from the joint_state_publisher to move the joints of the robot in *RVIZ*!

![rviz]({static}/images/creating-a-dual-arm-manipulator-in-ROS_rviz.png)

The package for this tutorial is available in [Github](https://github.com/sevieyra/enesbot_dualarm)! You can clone it and try it yourself by running the following command:

``` shell
git clone https://github.com/sevieyra/enesbot_dualarm.git
```

*Note*: remember to place it in your catkin workspace!
