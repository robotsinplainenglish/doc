# Command Language Tutorial

## Microcontroller-based real-time behavior engine (RTBE)

We describe our simple command language used in the hackathon for the
real-time behavior engine (RTBE). This engine is a substitute for the
Intera behavior engine.

The real-time behavior engine (RTBE) runs on a 32-bit microcontroller
with no operating system, so it's a really simple program. The engine
doesn't actually do anything; it just invokes a modified version of
the NodeJS behavior engine on the controller PC to do the actual work,
and monitors it as it runs.

What the RTBE offers is a way to orchestrate real-time tasks. For
example, if you want a task to respond to a GPIO sensor signal and
immediately make a change, then the Intera engine can't do it, because
it is scheduled as a best-effort process on Linux and communicates
using ROS. But the RTBE can.

To invoke long-running move commands, the RTBE can initiate the
commands by communicating with the NodeJS engine. It can monitor the
progress of a command and react to changed states. These commands
won't be real-time, just like the regular Intera NodeJS engine.

The RTBE runs at 1 kHz. We could easily run it at ten times this
speed, but we don't actually have any real-time tasks that need that
kind of response time. So, the RTBE is sleeping most of the time.

The RTBE reads and writes to a serial line over a UART. This is how
you communicate with it.


## Language for RTBE

You send ASCII strings to the RTBE over the serial line, and the RTBE
responds over the same line. The RTBE invokes predefined move commands
on the controller.

```
  you --> serial line <--> RTBE on microcontroller

  RTBE on microcontroller <--> robot controller PC
```

The language we present here is very easy to use for moving the robot
and for querying its current state.

Here's an example of how to move the robot.

### Example: How to move the robot

Say the robot is at the home position, and we'd like to move it to a
certain hovering position above a part in preparation for picking up
the part.

The RTBE defines a "command block" of type joint move, `R0JMOVE`. This
command block has a fixed structure:

* A parameter for a destination pose, i.e., the seven positions of
  the seven Sawyer joints `J0..J6`. It can move the robot to that pose.
* A list of states that it will pass through as the command executes.

You create an **action** called `GO_HOVER` that writes the parameter
of this command block and monitors the states.

```
: ACTION GO_HOVER R0JMOVE  FLOATARRAY 7 0 0 0.15 0.25 0 0 0 ;
```

The above definition, which starts with a colon symbol `:` and ends
with a semicolon `;`, defines an **action** called `GO_HOVER`. The
action provides the `R0JMOVE` command block with the parameter that it
requires, as an array of 7 floating-point numbers.

For convenience, you can assign the seven joint positions to a
**variable** named `HOVERJ`, then use that in defining the action:

```
: VAR HOVERJ FLOATARRAY 7 0 0 0.15 0.25 0 0 0 ;
: ACTION GO_HOVER R0JMOVE HOVERJ ;
```

The above lines are effectively the same as the single line further
up.  The array of seven numbers is still supplied to the action
`GO_HOVER` as a parameter. (The assignment is by value; if you modify
the variable `HOVERJ` after this, it won't affect the value of the
parameter).

You could either type these numbers in yourself, or you could put the
arm in zero-G and move it to the desired pose, then issue a `GET`
command (see manual) to get the current value of the robot's joint
positions.

At this point, you have created the action that you can execute to
move Sawyer to the hover position. Now to execute the action, you do
the following.

There are two **predefined states** in the RTBE:

* `$HOME`, true when the robot is at the home position.
* `$START`, a built-in monostable state used to trigger
  behaviors. ("Monostable" means that it is normally false, and when
  you set it to true, it stays true only for one tick and then returns
  to false).

Both the above state names start with `$`. You are not allowed to name
your states or actions to start with `$`.

To move the robot, you command the RTBE to execute this action by
sending it a string over the serial line.

```
SET $START 1
```

Whenver you create a robot action, RTBE by default sets the action to
trigger whenever the `$START` state is true.

When the RTBE receives the above message, it sets the monostable
`$START` to 1 (true), which triggers the action `GO_HOVER`. Then the
`$START` state becomes 0 (false) again.

The RTBE performs the `GO_HOVER` action by moving the robot. When the
motion executes, the command block updates a series of states
described in the RTBE manual. Those states are:

```
  R0JMOVE.STARTED R0JMOVE.RUNNING R0JMOVE.SUCCESS R0JMOVE.ERROR
```

