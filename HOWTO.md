# How-To Operate
This document should be a short introduction into the usage of the ARL Upper Limb Robot. 
It will contain general information about the control system and also examples how to interact
and use the robot.
This document describes the robot running with a Raspberry Pi 3.

## Introduction to the Setup
The introduction to the robot's setup as a whole should lead to an easier understanding about how the individual parts interact with each other.

### Technical Background

#### Controllers
Pictures of the Raspberry Pi's actual pin usage and the pin order on the individual controller can be found within the GitHub project containing all [schematics](https://github.com/arne48/rasp-pi_extension). 
[Raspberry Pi Extensions](https://github.com/arne48/rasp-pi_extension/blob/master/Info/pin_usage.odp)

1. __The Air Valve Controller (Activation)__
The Controller for the contineouse air valves _(Festo MPYE-5-M5)_   is a AD5360 DAC from Analog Devices.
This DAC provides a voltage range of __ -10V ... 10V__ in _65536_ steps.
_Festo's MPYE-5-M5_ puts it's actuation range from blow-off to completely open into the range of _0V... 10V_ which leaves the AD5360 with 32768 usable steps for setting the valve's position. 
This range is mapped in within the controllers to a normalized mapping from _-1 ... 1_

2. __The Pressure Sensor Controller (Pressure)__
Analog Device's AD7616 ADC is used for reading out each muscle's pressure sensor _(SMC PSE540-R04)_.
SMC's pressure sensor provides a range of measurement from _0 ... 1MPa_ with a resolution of _16-bit_.
A calibration which would map the ADC's raw measurment to _bar_ or _psi_ is not applied.
The muscles of this robot commonly are acting within a range of _3360 ... 8500_

3. __The Load Cell Controller (Tension)__
The load cells of the tension sensors are read out using the AD7730 of Analog Devices. One module for 16 muscles is populated with 8 of this bridge tranceivers. Each of this 8 is responsible for two tension sensors.
Those 8 transceivers are not read by the Raspberry Pi. Inbetween a STM32F103 acts as an SPI master to the transceivers and as an SPI slave to the Raspberry Pi and aggregates the individual tranceivers measurement for the Raspberry Pi.
The AD7730 provides a resolution of 24-bit and a two-staged filter setup. The setting of this filters and the resolution are hardcoded within the [firmware](https://github.com/arne48/load_cell_controller_firmware). 

#### Communication
1. __Between the controllers and the Raspberry Pi__
The communication between the Raspberry Pi and the controllers is realized using SPI with software chip-selects. The DAC of the air valve and the ADC of the pressure sensor controller are adressed using a 25MHz clock speed. 
The microcontroller of the load cell controller would support up to 18MHz, which according to the available clock dividers of the Raspberry Pi 3 leads to a usable SPI clock speed of 12.5 MHz.

2. __Between Raspberry Pi and external computer__
	
	The control of a specific muscle is provided in two flavors.
	
	__Activation Control__
	For the activation control a _Float64_ message from the _std_msgs_ is used. 
	The topic name is based on the following scheme:
	_/muscle\___\{ascending number\}__\_controller/activation_command_
	The command represents a normalized activation value between __-1 ... 1__ analogous to a fully open or fully blowing-off valve.
	
	__Pressure Control__
	For the pressure control a _Float64_ message from the _std_msgs_ is used as well. 
	The topic name is based on the following scheme:
	_/muscle\___\{ascending number\}__\_controller/pressure_command_
	The command represents the raw measurement of the ADC from __0 ... 65535__ analogous to a pressure from _0 1 MPa_.
	A common value for an empty muscle is about __3360__ with a variable maximum depending on the pressure currently available from the compressor.
	Once requested the desired pressure is maintained by a _PID_ controller which is available for additional tuning using __dynamic_reconfigure__ for each individual muscle controller.
	
	__Muscle Message__
	The current state of a muscle as a state message is also provided.
	The topic name is based on the following scheme:
	_/muscle\___\{ascending number\}__\_controller/state_
	
	Included by this message are the following details:
	- _name(String):_ Name of the described Muscle
	- _desired_pressure(Float64):_ If under pressure control this indictes the goal pressure
	- _current_pressure(Float64):_ The current pressure within the muscle
	- _tension(Float64):_ raw value from the tension sensor's load cell
	- _tension_filtered(Float64):_ filtered version of the tension's sensors values
	- _activation(Float64):_ current value the valve is currently actuated with
	- _control_mode(uint8):_ 0 indicates that muscle is under pressure control while 1 indicates activation control
	
	__Emergency Stop__
	To engage an _Emergency Stop_ a service with the service description _std_srv/Trigger_ can be called.
	The topic of the _Emergency Stop_ is:
	
	_/emergency_stop_
	
	This call blows-off the air of all muscles immediately.

#### Software Components
1. __Driver__
The driver is located in the [arl_hw](https://github.com/arne48/arl_hw) package.
It maintains the realtime control loop and orchestrates the reading and writing of the controllers as well as calls the individual muscle controllers.

2. __Custom Messages__
The available custom messages can be found in the [arl_hw_msgs](https://github.com/arne48/arl_hw_msgs) package
Currently the only one actively used is the _Muscle Message_ which is used for publishing the muscle's detailed state. Details of this message can be found under the earlier bullet point _Muscle Message_.

3. __Muscle Controller__
The custom _MuscleController_ which is located in the [arl_controllers](https://github.com/arne48/arl_controllers) package is in charge of receiving muscle commands and publishing muscle states. Also the PID controller which is used during pressure control and the filter for the raw tension measurements are part of this controller.

4. __Commons__
The [arl_commons](https://github.com/arne48/arl_commons) package contains test nodes and the launch file for starting the _diagnostic_aggregator_ for the driver.

5. __Muscle Tester rqt Plugin__
The [arl_muscle_tester](https://github.com/arne48/arl_muscle_tester) is a plugin for rqt which can be used to test the operability of individual muscles. A more detailed description can be found the it's [README](https://github.com/arne48/arl_muscle_tester/blob/master/README.md).

## Usage of the Package
In this section the start-up procedure of the robot and examples of it's usage will be presented.

### Starting the System
1. Power up the Robot and the Raspberry Pi
2. When the Raspberry Pi is booted up press the reset button on the controllers to bring them into a defined state.
3. Because the on the Raspberry Pi root access is needed for full GPIO access the driver needs to be started as follows:
```bash
sudo su
source /home/${USER}/.bashrc
roslaunch arl_hw arl_robot_driver.launch
``` 
4. Once the driver is started the power switch of the valves (left on the robots backside) can be activated.
5. Now the muscle controllers can be started with regular user permissions
```bash
roslaunch arl_controllers muscle_controller.launch
``` 
6. As the last step the _diagnostic_aggregator_ for the driver will get started by
```bash
roslaunch arl_commons diagnostic_aggregator.launch
``` 
7. Now it should be noticeable that all valves are moderately blowing-off air from the muscles.
8. The robot is ready to operate.

### Examples for using the System

#### Python
##### Controlling a Muscle
```python
#!/usr/bin/env python
import rospy
from std_msgs.msg import Float64

def commander():
    rospy.init_node('muscle_commander', anonymous=True)
    muscle_cmd_pub = rospy.Publisher('/muscle_1_controller/pressure_command', Float64, queue_size=10)
    rate = rospy.Rate(2)
    while not rospy.is_shutdown():
        pub.publish(6000.0)
        rate.sleep()
        
        pub.publish(4000.0)
        rate.sleep()

if __name__ == '__main__':
    try:
        commander()
    except rospy.ROSInterruptException:
        pass
```


##### Receiving Muscle Details
```python
#!/usr/bin/env python
import rospy
from arl_hw_msgs.msg import Muscle

def muscle_callback(msg):
    rospy.loginfo("My muscle %s flexed with an activation of %f", msg.name, msg.activation)
    
def observer():
    rospy.init_node('muscle_observer', anonymous=True)
    rospy.Subscriber("/muscle_1_controller/state", Muscle, muscle_callback)
    rospy.spin()

if __name__ == '__main__':
    observer()
```

#### C++
##### Controlling a Muscle
```cpp
#include "ros/ros.h"
#include "std_msgs/Float64.h"

int main(int argc, char **argv) {
  ros::init(argc, argv, "muscle_commander");
  ros::NodeHandle nh;
  ros::Publisher muscle_cmd_pub = nh.advertise<std_msgs::Float64>("/muscle_1_controller/pressure_command", 10);

  ros::Rate loop_rate(2);
  std_msgs::Float64 msg;
  
  while (ros::ok()) {
    
    msg.data = 6000.0;
    muscle_cmd_pub.publish(msg);
    ros::spinOnce();
    loop_rate.sleep();
    
    msg.data = 4000.0;
    muscle_cmd_pub.publish(msg);
    ros::spinOnce();
    loop_rate.sleep();

  }
  return 0;
}
```

##### Receiving Muscle Details
```cpp
#include "ros/ros.h"
#include "arl_hw_msgs/Muscle.h"

void muscleCallback(const arl_hw_msgs::Muscle::ConstPtr& msg) {
  ROS_INFO("My muscle %s flexed with an activation of %f", msg->name.c_str(), msg->activation);
}

int main(int argc, char **argv) {
  ros::init(argc, argv, "muscle_observer");
  ros::NodeHandle nh;
  
  ros::Subscriber muscle_sub = nh.subscribe("/muscle_1_controller/state", 10, muscleCallback);
  ros::spin();
  
  return 0;
}
```

## Important Remarks
### Hardware
TBA
### Software
TBA