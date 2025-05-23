:github_url: https://github.com/UniversalRobots/Universal_Robots_ROS2_Driver/blob/main/ur_controllers/doc/index.rst

ur_controllers
==============

This package contains controllers and hardware interface for ``ros2_controllers`` that are special to the UR
robot family. Currently this contains:


* A **speed_scaling_state_broadcaster** that publishes the current execution speed as reported by
  the robot to a topic interface. Values are floating points between 0 and 1.
* A **scaled_joint_trajectory_controller** that is similar to the *joint_trajectory_controller*\ ,
  but it uses the speed scaling reported to align progress of the trajectory between the robot and controller.
* A **io_and_status_controller** that allows setting I/O ports, controlling some UR-specific
  functionality and publishes status information about the robot.
* A **tool_contact_controller** that exposes an action to enable the tool contact function on the robot.

About this package
------------------

This package contains controllers not being available in the default ``ros2_controllers`` set. They are
created to support more features offered by the UR robot family. Some of these controllers are
example implementations for certain features and are intended to be generalized and merged
into the default ``ros2_controllers`` controller set at some future point.

Controller description
----------------------

This packages offers a couple of specific controllers that will be explained in the following
sections.

.. _speed_scaling_state_broadcaster:

ur_controllers/SpeedScalingStateBroadcaster
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This controller publishes the current actual execution speed as reported by the robot. Values are
floating points between 0 and 1.

In the `ur_robot_driver
<https://index.ros.org/p/ur_robot_driver>`_
this is calculated by multiplying the two `RTDE
<https://www.universal-robots.com/articles/ur/real-time-data-exchange-rtde-guide/>`_ data
fields ``speed_scaling`` (which should be equal to the value shown by the speed slider position on the
teach pendant) and ``target_speed_fraction`` (Which is the fraction to which execution gets slowed
down by the controller).

.. _scaled_jtc:

ur_controlers/ScaledJointTrajectoryController
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

These controllers work similar to the well-known
`joint_trajectory_controller <https://control.ros.org/master/doc/ros2_controllers/joint_trajectory_controller/doc/userdoc.html>`_.

However, they are extended to handle the robot's execution speed specifically. Because the default
``joint_trajectory_controller`` would interpolate the trajectory with the configured time constraints (ie: always assume maximum velocity and acceleration supported by the robot),
this could lead to significant path deviation due to multiple reasons:


* The speed slider on the robot might not be at 100%, so motion commands sent from ROS would
  effectively get scaled down resulting in a slower execution.
* The robot could scale down motions based on configured safety limits resulting in a slower motion
  than expected and therefore not reaching the desired target in a control cycle.
* Motions might not be executed at all, e.g. because the robot is E-stopped or in a protective stop
* Motion commands sent to the robot might not be interpreted, e.g. because there is no
  `external_control <https://github.com/UniversalRobots/Universal_Robots_ROS_Driver#prepare-the-robot>`_
  program node running on the robot controller.
* The program interpreting motion commands could be paused.

The following plot illustrates the problem:

.. image:: traj_without_speed_scaling.png
   :target: traj_without_speed_scaling.png
   :alt: Trajectory execution with default trajectory controller


The graph shows a trajectory with one joint being moved to a target point and back to its starting
point. As the joint's speed is limited to a very low setting on the teach pendant, speed scaling
(black line) activates and limits the joint speed (green line). As a result, the target
trajectory (light blue) doesn't get executed by the robot, but instead the pink trajectory is executed.
The vertical distance between the light blue line and the pink line is the path error in each
control cycle. We can see that the path deviation gets above 300 degrees at some point and the
target point at -6 radians never gets reached.

All of the cases mentioned above are addressed by the scaled trajectory versions. Trajectory execution
can be transparently scaled down using the speed slider on the teach pendant without leading to
additional path deviations. Pausing the program or hitting the E-stop effectively leads to
``speed_scaling`` being 0 meaning the trajectory will not be continued until the program is continued.
This way, trajectory executions can be explicitly paused and continued.

