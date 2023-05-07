# Trickshots

[Blender](https://www.blender.org/) add-on for executing trickshots using physics simulation. You write a short scenario and this Python package will run numerous simulations to find the right scene parameters to fulfill the scenario. It is a fast way to win a game called dude, perfect!

- No fakes - physics is simulated with Blender for real.
- Unique solver allows creating variables and optimization tasks as scenario unfolds.
- Can handle any scene parameters.
- Includes convenience functions for setting initial object velocity and detecting scenario events.

You can use this package for motion design, robotics, education and who knows what else.


### Features

For the solver the simulation is a black box as if it was the real world: no derivatives, no exact collisions, poor repeatability. 

- Check if objects had a collision between frames  
- Track velocities and accelerations  
- Creating variables during scenario execution  
- Setting optimization tasks during scenario execution  
- Flexible scenario structure with yield  
- Tasks math: bool, or, and  
- Triggers - remembers if the watched event happened  
- Minimize an arbitrary expression for guiding the solver  
- Supply a callback to modify the scene or use it to early exit the sim  
- Extend the code by defining your own tasks  


### Supported Blender versions

Supported: 3.3, 3.4, 3.5, 3.6  
OS: Windows, Linux, MacOS  
Not supported: 2.93 and earlier


### Installation

In Blender go to Edit -> Preferences -> Add-ons tab, press Install button and select trickshots.zip.


### Tutorial

Please read [the full manual](manual.md). And here's just a small example:

```python
def scenario():
    set_vel('ball', var3(-10, 10))
    yield hit('ball', 'box1')
    yield hit('ball', 'box2')
    yield hit('ball', 'box3')

solve(scenario)
```

This short piece of code is enough for animating the shot:

<video src="bouncyball.mp4" controls style="max-width: 450px;" loop autoplay>
</video>

More examples of Blender scenes together with corresponding scenarios written in Python are available [here](https://github.com/mikhail-matrosov/trickshots-examples).
