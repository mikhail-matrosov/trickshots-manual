Let's have a closer look at this example:

```python
from trickshots import *

def scenario():
    set_vel('ball', var3(-10, 10))
    yield hit('ball', 'box1')
    yield hit('ball', 'box2')
    yield hit('ball', 'box3')

solve(scenario)
```

This short piece is enough for animating [pitched boxes](https://github.com/mikhail-matrosov/trickshots-examples/):

<video src="bouncyball.mp4" controls style="max-width: 450px;" loop autoplay>
</video>

In the code we define a `scenario` function:

- `set_vel(obj, vector)` Sets initial velocity of object `obj`. The second argument is an actual vector that was produced by the solver in a call of function `var3`.
- `var3(min, max)` Returns a vector of size 3 produced by the solver.
- `hit(obj1, obj2)` Constructs a new task that is `yield`ed to the solver. Only after the task is fulfilled by the solver execution of the scenario is continued from the same line. Note that the solver will only consider the tasks that were yielded from the scenario, one after another. That's why the solver will first aim to hit box1, then box2 and only after that box3. Multiple tasks can also be combined.

Someone else's code is the best way to learn scenarios structure and logic, so please checkout the [examples](https://github.com/mikhail-matrosov/trickshots-examples).


## Scenario flow and execution

When you call `solve(scenario)` the solver will call the `scenario`. The latter should `yield` a task. Once the solver received it's first task it starts physics simulation in Blender by advancing time frame by frame. At each frame the solver checks if the given task is accomplished. When it is done the solver asks for the next task from the `scenario`, the simulation continues from the same frame. After the simulation has reached its last frame (Output properties -> Frame range -> End or `bpy.context.scene.frame_end`) the simulation is restarted and `scenario` is reset as well to be called again from the beginning.


## Callbacks

You can supply a `callback` function to the solver that will be called after each frame. It can modify the scene, print some debugging information or tell the solver if the simulation has diverged. This allows for an early exit from the simulation that can greatly speed up finding a solution for the scenario. Below is an example of such usage: if the callback returns `True` the solver will interrupt the current simulation, reset and try another set of inputs.

```
def escape():
    x, y, z = get_pos('Cube')
    return not ((-5 < x < 5) and (-10 < y < 10) and (-5 < z < 10))

solve(scenario, escape)
```


## Combining tasks

If you need more complicated scenarios you can combine tasks.

Operator `|` (OR) combines two tasks in such a way that both are pursued but fulfilling only one of them is enough.

Operator `&` (AND) combines two tasks in such a way that both are pursued and they both must be `True` simultaneously at the same frame.

`trigger(task)` in cimbination with `&` can be used to define a scenario in which two events happen in any order but both are required to happen:  
```python
yield trigger(hit('cube1', 'target1')) & trigger(hit('cube2', 'target2'))
```

Let's also have a look at the [bottle flip](https://github.com/mikhail-matrosov/trickshots-examples/) example:

<video src="bottle_flip.mp4" controls style="max-width: 450px;" loop autoplay>
</video>

```python
def btl_angle():
    return bottle.matrix_world.to_quaternion().angle

def scenario():
    bottle = bpy.context.scene.objects['bottle']
    set_avel(bottle, [0, -5, 0])  # pt. 1
    set_vel(bottle, var3([0,0,0], [10, 0, 10]))
    yield hit(bottle, 'target')  # pt. 2
    yield least_squares(btl_angle()) & delay(10)  # pt. 3
    yield least_squares(btl_angle()) & hit(bottle, 'trigger')  # pt. 4
    yield least_squares(btl_angle()) & (wait_deactivation(bottle) | delay(100))
    yield hit(bottle, 'trigger')  # pt. 6

def escape():
    x, y, z = get_pos('bottle')
    return not ((-5 < x < 5) and (-5 < y < 5) and (0.5 < z < 10))

solve(scenario, escape)
```

Note how the scenario is decomposed into a sequence of events:

1. We want the bottle to make a flip so set it's angular velocity.
2. The bottle touches a target - a small box on the floor. It is needed to ensure where the first impact point of the bottle would be - right in the center of the floor.
3. After the bottle has landed we'd like to ensure that it wouldn't fall. Signal it's orientation to the solver after 10 frames. The solver will try to keep btl_angle closer to zero (straight vertical).
4. Check that the bottle hits the trigger - a flat shape above the floor. The bottle can touch it only if it stands vertically. Again, straight orientation of the bottle is prefered.
5. Wait for the bottle deactivation, but no more than 100 frames (otherwise it could take too long). After that straight orientation of the bottle is prefered once again.
6. Ensure that the bottle still touches the trigger.

This is not the required sequence of checks for the bottle flip to happen, but the one that demonstrates combining multiple tasks.


## Reference

`delay(frames)` Constructs a Task for the solver to wait for the specified number of frames during the scenario execution.

`get_acc(obj)` Returns the object's acceleration vector as a numpy.Array of size 3, estimated from finite differences. `obj` is either `str` or Blender's Object, e.g. `bpy.data.objects['ball']`.

`get_avel(obj)` Returns the object's angular velocity vector as a numpy.Array of size 3, estimated from finite differences.

`get_pos(obj)` Returns the object's position as a numpy.Array of size 3. It is a shorthand for obj.matrix_world.translation which is different from `obj.location` because the former is updated for each frame.

`get_vel(obj)` Returns the object's velocity vector as a numpy.Array of size 3, estimated from finite differences.

`hit(obj1, obj2)` Constructs a Task for the solver to collide the two objects.

`is_deactivated(obj)` Returns boolean if the object is deactivated - stopped moving and rests in one place. This works independently from Blender's deactivation option of a rigid body.

`least_squares(expr)` Constructs a Task for the solver to minimize sum of squares of the given expression, which might be an array or a single number. The task doesn't force the solver to find an exact minimum and doesn't block the scenario, but the solver will prefer solutions with lower norm of the expression. This can be used to guide the solver to a specific solution. See an example of usage in [bottle flip](https://github.com/mikhail-matrosov/trickshots-examples/)

`set_avel(obj, vector)` Sets the object's angular velocity for the current frame. Vector is an axis-angle velocity in global space.

`set_pos(obj, vector)` Sets the object's position.

`set_vel(obj, vector)` Sets the object's velocity for the current frame.

`set_verbosity(level=1)` Sets solver verbosity, which is from 0 (quiet) to 5 (the most verbose).

`solve(scenario, callback=None, iters=1000, polish_iters=0, retries=10)` Invoke the solver. The solver will make `iters` iterations and retry up to `retries` times in case it wouldn't converge. After the solution is found it will run additional `polish_iters` iterations to find an even better solution. Note that in some cases Blender physics is not reliable enough and these additional iterations will disturb the solution and it won't be reproduced anymore. In such cases don't use polishing. Instead in the Properties panel go to Scene Properties tab -> Rigid Body World -> Cache -> Current Cache to Bake to freeze the found solution right after the solver finished it's work. Returns a Solution object.

`solver.ScenarioLoss` An objects that holds a loss value for the scenario. It is designed in such a way that you can directly compare it by using operator `<` to find which solution is better.
`task_ix` Index number of the last yielded from the scenario task, starting from 0 and counting negative numbers.
`finished` 0 or -1 (if the task was successfully finished).
`loss` Norm of the `err_vec`.
`frame` Current frame number.
`hint` A scalar value to guide the solver in case it stuck on a platoe.
`err_vec` An error vector, numpy.Array of concatenated errors returned by the current task.

`solver.Solution` An objects that holds a found solution. x y iters retries problem  
`x` A numpy.Array of concatenated variables supplied to the scenario.  
`y` A `ScenarioLoss` object for the given `x`.  
`iters` Number of iterations used for solver to converge in the last trial.  
`retries` Number of retries used for solver to converge.

`solver.Task` An object that holds the current task for the solver. State of the task is evaluated for the current frame with `bool()`. It can flip between `True` and `False` depending on the task's terms. Multiple tasks can be combined by using operators `|` and `&`. Note that `&` of two tasks will be `True` only if both tasks are `True` at the same frame during the scenario execution.

`trigger(task)` Constructs a Task that becomes finished (evaluates to `True`) once the `task` successed during scenario execution.

`var(min=None, max=None, name=None)` Defines a scalar variable and returns its value. The solver will remember the point in the scenario where this exact call occurs and will attempt to supply such values that solve the scenario. Name is an optional argument that allows assigning a name to the variable. All calls with this name will return the same variable.

`var3(min=None, max=None, name=None)` Defines a vector variable and returns its value. Arguments are parsed as:  
```
No args: var3()                 -> ([0,0,0], [1,1,1])  
Radius:  var3(3)                -> ([0,0,0], [3,3,3])  
Range:   var3(1, 5)             -> ([1,1,1], [5,5,5])  
Radius:  var3([x,y,z])          -> ([-x,-y,-z], [x,y,z])  
Center:  var3([x,y,z], 1)       -> ([x-1, y-1, z-1], [x+1, y+1, z+1])  
Range:   var3([a,b,c], [d,e,f]) -> ([a,b,c], [d,e,f])
```

`wait_deactivation(obj)` Constructs a Task for the solver to wait until the object obj is deactivated.