With the scaled version of the trajectory controller the example motion shown in the previous diagram becomes:

.. image:: traj_with_speed_scaling.png
   :target: traj_with_speed_scaling.png
   :alt: Trajectory execution with scaled_joint_trajectory_controller


The deviation between trajectory interpolation on the ROS side and actual robot execution stays minimal and the
robot reaches the intermediate setpoint instead of returning "too early" as in the example above.

Under the hood this is implemented by proceeding the trajectory not by a full time step but only by
the fraction determined by the current speed scaling. If speed scaling is currently at 50% then
interpolation of the current control cycle will start half a time step after the beginning of the
previous control cycle.

.. _io_and_status_controller:

ur_controllers/GPIOController
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This controller allows setting I/O ports, controlling some UR-specific functionality and publishes
status information about the robot.

Published topics
""""""""""""""""

* ``~/io_states [ur_msgs/msg/IOStates]``: Status of all I/O ports
* ``~/robot_mode [ur_dashboard_msgs/msg/RobotMode]``: The current robot mode (e.g. ``POWER_OFF``,
  ``IDLE``, ``RUNNING``)
* ``~/robot_program_running [std_msgs/msg/Bool]``: Publishes whether **the External Control
  program** is running or not. If this is ``false`` no commands can be sent to the robot.
* ``~/safety_mode [ur_dashboard_msgs/msg/SafetyMode]``: The robot's current safety mode (e.g.
  ``PROTECTIVE_STOP``, ``ROBOT_EMERGENCY_STOP``, ``NORMAL``)
* ``~/tool_data [ur_msgs/msg/ToolDataMsg]``: Information about the robot's tool configuration

Advertised services
"""""""""""""""""""

* ``~/hand_back_control [std_srvs/srv/Trigger]``: Calling this service will make the robot program
  exit the *External Control* program node and continue with the rest of the program.
* ``~/resend_robot_program [std_srvs/srv/Trigger]``: When :ref:`headless_mode` is used, this
  service can be used to restart the *External Control* program on the robot.
* ``~/set_io [ur_msgs/srv/SetIO]``: Set an output pin on the robot.
* ``~/set_analog_output [ur_msgs/srv/SetAnalogOutput]``: Set an analog output on the robot. This
  also allows specifying the domain.
* ``~/set_payload [ur_msgs/srv/SetPayload]``: Change the robot's payload on-the-fly.
* ``~/set_speed_slider [ur_msgs/srv/SetSpeedSliderFraction]``: Set the value of the speed slider.
* ``~/zero_ftsensor [std_srvs/srv/Trigger]``: Zeroes the reported wrench of the force torque
  sensor.

.. _passthrough_trajectory_controller:

ur_controllers/PassthroughTrajectoryController
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This controller uses a ``control_msgs/FollowJointTrajectory`` action but instead of interpolating
the trajectory on the ROS pc it forwards the complete trajectory to the robot controller for
interpolation and execution. This way, the realtime requirements for the control PC can be
massively decreased, since the robot always "knows" what to do next. That means that you should be
able to run a stable driver connection also without a real-time patched kernel.

Interpolation depends on the robot controller's implementation, but in conjunction with the
ur_robot_driver it defaults to mimicking ros2_control's spline interpolation. So, any trajectory
planned e.g. with MoveIt! will be executed following the trajectory exactly.

A trajectory sent to the controller's action server will be forwarded to the robot controller and
executed there. Once all setpoints are transferred to the robot, the controller goes into a waiting
state where it waits for the trajectory to be finished. While waiting, the controller tracks the
time spent on the trajectory to ensure the robot isn't stuck during execution.

This controller also supports **speed scaling** such that and scaling down of the trajectory done
by the robot, for example due to safety settings on the robot or simply because a slower execution
is configured on the teach pendant. This will be considered, during execution monitoring, so the
controller basically tracks the scaled time instead of the real time.

.. note::

   When using this controller with the URSim simulator execution times can be slightly larger than
   the expected time depending on the simulation host's resources. This effect will not be present
   when using a real UR arm.

.. note::

   This controller can currently only be used with URSim or a real UR robot. Neither mock hardware
   nor gazebo support this type of trajectory interfaces at the time being.

Tolerances
""""""""""

