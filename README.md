LAMA - Location Aware Movement API
====================================

This API provides *persistent* position and facing awareness for [ComputerCraft][] [turtles][]. You can query a turtle's position and facing at any time, and it will not 'desynchronize' if the turtle is unloaded (game quit / chunk unloaded) while it moves. To achieve this, it offers replacements for the default turtle movement functions, i.e. for `turtle.forward()`, `turtle.back()`, `turtle.up()`, `turtle.down()`, `turtle.turnLeft()` and `turtle.turnRight()`, as well as one for refueling, i.e. for `turtle.refuel()`. It also provides a few more high level movement functions to travel multiple blocks, as well as a waypoint system to store coordinates and refer to them by name.

With this API you can write *resumable* programs that make decisions based on the turtle's position. For example, saving the following program as the `startup` file will make the turtle run in circles until it runs out of fuel (precondition: turtle starts at `x = 0, y = 0`). It will not leave its track even if the turtle is forcibly shut down at any point.

```lua
os.loadAPI("lama")
while true do
    local x, y, z = lama.get()
    if x == 0 and z == 0 then
        lama.turn(lama.side.north)
    elseif x == 0 and z == -2 then
        lama.turn(lama.side.east)
    elseif x == 2 and z == -2 then
        lama.turn(lama.side.south)
    elseif x == 2 and z == 0 then
        lama.turn(lama.side.west)
    end
    lama.forward(math.huge) -- Keep trying.
end
```

Requirements
============

This API only works on turtles that have an ID and that use fuel. The ID is required for state persistence, the fuel is used for checking whether the turtle finished a move that was interrupted by a forced shut down (game quit / chunk unloaded).

**Important:** when using this API, you *must not* use any of the original movement related functions, since that would invalidate the API's internal state (coordinates would no longer match), neither the original refuel function, which would also invalidate the state. Always use the functions of this API to move and refuel the turtle. You can use `lama.hijackTurtleAPI()` to have the functions in the turtle API replaced with wrappers for this API's functions. The native functions won't be touched; don't use them either way.

Recommended
-----------

The API will create a startup file while performing a move, to allow completion of multi-try moves after reboot. If there's an original startup file, it will be backed up and after API initialization/move completion will be restored an executed. This obviously introduces additional file i/o overhead for each move. To avoid that, it is highly recommended to use some multi-startup-script solution, such as [Forairan's init-scripts][forairan] startup file or [my startup API][startup]. If present, LAMA will instead create a startup script for the multi-script-environment once, with no additional file i/o overhead when moving.

If you'd like to see compatibility with other multi-startup-script systems (I have not dabbled with custom OSs much, yet), let me know - or even better: implement it yourself and submit a pull request. It's pretty easy to add more, as long as there's a way to detect the used script/system. Should you wish to give it a go, search for `local startupHandlers`, which is the table containing the logic for different startup handlers.

Installation
============

To install the API, first run:

```> pastebin get  lama-installer```

which will give you the [installer](installer). After it downloaded, run it:

```> lama-installer```

It will fetch the API and, if so desired the `lama-conf` program as well as an example program (it will interactively ask you). Note that the installer will *not* self-destruct, so you can use it to update your installation.


