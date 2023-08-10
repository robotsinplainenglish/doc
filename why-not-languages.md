# Why existing languages are no good

We describe the problematic state of today's automation infrastructure.

The person automating a workflow is usually different from the experts
who program the various equipment, for example, the robot vendors. Let
us call this person the "end user".

The end user usually understands the particular vertical within which
some workflows need to be automated, for example, warehouse
management, welding, machine shops, food preparation, etc. But the end
user usually knows very little about the internals of the automation
equipment that needs to be programmed.

Usually, some system integrator or "solution provider" with expertise
in that area needs to build an application with an easy-to-use user
interface specific to the particular vertical, so that the end user
can rapidly develop, modify, and maintain a semi-automated workflow.

But these solution providers usually have limited software development
capabilities. They have to interface with diverse equipment using
their APIs and develop an application with a user interface.

Today, these solution providers have two kinds of languages available:

* Commercial robots with their built-in programming languages.

* PLCs with their standardized programming languages (IEC 61131-3).

Neither of these provide a good automation application development
platform.


## The problem with robot languages

Programming an industrial robot takes a lot of expertise today.
That's because (a) robots are challenging to program, and (b) the
robot must be made part of an automation application. It must be
integrated with different types of equipment made by multiple vendors:
grippers, PLCs, welding machines, gluing guns, conveyors, and so on.

When we talk about "programming a robot," we really mean developing an
automation application orchestrating all this equipment. Although
there are industrial standards (many of them!), those cover only the
most basic level of communication. Instead, robot vendors have
published some kinds of APIs and formed large *ecosystems* of
equipment makers who develop and test interoperation where it makes
sense.

The automation application can either be programmed using an
industrial PLC, which handles most of the orchestration and leaves the
robot to play the role of a puppet, or the automation application can
be programmed using the robot controller, in which case the robot
program must orchestrate multiple pieces of equipment. Most often the
role of the robot controller is somewhere between these two extremes.


### Commercial robot programming languages

Most industrial robot vendors have developed their own special-purpose
languages:

> |  Vendor        | Language  |
> |--------------- | --------- |
> |  FANUC         | KAREL     |
> |  ABB           | RAPID     |
> |  KUKA          | KRL       |

The user writes a robot application as a source text document in the
vendor's language.  A special *interpreter,* supplied by the vendor,
runs on the controller. This interpreter reads the source document and
executes the robot application.

Most of these languages proceed as single-threaded, sequential
programs, with sub-programs like regular programming languages.  But
they also define some syntax for defining handlers for errors and
interrupts, and for freeing up resources when destroying the call
stack:

  * Functions (programs that return values) and procedures (which
    don't) can be declared and called from other programs. The
    parameters and local variables are tracked using a stack, just
    like regular programming languages.

  * Keywords like `INTERRUPT` and `ERROR` can introduce programs that
    will be run when an interrupt or an error condition happens,
    respectively. In the program, you can enable or disable interrupt
    detection. When an interrupt is enabled, an external condition
    like a digital input will directly enter the handler program,
    then resume processing from where the main program left off.

  * ABB RAPID also has a special `UNDO` keyword that can be used to
    set up resource freeing code like C++ destructors. An `EXIT` out
    of a deeply nested routine in the call stack will unwind all the
    `UNDO` routines and free up resources like open files.



### Kinematic planning and control limitations encoded in language

For generating motion trajectories, vendors have developed their
algorithms for their own mechanical arms. These algorithms were
developed a long time ago to deal with a certain mechanical
configuration.  Most industrial robots have six degrees of
freedom. Moreover, a vendor tries to follow a consistent scheme for
their own robot model families so that the same software can be used
to control all their robots in the same family.

The robot motion algorithms have numerous historical limitations,
which in turn become crystallized in the language syntax.  Personnel
who deploy these robots have grown accustomed to their peculiarities.
The large installed base tends to prevent radical changes to the
language, which locks in the underlying algorithms. More modern,
numerical algorithms have been developed without these limitations,
but the vendors see no reason to spend development effort to adopt
them and break backward compatibility with the language.

As a result, the vendor languages tend to remain unchanged for
decades. As long as the same community keeps using the robots, this is
not a problem, but it forms a barrier for new robot users learning the
language.

If vendors were to adopt the newer algorithms for trajectory planning,
inverse kinematics, and the like, then they would be able to support a
much larger family of robots: kinematic chains with more or fewer
degrees of freedom, mobile robots, external devices like motion
tables, conveyors, and rails. They would also be able to more robustly
deal with task and kinematic redundancies.

Some examples show how the robot languages have developed with
assumptions about the current algorithms already incorporated inside
them.

* Conventions for resolving poses and avoiding kinematic singularities
  * KUKA KRL commands that use the robot pose as an argument, add two
    three-bit arguments called `STATUS` and `TURN`. These encode
    particular conventions for resolving ambiguities in how joint
    angles are specified.

  * Numerous "system variables" are defined in KUKA KRL and ABB RAPID
    that encode additional conventions for how singularities are to be
    avoided.  For example, the system variable `$SINGUL_POS[1]`
    encodes how to resolve a singularity called "overhead singularity"
    in KUKA's 6-axis arms driven via KRL.

  
### Issues with domain-specific languages

These domain-specific languages are deeply integrated into the
vendors' motion planning and control algorithms and into their I/O
handling.

The users of these languages are expected to spend some time learning
these languages. While they are not professional programmers, they are
expected to learn the syntax and the specific way in which:

  * Variables encode the state of the robot; and
  * Keywords manipulate the resources of the robot.

Two key problems are:

1. **Non-portable:** Because of individual products' peculiarities,
the knowledge of one language is not directly transferable to other
vendors' equipment. This is perhaps good for individual vendors, but
it inhibits the widespread knowledge of robot programming and
impoverishes the ecosystem.

2. **Stuck in the past:** Vendors find it difficult to incorporate new
algorithms, particularly for motion and trajectory planning and for
orchstration of external equipment.  The existing languages have
explicit support for older limitations and conventions. It is
difficult to modify the language interpreter without breaking lots of
existing automation applications.


## PLCs

The industry needs an infrastructure for developing automation
applications. This was originally the purpose of the standardization
of PLCs and the standardized PLC languages like ladder logic diagrams
and related languages (IEC 61131-3).

PLCs can handle simple, ad-hoc automation, but they are inadequate for
automating workflows for the end user. Also, the languages that have
been standardized are not suitable for end users and are too
low-level.


## Desired platform

What I would like to see is a platform with some tooling that could be
used by solution providers to build a vertical-specific automation
application.

Such a platform should be:

* Easy to extend to use new automation equipment.

* Easy to program using languages that the automation solution
  providers are comfortable with.

* Easy to deploy and manage in the field.

Most likely, such a platform would be some sort of real-time operating
system that can interface with domain-specific robot controllers and
with real-time equipment.