Currently, the trajectory passthrough controller only supports goal tolerances and goal time
tolerances passed in the action directly. Please make sure that the tolerances are completely
filled with all joint names.

A **goal time tolerance** of ``0.0`` means that no goal time tolerance is set and the action will
not fail when execution takes too long.

Action interface / usage
""""""""""""""""""""""""

To use this controller, publish a goal to the ``~/follow_joint_trajectory`` action interface
similar to the `joint_trajectory_controller <https://control.ros.org/master/doc/ros2_controllers/joint_trajectory_controller/doc/userdoc.html>`_.

Currently, the controller doesn't support replacing a running trajectory action. While a trajectory
is being executed, goals will be rejected until the action has finished. If you want to replace it,
first cancel the running action and then send a new one.

Parameters
""""""""""

The trajectory passthrough controller uses the following parameters:

+----------------------------------+--------------+----------------------------------------+------------------------------------------------------------------------------------------------------------------+
| Parameter name                   | Type         | Default value                          | Description                                                                                                      |
|                                  |              |                                        |                                                                                                                  |
+----------------------------------+--------------+----------------------------------------+------------------------------------------------------------------------------------------------------------------+
| ``joints`` (required)            | string_array | <empty>                                | Joint names to  listen to                                                                                        |
+----------------------------------+--------------+----------------------------------------+------------------------------------------------------------------------------------------------------------------+
| ``state_interfaces`` (required)  | string_array | <empty>                                | State interfaces provided by the hardware for all joints. Subset of ``["position", "velocity", "acceleration"]`` |
+----------------------------------+--------------+----------------------------------------+------------------------------------------------------------------------------------------------------------------+
| ``speed_scaling_interface_name`` | string       | ``speed_scaling/speed_scaling_factor`` | Fully qualified name of the speed scaling interface name.                                                        |
+----------------------------------+--------------+----------------------------------------+------------------------------------------------------------------------------------------------------------------+
| ``tf_prefix``                    | string       | <empty>                                | Urdf prefix of the corresponding arm                                                                             |
+----------------------------------+--------------+----------------------------------------+------------------------------------------------------------------------------------------------------------------+

Interfaces
""""""""""

In order to use this, the hardware has to export a command interface for passthrough operations for each joint. It always has
to export position, velocity and acceleration interfaces in order to be able to project the full
JointTrajectory definition. This is why there are separate fields used, as for passthrough mode
accelerations might be relevant also for robots that don't support commanding accelerations
directly to their joints.

.. code:: xml

   <gpio name="${tf_prefix}trajectory_passthrough">
     <command_interface name="setpoint_positions_0"/>
     <command_interface name="setpoint_positions_1"/>
     <command_interface name="setpoint_positions_2"/>
     <command_interface name="setpoint_positions_3"/>
     <command_interface name="setpoint_positions_4"/>
     <command_interface name="setpoint_positions_5"/>
     <command_interface name="setpoint_velocities_0"/>
     <command_interface name="setpoint_velocities_1"/>
     <command_interface name="setpoint_velocities_2"/>
     <command_interface name="setpoint_velocities_3"/>
     <command_interface name="setpoint_velocities_4"/>
     <command_interface name="setpoint_velocities_5"/>
     <command_interface name="setpoint_accelerations_0"/>
     <command_interface name="setpoint_accelerations_1"/>
     <command_interface name="setpoint_accelerations_2"/>
     <command_interface name="setpoint_accelerations_3"/>
     <command_interface name="setpoint_accelerations_4"/>
     <command_interface name="setpoint_accelerations_5"/>
     <command_interface name="transfer_state"/>
     <command_interface name="time_from_start"/>
     <command_interface name="abort"/>
   </gpio>