This includes only `lama-conf`, but *not* the example program. Note that you will have to use `os.loadAPI("/bin/lama-lib/lama")` in this case (in particular, you'll have to adjust that call in the example program).

In both cases you will get a minified version, which is roughly only a third the size of the [original version](lama). Note the the custom installer will provide you with the option to download the full source version, too.

API
===

This list is just meant to provide a quick overview of the available functionality. Please see the [actual code](lama) for full documentation. It is rather well commented. While you're at it, have a look at the `lama.reason` table to see how the movement functions may fail.

Constants
-------

* `lama.version`  
    A string representing the current API version.
* `lama.side`  
    A table of constants used to represent the direction a turtle is facing.
* `lama.reason`  
    A table of constants used to indicate failure reasons when trying to move.

State
-----

* `lama.get()  -> x, y, z, facing`  
    Get the current position and facing of the turtle.
* `lama.getX() -> number`  
    Get the current X position of the turtle.
* `lama.getZ() -> number`  
    Get the current Y position of the turtle.
* `lama.getY() -> number`  
    Get the current Z position of the turtle.
* `lama.getPosition() -> vector`  
    Get the current position of the turtle as a vector.
* `lama.getFacing()   -> lama.side`  
   Get the current facing of the turtle.
* `lama.set(x, y, z, facing) -> x, y, z, facing`  
   Set the current position and facing of the turtle.

Movement
--------

* `lama.forward(tries, aggressive) -> boolean, lama.reason`  
   Replacement for `turtle.forward()`.
* `lama.back(tries) -> boolean, lama.reason`  
    Replacement for `turtle.back()`.
* `lama.up(tries, aggressive) -> boolean, lama.reason`  
    Replacement for `turtle.up()`.
* `lama.down(tries, aggressive) -> boolean, lama.reason`  
    Replacement for `turtle.down()`.
* `lama.moveto(x, y, z, facing, tries, aggressive, longestFirst) -> boolean, lama.reason`  
    Makes the turtle move to the specified coordinates or waypoint. Will continue across reboots.
* `lama.navigate(path, tries, aggressive, longestFirst) -> boolean, lama.reason`  
    Makes the turtle move along the specified path of coordinates and/or waypoints. Will continue across reboots.

The parameters `tries` and `aggressive` for the movement functions do the following: if `tries` is specified, the turtle will try as many times to remove any obstruction it runs into before failing. If `aggressive` is true in addition to that, the turtle will not only try to dig out blocks but also attack entities that stand in its way.

Regarding `lama.navigate()`, note that the facing of all non-terminal path nodes will be ignored to avoid unnecessary turning along the way.

Rotation
--------

* `lama.turnRight() -> boolean`  
    Replacement for `turtle.turnRight()`.
* `lama.turnLeft() -> boolean`  
    Replacement for `turtle.turnRight()`.
* `lama.turnAround() -> boolean`  
    Turns the turtle around.
* `lama.turn(towards) -> boolean`  
    Turns the turtle to face in the specified direction.

Note that all turning functions behave like the original ones in that they return immediately if they fail (they can only fail if the VM's event queue is full). Otherwise they're guaranteed to complete the operation successfully. For `lama.turn()` and `lama.turnAround()` it is possible that one of two needed turn commands has already been issued, when failing. The internal state will represent that however, i.e. `lama.getFacing()` will still be correct.

Refueling
--------

* `lama.refuel(count) -> boolean`  
    Replacement for `turtle.refuel()`.

This has to be called instead of `turtle.refuel()` to ensure the API's state validity, since it uses the fuel level to check if it's in an unexpected state after rebooting. It is is otherwise functionally equivalent to `turtle.refuel()`.

Waypoints
---------

In some cases it may be easier (and more readable) to send your turtles to predefined, named locations, instead of some set of coordinates. That's what waypoints are for. They can be manipulated like so:

* `lama.waypoint.add(name, x, y, z, facing) -> boolean`  
    Adds a new waypoint or updates an existing one with the same name.
* `lama.waypoint.remove(name) -> boolean`  
    Removes a waypoint from the list of known waypoints.
* `lama.waypoint.exists(name) -> boolean`  
    Tests if a waypoint with the specified name exists.
* `lama.waypoint.get(name) -> x, y, z, facing`  
    Returns the coordinates of the specified waypoint.
* `lama.waypoint.iter() -> function`  
    Returns an iterator function over all known waypoints.
* `lama.waypoint.moveto(name, tries, aggressive, longestFirst) -> boolean, lama.reason`  
    This is like `lama.moveto()` except that it takes a waypoint as the target.

Note that waypoints *may* have a facing associated with them, but don't have to. This means that if a waypoint has a facing and the turtle is ordered to move to it, it will rotate into that direction, if it does not the turtle will remain with the orientation in which it arrived at the waypoint.

Utility
-------

* `lama.init()`  
    Can be used to ensure the API has been initialized without any side-effects. If this is the first call and the API has to be initialized because of that, this will block until any pending moves are completed.
* `lama.startupResult() -> boolean, lama.reason`  
    Can be used in resumable programs to check whether a move command that was issued just before the turtle was forcibly shut down completed successfully or not. You will need this if you make decisions based on the result of the movement's success (e.g. stop the program if the turtle fails to move).
* `lama.hijackTurtleAPI(restore)`  
    Replaces the original movement functions in the turtle API with wrappers for this API. Note that the wrappers have the same signature and behavior as the original movement functions. For example, calling the wrapper `turtle.forward(5)` will *not* make the turtle dig through obstructions! This is to make sure existing programs will not behave differently after injecting the wrappers. The original functions can be restored by passing `true` as this functions parameter. **Important:** the changes to the turtle API will be global.

Additional files
================

**lama-conf** can be used to query the current position and facing (`lama-conf get`), as well as set it (`lama-conf set x y z facing`) e.g. for calibrating newly placed turtles. It can also be used to manage waypoints (`lama-conf add name x y z [facing]`, `lama-conf remove name` and `lama-conf list`).

**lama-example** is a small example program, demonstrating how to use the API in a resumable program. It allows ordering the turtle to move along a path of waypoints.

Limitations
===========

If the turtle is moved by some external force (player pickaxing it and placing it somewhere else, RP2 frames, ...) the coordinates will obviously be wrong, because we have no way of tracking such influences (aside from GPS, perhaps - might look into that for roughly validating coordinates in some future version).

All multiblock movement will only perform straight moves, i.e. turtles will never try to evade obstacles, only break through them, if allowed. For now, I feel that pathfinding is beyond the scope of this API, so if you need it you'll have to build it on top of the API. The main reason I don't feel like this belongs in here is because it would quickly get out of hand, since you'd then have to keep track of fuel (because usage cannot be precomputed at the time the order is given) and possibly even map out the region as you go.

Closing remarks
===============

As far as I can tell, this is the first API using the fuel workaround. I'm quite sure this is the only truly working solution out there for properly keeping track of a turtle's position across forced shut downs (aside from GPS, which can be inaccurate). Two common methods I saw I can invalidate right away:

* Saving the state before and after `turtle.forward()` can fail because that call yields, and the turtle may be shut down during that yield, so you may never know whether it was successful or not.
* Saving after `turtle.native.forward()` and after successfully waiting for the result may fail if the turtle is shut down while waiting, since, again you will never know whether the move was successful, because the `turtle_response` event will not be sent after the reboot, see [my thread on the ComputerCraft forums on this issue][event post].

If you know of another method, I'd be intrigued to hear about it, though!

I have tested the robustness of the API using the following method:

- Have the turtle move randomly and continually in a 4x4x4 cube (it'd pick a new target when reaching one).
- Drop sand stacks into the turtle's path.
- Stand in the turtle's way.
- Have some bedrock in the cube.
- Have a mob in the cube.
- Quit the game over and over
- Resetting the turtle manually (Ctrl+R) to interrupt the program at different code locations, hopefully having hit each one at least once.


Still, this is software, and testing is a tricky business, so it's very possible for some interesting bugs to remain. If you come across any (and are sure it's not your fault) please let me know, thanks!

Changelog
=========

- Version 0.1
  - created api.

License
=======

This API is licensed under the [GNU][license], which basically means you can do with it as you please.


[computercraft]: http://www.computercraft.info/
[NinjaRacc]: https://github.com/NinjaRacc
[license]: https://www.gnu.org/licenses/gpl-3.0.en.html
[turtles]: http://www.computercraft.info/wiki/Turtle