Each of these states are 0 (false) to start with, but as soon as the `GO_HOVER` action starts execution, they are each set to 1 (true) in turn. If the move succeeds, then `R0JMOVE.SUCCESS` is set to 1. If there's an error, then `R0JMOVE.ERROR` is set to 1 instead.

You can monitor the changing states at the serial line, as the RTBE
sends messages whenever there are state changes.

```
R0JMOVE.STARTED 1
R0JMOVE.STARTED 0
R0JMOVE.RUNNING 1
R0JMOVE.RUNNING 0
R0JMOVE.SUCCESS 1
R0JMOVE.SUCCESS 0
```

The state `STARTED` becomes true (1) as soon as the action is
initiated. It stays at 1 until the RTBE successfully communicates with
the Intera engine and starts the joint move. Then it becomes 0, and
the `RUNNING` state becomes 1. It stays 1 until the RTBE hears back
that the move has succeeded, when `SUCCESS` becomes 1.

The `SUCCESS` state is different: it is a monostable state, which
automatically falls back to 0 on the next tick after it turns 1.

We can configure the RTBE, of course, to send these monitoring
messages over a different line if it's convenient, so that they won't
interfere with your commands and responses. We didn't build that
facility for the prototype.


## Summary: communication over serial line

The above description shows how to move Sawyer using a joint move, and
monitor its state during execution.

You can create or query variables, states, or actions in the RTBE by
sending it commands over the serial line. You can execute actions by
triggering them, usually by specifying that they should run when the
built-in `$START` monostable state is true. This has been done by
default, so we didn't need to show it above.


# State Language

Now let's talk about the RTBE and how it represents variables, states,
and actions. It holds two databases: the **configuration** and the
**blackboard**.

## Configuration

The configuration of the controller is the description of the I/O
(MODBUS, Ethernet/IP, etc.), robot kinematic model and other
configuration, safety devices, etc. attached to the controller. The
configuration is the data that remains stable for a long time, at
least hours, usually for years.

The controller configuration is represented by a list of variables
with values. Values can be:

* integer: signed 32-bit integer
* float: single-precision IEEE floating point
* boolean: 0 or 1
* string: ASCII characters, maximum length 254
* a fixed-length array of one of the above types

The variable names are ASCII with maximum length 24.


## Blackboard

Then there's a **blackboard**, which is data that changes during
operation.  The blackboard is represented by a dictionary containing
variables and their definitions.

Each variable can stand for a **value**, a **state**, or an **action**.

You can query the entire configuration of the controller or a selected
sub-set at any time. Similarly, you can also query the entire
blackboard or any sub-set of variables at any time using the `GET`
command.  Of course, some of the values in the blackboard might change
soon after you have queried them.

Some of the variables in the blackboard start with `$`. These are
**predefined** variables. For example, the various coordinate frames
and robot joint positions in world coordinates:

* `$R0JOINT`: the current joint positions of the main robot. Array of
  7 floats, J0-J6.
* `$R0FLANGE`: the current center point of the main robot flange.
  Array of 6 floats, X, Y, Z, Rx, Ry, Rz.
* `$R0BASE`: the current base coordinates of the main robot. Array of
  6 floats, X, Y, Z, Rx, Ry, Rz.
* `$R0CURRTCP`: the current tool center point of the main robot,
  relative to the center point of the flange.  Array of 6 floats, X,
  Y, Z, Rx, Ry, Rz.


## States

In RTBE, the word "state" represents some interesting condition about
the state of the controller, including its I/Os and robot.  For
example:

* Whether or not the robot is in the home position is an interesting
  condition.
* Whether or not the digital input 3 is on is an interesting condition.

Each interesting condition is evaluated (conceptually) at every tick.
If the condition evaluates to true, it means the controller is in that
state.

The controller is in several concurrent states at any time. For
example, the robot might be in `HOME` position, and at the same time,
digital input 3 might be `OFF`. These states are concurrently true.


## Predefined states

Depending on configuration, the controller maintains a fixed number of
**predefined states,** each either true or false.

For example, by default a `MOXA` Modbus TCP gateway has 8 digital
input ports, and each one can be `OFF` or `ON` at any given time.  The
RTBE represents the possible states of these input ports as:

* Port 0: `$DI0OFF` and `$DI0ON`
* Port 1: `$DI1OFF` and `$DI1ON`
* *(etc.)*

This makes 16 predefined states total, of which 8 are true and 8 are
false at any given time.

