# FDC - Non-linear frame deformation calibration and compensation 2.0
### For 3D printers running Klipper

## Credits
* This project lies upon the hard work and dedication of [Deutherius](https://github.com/Deutherius), [alchemyEngine](https://github.com/alchemyEngine) and [tanaes]( https://github.com/tanaes)
* Although not involved in this specific project, most of the heavy lifting was done by them and most of the code in this project was writen by them.
* If there is someone I didn't credit it is only by mistake, please let me know!

## Why do you need it?
If you suffer from any of the following:
1) You've tried Virtual Gantry Backers and / or z_thermal_adjust and it made an improvement but didn't fully fix your problem
2) Nicely laid first layer, but second layer is too high / too low and looks like crap
3) Z offset changes between prints (too high / too low when the printer is heat soaked, but too low when it is not, or vice versa)
4) Bed mesh doesn't seem to work well on full plates
5) Have to heat soak the printer for hours just for the above problems to disappear

## Why 2.0?
* I consider VGB + measure_thermal_behavior + klipper's z_thermal_adjust to be v1.0
* 1.0 works well for a lot of people, but it's because the diff between the needed value and the generated linear value is pretty close.
* For printers that are bigger / hotter / weaker or just unlucky, linear compensation is not enough.
* As I learned while trying to fix my top layers, frame deformation isn't linear, and it's printer specific. 
* Furthermore, the need to measure the changes to the mesh and the changes to the z height where double the time it needs to be
* Hence - FDC
* ![image](https://user-images.githubusercontent.com/6442378/206245509-7aa45f54-f028-4fa7-9ada-b1f44663651c.png)
* The picture shows the Z height changes per temperature, in the middle of the bed

## What does it do?
1. Measure changes in bed mesh and z height for x time
2. Generate a non-linear series of bed meshes and z height changes with linear changes in between data points
3. Dynamically adjust z height using the current z_thermal_adjust module to create a non-linear change
4. Dynamically switches bed meshes with the corresponding z height per temperature min, max and step
5. Generate a cool graph that will show you the frame non-linear behavior
6. Currently, only works in between the captured temps!
   1. So make sure the start really cold and time the test to finish with the hottest frame possible
   2. If you start the print with a lower temp then temp_min (or above temp_max) it will never change the z height
   3. If the frame temp goes above temp_max it will stop adjusting (but keep current adjustment)
   4. See roadmap

### It WILL work for "linear deformed frames", so it's not VBG or FDC, FDC is an improved version

## Roadmap 
1. Research different bed temps to see if there is a difference in the way things scale
   1. Implement multi bed temp support if needed
2. Generate non-linear equation to produce more points below and above captured temps
3. Save results to disk for every new mesh
   1. Currently, only saves at the end

## Want to understand more?
Check out the following repositories:

[Gantry bowing-induced Z-offset correction through relative reference index](https://github.com/Deutherius/Gantry-bowing-induced-Z-offset-correction-through-relative-reference-index), to fix that inconsistent Z offset between heatsoaked and not-as-well heatsoaked printer (pairs nicely with virtual gantry backers, but do this one first)

[Virtual Gantry Backers](https://github.com/Deutherius/VGB), a dynamic mesh loading system that counteracts gantry bowing due to bimetallic thermal expansion of gantry members

[measure_thermal_behavior](https://github.com/alchemyEngine/measure_thermal_behavior), This script runs a series of defined homing and probing routines designed to characterize how the perceived Z height of the printer changes as the printer frame heats up. It does this by interfacing with the Moonraker API, so you will need to ensure you have Moonraker running.

## How to run 
0. You are going to need a frame temp sensor! has to physically touch the frame!
   1. Also python 3.7+ (3.7+ as ordered dicts by default)
   2. Improve the speed of your probing and disable fade -  long probe sequences will captrued a distorted bed mesh due to the fast warming up of the bed and frame
      1. For our purposes, a quick probe is usually sufficient. Below are some suggested settings:
      2. Keep in mind - There is a real problem in the start of the test where the frame temps rise up really fast, that casus the mesh we captured to be distorted if the mesh takes too long

```
[probe]
...
speed: 10.0
lift_speed: 10.0
samples: 1
samples_result: median
sample_retract_dist: 1.5
samples_tolerance: 0.05
samples_tolerance_retries: 10

[bed_mesh]
...
speed: 500
horizontal_move_z: 10
fade_start: 1.0 
fade_end: 0
```
1. Enable z_thermal_adjust in your config with temp_coeff=0
   1. Remove VBG if you have it
2. Edit measure_thermal_behavior.py and change the required parameters.
   1. It is recommended that the bed temp will be your working bed temp, if you print ABS and PETG and require different bed temps there is a chance that the meshes will be different. A more robust version that support multiple bed temps will be made in the future
   2. You want to let it run as much as possible until the printer frame temperature reaches the highest temp you've seen during a long print
   3. Currently, the script will not generate bed meshes and z heights above and below the captured temperatures due to its non-linear behavior 
3. Make sure the frame is at the lowest temperature possible (like after it was idle for a night)
4. If you have any fans / nevermore, start them to simulate the same wind you going to have in the enclosure during a print
5. Run nohup python3 measure_thermal_behavior.py temperature_step> out.txt &
   1. temperature_step = the step accuracy in degree Celsius, default to 0.1
6. restart without saving the config to remove all the bed meshes, they are there to save the progress as a recovery option, you don't need them if you got a full json file
   1. If you saved the config that's alright, you can manually delete the meshes later
7. Take the output json file and run generate_FDC_meshes_z_heights.py json_file temperature_step
   1. Run it on your local PC
8. Copy the generated mesh from the new cfg file and paste it at the bottom of your printer.cfg
9. Copy the macro FDC.cfg to the same folder as printer.cfg
10. Edit the macro with the min max temp, step and z_height_temps dictionary that was printed when you ran the script
    1. variable_precision is the precision of step. ie - 0.1 step is 1, 0.05 is 2, 1 is 0
11. Add [include FDC.cfg] to your printer.cfg
12. Save + Restart

### Contact
You can dm me on discord if you have any issues, i'm on the Voron and Ratrig servers
I don't want to put the user name here to avoid bot spamming, but search for the github link and you'll find me