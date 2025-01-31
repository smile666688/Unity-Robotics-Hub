# Pick-and-Place Tutorial: Part 3

This part assumes you have access to a functional ROS workspace and that the previous two parts ([Part 1](1_urdf.md), [Part 2](2_ros_tcp.md)) have been completed.

Steps covered in this tutorial includes invoking a motion planning service in ROS, moving a Unity Articulation Body based on the calculated trajectory, and controlling a gripping tool to successfully grasp a cube.

**Table of Contents**
  - [The Unity Side](#the-unity-side)
  - [The ROS Side](#the-ros-side)
  - [ROS–Unity Communication](#rosunity-communication)
  - [Resources](#resources)
  - [Troubleshooting](#troubleshooting)
    - [Errors and Warnings](#errors-and-warnings)
    - [Hangs, Timeouts, and Freezes](#hangs-timeouts-and-freezes)
    - [Miscellaneous Issues](#miscellaneous-issues)

---

## Part 3: Pick-and-Place

## The Unity Side

1. If you have not already completed the steps in [Part 1](1_urdf.md) to set up the Unity project and [Part 2](2_ros_tcp.md) to integrate ROS with Unity, do so now.

1. If the PickAndPlaceProject Unity project is not already open, select and open it from the Unity Hub.

    > Note the `Assets/Scripts/TrajectoryPlanner.cs` script. This is where all of the logic to invoke a motion planning service lives, as well as the logic to control the gripper end effector tool.

    > The UI button `OnClick` callback will be reassigned later in this tutorial to the following function, `PublishJoints`, as defined:

    ```csharp
    public void PublishJoints()
    {
        MoverServiceRequest request = new MoverServiceRequest();
        request.joints_input = CurrentJointConfig();
        
        // Pick Pose
        request.pick_pose = new RosMessageTypes.Geometry.Pose
        {
            position = new Point(
                target.transform.position.z,
                -target.transform.position.x,
                // Add pick pose offset to position the gripper above target to avoid collisions
                target.transform.position.y + pickPoseOffset
            ),
            // Orientation is hardcoded for this example so the gripper is always directly above the target object
            orientation = pickOrientation
        };

        // Place Pose
        request.place_pose = new RosMessageTypes.Geometry.Pose
        {
            position = new Point(
                targetPlacement.transform.position.z,
                -targetPlacement.transform.position.x,
                // Use the same pick pose offset so the target cube can be seen dropping into position
                targetPlacement.transform.position.y + pickPoseOffset
            ),
            // Orientation is hardcoded for this example so the gripper is always directly above the target object
            orientation = pickOrientation
        };

        ros.SendServiceMessage<MoverServiceResponse>(rosServiceName, request, TrajectoryResponse);
    }

    void TrajectoryResponse(MoverServiceResponse response)
    {
        if (response.trajectories != null)
        {
            Debug.Log("Trajectory returned.");
            StartCoroutine(ExecuteTrajectories(response));
        }
        else
        {
            Debug.LogError("No trajectory returned from MoverService.");
        }
    }
    ```

    > This is similar to the `SourceDestinationPublisher.Publish()` function, but with a few key differences. There is an added `pickPoseOffset` to the `pick` and `place_pose` `y` component. This is because the calculated trajectory to grasp the `target` object will hover slightly above the object before grasping it in order to avoid potentially colliding with the object. Additionally, this function calls `CurrentJointConfig()` to assign the `request.joints_input` instead of assigning the values individually.

    > The `response.trajectories` are received in the `TrajectoryResponse()` callback, as defined in the `ros.SendServiceMessage` parameters. These trajectories are passed to `ExecuteTrajectories()` below:

    ```csharp
    private IEnumerator ExecuteTrajectories(MoverServiceResponse response)
    {
        if (response.trajectories != null)
        {
            for (int poseIndex  = 0 ; poseIndex < response.trajectories.Length; poseIndex++)
            {
                for (int jointConfigIndex  = 0 ; jointConfigIndex < response.trajectories[poseIndex].joint_trajectory.points.Length; jointConfigIndex++)
                {
                    var jointPositions = response.trajectories[poseIndex].joint_trajectory.points[jointConfigIndex].positions;
                    float[] result = jointPositions.Select(r=> (float)r * Mathf.Rad2Deg).ToArray();
                    
                    for (int joint = 0; joint < jointArticulationBodies.Length; joint++)
                    {
                        var joint1XDrive  = jointArticulationBodies[joint].xDrive;
                        joint1XDrive.target = result[joint];
                        jointArticulationBodies[joint].xDrive = joint1XDrive;
                    }
                    yield return new WaitForSeconds(jointAssignmentWait);
                }

                if (poseIndex == (int)Poses.Grasp)
                    CloseGripper();
                
                yield return new WaitForSeconds(poseAssignmentWait);
            }
            // Open Gripper at end of sequence
            OpenGripper();
        }
    }
    ```

    > `ExecuteTrajectories` iterates through the joints to assign a new `xDrive.target` value based on the ROS service response, until the goal trajectories have been reached. Based on the pose assignment, this function may call the `OpenGripper` or `CloseGripper` methods as is appropriate.

1. Return to Unity. Select the Publisher GameObject and add the `TrajectoryPlanner` script as a component.

1. Note that the TrajectoryPlanner component shows its member variables in the Inspector window, which need to be assigned. 

    Once again, drag and drop the `Target` and `TargetPlacement` objects onto the Target and Target Placement Inspector fields, respectively. Assign the `niryo_one` robot to the Niryo One field. Finally, assign the RosConnect object to the `Ros` field.

    ![](img/3_target.gif)

1. Select the previously made Button object in Canvas/Button, and scroll to see the Button component. Under the `OnClick()` header, click the dropdown where it is currently assigned to the SourceDestinationPublisher.Publish(). Replace this call with TrajectoryPlanner > `PublishJoints()`.

    ![](img/3_onclick.png)

1. The Unity side is now ready to communicate with ROS to motion plan!

---

## The ROS Side

> Note: This project was built using the ROS Melodic distro, and Python 2.

> Note the file `src/niryo_moveit/scripts/mover.py`. This script holds the ROS-side logic for the MoverService. When the service is called, the function `plan_pick_and_place()` runs. This calls `plan_trajectory` on the current joint configurations (sent from Unity) to a destination pose (dependent on the phase of the pick-and-place task).

```python
def plan_trajectory(move_group, destination_pose, start_joint_angles): 
    current_joint_state = JointState()
    current_joint_state.name = joint_names
    current_joint_state.position = start_joint_angles

    moveit_robot_state = RobotState()
    moveit_robot_state.joint_state = current_joint_state
    move_group.set_start_state(moveit_robot_state)

    move_group.set_pose_target(destination_pose)
    plan = move_group.go(wait=True)

    if not plan:
        print("RAISE NO PLAN ERROR")

    return move_group.plan()
```

> This creates a set of planned trajectories, iterating through a pre-grasp, grasp, pick up, and place set of poses. Finally, this set of trajectories is sent back to Unity.

### Use Docker Container

1. If you are using ROS docker container and have not already build the ROS docker image. Follow the steps in [Part 2](2_ros_tcp.md) to build the `unity-robotics:pick-and-place` docker image.

### Manually Setup ROS

1. If you have not already built and sourced the ROS workspace since importing the new ROS packages, navigate to your ROS workplace, e.g. `Unity-Robotics-Hub/tutorials/pick_and_place/ROS/`, run `catkin_make && source devel/setup.bash`. Ensure there are no errors.

1. If you have not already set the ROS parameter values in the `params.yaml`, navigate to `src/niryo_moveit/config/params.yaml` and open the file for editing. Follow the steps in [Part 2](2_ros_tcp.md) to configure the values.

1. The ROS side is now ready to interface with Unity!

--- 

## ROS–Unity Communication

### Use Docker Container

1. If you have not already, [build the ROS docker image](2_ros_tcp.md#use-docker-container) before executing the following command lines.

2. Run ROS in a new docker container

  ```bash
  docker run -it --rm -p 10000:10000 -p 5005:5005 unity-robotics:pick-and-place part_3 /bin/bash
  ```

3. Terminate docker container

    Press `Ctrl + C` or `Cmd + C` to terminate the docker container.

### Manually Setup ROS

1. Open a new terminal window in the ROS workspace. Once again, source the workspace. 

    Then, run the following `roslaunch` in order to start roscore, set the ROS parameters, start the server endpoint, start the Mover Service node, and launch MoveIt. 
    - This launch file also loads all relevant files and starts ROS nodes required for trajectory planning for the Niryo One robot (`demo.launch`). The launch files for this project are available in the package's `launch` directory, i.e. `src/niryo_moveit/launch/`.
  
	    > Note: Descriptions of what these files are doing can be found [here](moveit_file_descriptions.md).

    ```bash
    roslaunch niryo_moveit part_3.launch
    ```

    This launch will print various messages to the console, including the set parameters and the nodes launched. 
 The final two messages should confirm `You can start planning now!` and `Ready to plan`.

    > Note: This may print out various error messages such as `Failed to find 3D sensor plugin`. These messages are safe to ignore as long as the final message to the console is `You can start planning now!`.

1. Return to the Unity Editor and press Play. Press the UI Button to send the joint configurations to ROS, and watch the robot arm pick up and place the cube! 
   - The target object and placement positions can be moved around during runtime for different trajectory calculations. 
  
![](img/0_pick_place.gif)

---

## Resources

- [MoveIt!](https://github.com/ros-planning/moveit)
- Unity [Articulation Body Documentation](https://docs.unity3d.com/2020.1/Documentation/ScriptReference/ArticulationBody.html)

---

## Troubleshooting

### Errors and Warnings

- If the motion planning script throws a `RuntimeError: Unable to connect to move_group action server 'move_group' within allotted time (5s)`, ensure the `roslaunch niryo_moveit part_3.launch` process launched correctly and has printed `You can start planning now!`.
  
- `...failed because unknown error handler name 'rosmsg'` This is due to a bug in an outdated package version. Try running `sudo apt-get update && sudo apt-get upgrade` to upgrade.

### Hangs, Timeouts, and Freezes

- If Unity fails to find a network connection, ensure that the ROS IP address is entered correctly as the Host Name in the RosConnect in Unity, and that the `src/niryo_moveit/config/params.yaml` values are set correctly. 

### Miscellaneous Issues

- If the robot appears loose/wiggly or is not moving with no console errors, ensure that the Stiffness and Damping values on the Controller script of the `niryo_one` object are set to `10000` and `100`, respectively.

- If the robot moves to the incorrect location, or executes the poses in an expected order, verify that the shoulder_link (i.e. `niryo_one/world/base_link/shoulder_link`) X Drive Force Limit is `5`.

- Before entering Play mode in the Unity Editor, ensure that all ROS processes are still running. The `server_endpoint` node may time out and will need to be re-run.