Similarly, there are states for:

* `$R0SAFETYSTOP`: Is the main robot in safety-stopped state?
* `$R0MANUALENABLED`: Is the main robot being manually manipulated?

The full set of states is listed in the RTBE sources.


## State predicates

Any combination of states of the controller can be described by a
*predicate*, i.e., by a yes/no question.  A predicate is composed of a
boolean (logical) expression that can combine together many states,
for example, "is the robot in the home position, `AND` is the digital
input 1 in the `OFF` state?"  A blackboard variable can have as its
definition such a predicate.

For example, these are some predefined states, named with a starting
`$` sign for convenience:

* `$R0HOME`: is the main robot in the home position?
* `$DI0OFF`: is the zeroth digital input in the `OFF` state?

And you can combine these to define your own states with names in the
dictionary.

You can also define new states of the robot by specifying its joint
positions:

```
: STATE HOVER R0JOINT 7 0 0 0.15 0.25 0 0 0;
: STATE APPROACH R0JOINT 7 0.411 1.667 0.15 0.25 0.98 0.21 0;
```

The above syntax defines two new states by using the `:` command,
which starts with a colon and ends with a semicolon.  The two new
states are `HOVER` and `APPROACH`. Both are of type R0JOINT, i.e., the
joint positions of the main robot.

You could find these joint values by moving the robot by hand to the
approach position and reading the $R0JOINT variable. You can use
the variable directly in the `:` command:

```
: STATE APPROACH R0JOINT $R0JOINT;

```

(RTBE does not refer to the individual joint angles as states. We
prefer to use only discrete boolean values as states).


## Actions and sequences

An action is something that brings the controller from one state to
another state. Actions are implemented by programs, usually running
outside the RTBE. The most important actions are the ones that invoke
robot motions and the ones that set I/O outputs.

The RTBE presents robot moves as "command blocks"; only one command
block can be active at a time.

Some predefined command blocks:

* R0JMOVE: bring the main robot to a given pose by interpolating
  joint position values.
* R0LINMOVE: bring the main robot TCP to a given cartesian position by
  interpolating cartesian TCP positions, i.e., in a straight line
  path.
* R0SPLMOVE: bring the main robot TCP to a given cartesian position by
  interpolating through a list of given cartesian waypoints, i.e., in a spline
  path that passes through the given waypoints.

The `R0JMOVE` and `R0LINMOVE` command blocks require slightly
different parameters--- joint positions versus cartesian TCP
position. They both expose this set of states:

```
  R0JMOVE.STARTED R0JMOVE.PLANNED
  R0JMOVE.RUNNING R0JMOVE.SUCCESS R0JMOVE.ERROR

  R0LINMOVE.STARTED R0LINMOVE.PLANNED
  R0LINMOVE.RUNNING R0LINMOVE.SUCCESS R0LINMOVE.ERROR
```

The `R0SPLMOVE` command block takes a list of waypoint parameters, and
it provides a list of intermediate states that show which waypoint the command
is currently approaching: `R0SPLMOVE.EXEWAY1`, `R0SPLMOVE.EXEWAY2`, etc.


## Action sequences

You can also define sequences of robot actions, which is an action of type
`SEQUENCE`:

```
: ACTION PREPARE_PICK SEQUENCE 2 HOVER APPROACH;
```

The above `:` statement defines a new action sequence of two robot
moves.

The `PREPARE_PICK` action now means to run the `HOVER` and `APPROACH`
actions in sequence. You can execute this action, at the end of which
the robot will be in `APPROACH` state if successful. Each of the two
actions may be defined using a different command block or the same
command block; in any case, only one of the action can be executed at
one time.

The `SEQUENCE` type action provides a list of intermediate states that
show which action it is in the process of executing:
`PREPARE_PICK.EXE1`, `PREPARE_PICK.EXE2`, etc.




## Orchestration

Instead of commanding RTBE on the serial line while the robot is
moving, you can prepare behaviors that are triggered automatically
when certain states become true. This is called orchestration. You can
use it to program real-time behaviors where the robot reaction needs
to be immediate within a guaranteed time frame.

In general, we have syntax for triggering an action based on states.

The `SEQUENCE` type action described above, is a specific kind of
orchestration, which orchestrates a list of actions one after
another. You can have `PARALLEL` type actions and other useful kinds,
similar to the various kinds of inner nodes in behavior trees.