.. note::

   The hardware component has to take care that the passthrough command interfaces cannot be
   activated in parallel to the streaming command interfaces.

Implementation details / dataflow
"""""""""""""""""""""""""""""""""

* A trajectory passed to the controller will be sent to the hardware component one by one.
* The controller will send one setpoint and then wait for the hardware to acknowledge that it can
  take a new setpoint.
* This happens until all setpoints have been transferred to the hardware. Then, the controller goes
  into a waiting state where it monitors execution time and waits for the hardware to finish
  execution.
* If execution takes longer than anticipated, a warning will be printed.
* If execution finished taking longer than expected (plus the goal time tolerance), the action will fail.
* When the hardware reports that execution has been aborted (The ``passthrough_trajectory_abort``
  command interface), the action will be aborted.
* When the action is preempted, execution on the hardware is preempted.

.. _force_mode_controller:

ur_controllers/ForceModeController
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This controller activates the robot's *Force Mode*. This allows direct force control running on the
robot control box. This controller basically interfaces the URScript function ``force_mode(...)``.

Force mode can be combined with (and only with) the :ref:`passthrough trajectory controller
<passthrough_trajectory_controller>` in order to execute motions under a given force constraints.

.. note::
   This is not an admittance controller, as given force constraints in a certain Cartesian
   dimension will overwrite the motion commands in that dimension. E.g. when specifying a certain
   force in the base frame's ``z`` direction, any motion resulting from the move command in the
   base frame's ``z`` axis will not be executed.

Parameters
""""""""""

+----------------------------------+--------+---------------+---------------------------------------------------------------------+
| Parameter name                   | Type   | Default value | Description                                                         |
|                                  |        |               |                                                                     |
+----------------------------------+--------+---------------+---------------------------------------------------------------------+
| ``tf_prefix``                    | string | <empty>       | Urdf prefix of the corresponding arm                                |
+----------------------------------+--------+---------------+---------------------------------------------------------------------+
| ``check_io_successful_retries``  | int    | 10            | Amount of retries for checking if setting force_mode was successful |
+----------------------------------+--------+---------------+---------------------------------------------------------------------+

Service interface / usage
"""""""""""""""""""""""""

The controller provides two services: One for activating force_mode and one for leaving it. To use
those services, the controller has to be in ``active`` state.

* ``~/stop_force_mode [std_srvs/srv/Trigger]``: Stop force mode
* ``~/start_force_mode [ur_msgs/srv/SetForceMode]``: Start force mode

In ``ur_msgs/srv/SetForceMode`` the fields have the following meanings:

task_frame
   All information (selection vector, wrench, limits, etc) will be considered to be relative
   to that pose. The pose's frame_id can be anything that is transformable to the robot's
   ``base`` frame.
selection_vector_<x,y,z,rx,ry,rz>
   1 means that the robot will be compliant in the corresponding axis of the task frame.
wrench
   The forces/torques the robot will apply to its environment. The robot adjusts its position
   along/about compliant axis in order to achieve the specified force/torque. Values have no effect for non-
   compliant axes.
   Actual wrench applied may be lower than requested due to joint safety limits.
type
   An integer [1;3] specifying how the robot interprets the force frame

   1
      The force frame is transformed in a way such that its y-axis is aligned with a vector pointing
      from the robot tcp towards the origin of the force frame.
   2
      The force frame is not transformed.
   3
      The force frame is transformed in a way such that its x-axis is the projection of the robot tcp
      velocity vector onto the x-y plane of the force frame.
speed_limits
   Maximum allowed tcp speed (relative to the task frame). This is **only relevant for axes marked as
   compliant** in the selection_vector.
deviation_limits
   For **non-compliant axes**, these values are the maximum allowed deviation along/about an axis
   between the actual tcp position and the one set by the program.
damping_factor
   Force mode damping factor. Sets the damping parameter in force mode. In range [0;1], default value is 0.025
   A value of 1 is full damping, so the robot will decelerate quickly if no force is present. A value of 0
   is no damping, here the robot will maintain the speed.
