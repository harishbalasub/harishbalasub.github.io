# AI for Autonomous Navigation - Tesla

## Some autonomous systems in use
Vaccum robots are some of the most successful home robots in use. Some vaccum robot models like iRobot Roomba (launched in 2002) use a camera for [visual SLAM](https://www.technologyreview.com/2015/09/16/247936/the-roomba-now-sees-and-maps-a-home/) (Simultaneous Localization and Mapping), other models like Neato D10 use 2D Lidar for SLAM. Use of a Lidar indoors allows it to work in dark areas, enables faster mapping and a
ords more privacy for residents. Neato Robotics after 18 years is now being shutdown due to underperformance. iRobot has been acquired by Amazon leading to monopoly and more privacy concerns.

## Tesla HW and SW evolution
The [hardware releases](https://en.wikipedia.org/wiki/Tesla_Autopilot#Hardware) so far
* HW1 (2014-)

   Mobileye EyeQ primarily about stop & go adaptive cruise control and lane-centering on the freeway
* HW2 (2016-)

   Enhanced autopilot with NVIDIA DRIVE PX 2, 8 cameras, front radar, 12 ultrasonic sensors
* HW3 (2020-)

   Tesla FSD computer with 8 1280x960 cameras @ 36hz. Removed radar.
* HW4 (2023-)

   Tesla FSD computer 2 with better 5.4mp cameras. Radar back?

The Full Self Driving Beta SW stack is an optional upgrade to the Autopilot SW stack. The additional capabilities include Traffic & Stop sign Control, Autosteer on city streets. While the Autopilot SW stack kept its focus on relative easier Highway driving, the complexity of driving in city streets needed a different SW architecture. Over time a single FSD SW stack is now capable of handling both city streets and highways.

Key [FSD beta releases](https://www.notateslaapp.com/fsd-beta/) include
* Oct 2020 first public beta release on HW3 capable cars. 2D vector space for planning.
* v10.11 (Mar 2022) - Lane guidance module for intersections with autoregressive decoders 
* v10.69 (June 2022) - Introduction of 3D occupancy networks and flow for planning
* v11 (Nov 2022) - Enable FSD beta on both freeway and city streets. As this release went widepsread in early 2023, comfort level for FSD users from social media reports was seen to be much improved. Personally, v11 release was when FSD beta changed from a curiosity to a useful feature.
* v12 - Elon's claim of end to end ai?


## Why no Lidar, Radar? Ultrasonics?
Tesla is betting on a vision based FSD. The Autopilot team is especially confident that vision can be used to provide depth and velocity information reliably.
Lidar is sparse compared to vision images by 10-100x. Higher resolution Lidars are very costly at this point of time.
The figures below show commonly understood relative strengths of various sensors

![alt text](https://github.com/harishbalasub/harishbalasub.github.io/blob/master/docs/assets/img/Tesla-sensor1.png)
![alt text](https://github.com/harishbalasub/harishbalasub.github.io/blob/master/docs/assets/img/Tesla-sensor2.png)

Tesla seems pretty confident about not needing Lidar in production cars and they can confirm the accuracy of their vision based predictions in their unit tests. Radar on the other hand may be useful in inclement weather where we have not seen a lot of public FSD beta tests so far. Radar may make a comeback with HW4.

Similarly Ultrasonic sensors could be useful while parking in confined spaces like garages. From personal experience, the vision based predictions inside my garage are not very reliable and i have a damaged bike to speak for it.


## Architecture
### End to End Learning
End-to-End learning is probably the holy grail for autonomous navigation but is harder to achieve in practice. Driving is much harder state space but simpler set of actions unlike self-play games like Go. Wayve is a London-based company that is approaching self-driving cars with end-to-end learning approach.

![alt text](https://github.com/harishbalasub/harishbalasub.github.io/blob/master/docs/assets/img/Tesla-e2e-learning.png)
### Modular
Most companies (like Tesla, Waymo) go for the modular architecture approach. It allows teams to focus and build, optimize and debug various modules independently. Below is an example of a modular architecture.

<img src="https://github.com/harishbalasub/harishbalasub.github.io/blob/master/docs/assets/img/Tesla-modular.png" width="500" height="250">

### Tesla Architecture
The Tesla architecture is vision based. The vision subsystem converts the input camera data into an intermediate vector space for use by planning and control subsystems.
The vision subsystem is fully NN based. The vector space representation has changed from 2D to recently 3D space (occupancy networks). The planning module is partly NN based heuristics and part explicit coding. Elon recently hinted that v12 FSD beta will be fully end to end ai, perhaps entire planning may be NN or AI based without any explicit coding.



<img src="https://github.com/harishbalasub/harishbalasub.github.io/blob/master/docs/assets/img/Tesla-fsd-arch.png" width="500" height="250">

### Vision subsystem
Tesla vision uses a Hydranet architecture where the multi-camera image inputs are pre-processed through CNN+Transformer+Spatio-Temporal module and used as inputs for various vision processing tasks.

Lets look at the vision architecture before Tesla started moving to 3D Occupancy network based intermediate vector space

<img src="https://github.com/harishbalasub/harishbalasub.github.io/blob/master/docs/assets/img/Tesla-fsd-arch-vision-2D.png" width="50%" height="50%">

#### Why video module?
Single frame processing is not sufficient to recall temporary occlusions and traffic signs, detect velocity/flow. This requires processing of a sequence of frames (in order of seconds) with a spatial and temporal feature queue. For efficient processing, a small region around the ego car is used and a spatial RNN is run over each block in the region. 

<img src="https://github.com/harishbalasub/harishbalasub.github.io/blob/master/docs/assets/img/Tesla-fsd-arch-vision-video-2D.png" width="400" height="400">

#### Moving to Occupancy Networks
Moving from 2D vector space to 3D Occupancy network provides some advantages

<img src="https://github.com/harishbalasub/harishbalasub.github.io/blob/master/docs/assets/img/Tesla-fsd-arch-2Dvs3D-1.png" >
<img src="https://github.com/harishbalasub/harishbalasub.github.io/blob/master/docs/assets/img/Tesla-fsd-arch-2Dvs3D-2.png" >
<img src="https://github.com/harishbalasub/harishbalasub.github.io/blob/master/docs/assets/img/Tesla-fsd-arch-2Dvs3D-3.png" >

The updated architecture for generating 3D occupancy network and flow is shown below
<img src="https://github.com/harishbalasub/harishbalasub.github.io/blob/master/docs/assets/img/Tesla-fsd-arch-vision-3D.png" >

#### Validation of depth and velocity from vision
The depth and velocity information output vision based multi-camera fusion and video module (blue) has been found to be similar in performance to radar output (green) as shown in the figure below. The single camera based predictions are provided to illustrate the utility of the multi-camera fusion model.

<img src="https://github.com/harishbalasub/harishbalasub.github.io/blob/master/docs/assets/img/Tesla-fsd-arch-vision-vs-radar.png" >

### Planning subsystem
#### Lane guidance module
Detecting lane trajectories in unmarked areas especially intersections is a challenge and doing it in image space is not efficient. Tesla FSD uses autoregressive decoders (decoder stage of a Transfomer) to predict all the lane trajectory options from a given start point along with the spline coefficients.

<img src="https://github.com/harishbalasub/harishbalasub.github.io/blob/master/docs/assets/img/Tesla-fsd-arch-lane-guidance.png" >

<img src="https://github.com/harishbalasub/harishbalasub.github.io/blob/master/docs/assets/img/Tesla-fsd-arch-lane-language-2.png" >


### Control subsystem
Cars like GM use model predictive control (MPC) for various applications like adaptive cruise control, lane-keeping, lane following etc. MPC is useful to match predicted trajectory with actual trajectory.


## Training and Simulation
### Need for end to end simulation
Before every firmware update, demonstrating vehicle safety over several million miles of various driving scenarios can be desired. Modular testing of individual modules not sufficient. Use of graphics engines to create synthetic driving environments does not recreate realistic real-world situations.
To recreate real-world situations closer to reality, a more augmented reality simulation is required. The concept of Neural Radial Fields (Nerf) allows recreating 3D scene from set of 2D images captured during a test drive. This allows creating sensor data which can be trained to recreate photo-realistic camera images.

<img src="https://github.com/harishbalasub/harishbalasub.github.io/blob/master/docs/assets/img/Tesla-e2e-neural-sim.png" >
<img src="https://github.com/harishbalasub/harishbalasub.github.io/blob/master/docs/assets/img/Tesla-e2e-nerfs.png" >

End to End simulation enables
* Scene reconstruction (pick a problematic intersection in a city, add in weather conditions etc)
* Object reconstruction & retrieval (add/remove vehicles, pedestrians etc)
* Traffic modelling
* Sensor simulation
* Adversarial scenario generation

## Risks and Challenges
Videos

## References