gain_scaling
   Force mode gain scaling factor. Scales the gain in force mode. scaling parameter is in range [0;2], default is 0.5.
   A value larger than 1 can make force mode unstable, e.g. in case of collisions or pushing against hard surfaces.

.. _freedrive_mode_controller:

ur_controllers/FreedriveModeController
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This controller activates the robot's *Freedrive Mode*, allowing to manually move the robot' joints.
This controller can't be combined with any other motion controller.

Parameters
""""""""""

+----------------------+--------+---------------+---------------------------------------------------------------------------------------+
| Parameter name       | Type   | Default value | Description                                                                           |
|                      |        |               |                                                                                       |
+----------------------+--------+---------------+---------------------------------------------------------------------------------------+
| ``tf_prefix``        | string | <empty>       | Urdf prefix of the corresponding arm                                                  |
+----------------------+--------+---------------+---------------------------------------------------------------------------------------+
| ``inactive_timeout`` | int    | 1             | Time interval (in seconds) of message inactivity after which freedrive is deactivated |
+----------------------+--------+---------------+---------------------------------------------------------------------------------------+

Usage
"""""

The controller provides the ``~/enable_freedrive_mode`` topic of type ``[std_msgs/msg/Bool]`` for handling activation and deactivation:

* to start and keep freedrive active, you'll have to frequently publish a ``True`` msg on the indicated topic.
  If no further messages are received by the controller within the ``inactive_timeout`` seconds,
  freedrive mode will be deactivated. Hence, it is recommended to publish a ``True`` message at least every
  ``inactive_timeout/2`` seconds.

  .. code-block::

     ros2 topic pub --rate 2 /freedrive_mode_controller/enable_freedrive_mode std_msgs/msg/Bool "{data: true}"

* to deactivate freedrive mode is enough to publish a ``False`` msg on the indicated topic or
  to deactivate the controller or to stop publishing ``True`` on the enable topic and wait for the
  controller timeout.

.. _tool_contact_controller:

ur_controllers/ToolContactController
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
This controller can enable tool contact on the robot. When tool contact is enabled, and the robot
senses whether the tool has made contact with something. When that happens, it will stop all
motion, and retract to where it first sensed the contact.

This controller can be used with any of the motion controllers.

The controller is not a direct representation of the URScript function `tool_contact(direction)
<https://www.universal-robots.com/manuals/EN/HTML/SW5_21/Content/prod-scriptmanual/all_scripts/tool_contact%28direction%29.htm?Highlight=tool_contact>`_,
as it does not allow for choosing the direction. The direction of tool contact will always be the
current TCP direction of movement.

Parameters
""""""""""

+-------------------------+--------+---------------+---------------------------------------------------------------------------------------+
| Parameter name          | Type   | Default value | Description                                                                           |
|                         |        |               |                                                                                       |
+-------------------------+--------+---------------+---------------------------------------------------------------------------------------+
| ``tf_prefix``           | string | <empty>       | Urdf prefix of the corresponding arm                                                  |
+-------------------------+--------+---------------+---------------------------------------------------------------------------------------+
| ``action_monitor_rate`` | double | 20.0          | The rate at which the action should be monitored in Hz.                               |
+-------------------------+--------+---------------+---------------------------------------------------------------------------------------+

Action interface / usage
""""""""""""""""""""""""
The controller provides one action for enabling tool contact. For the controller to accept action goals it needs to be in ``active`` state.

* ``~/detect_tool_contact [ur_msgs/action/ToolContact]``

  The action definition of ``ur_msgs/action/ToolContact`` has no fields, as a call to the action implicitly means that tool contact should be enabled.
  The result of the action is available through the status of the action itself. If the action succeeds it means that tool contact was detected, otherwise tool contact will remain active until it is either cancelled by the user, or aborted by the hardware.
  The action provides no feedback.

  The action can be called from the command line using the following command, when the controller is active:

  .. code-block::

     ros2 action send_goal /tool_contact_controller/detect_tool_contact ur_msgs/action/ToolContact
