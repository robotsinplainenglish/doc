# Command Language Using REST

In the document [Serial Command Language](serial-commands.md), we
described how to use the serial port to send commands and receive
messages back from RTBE.

Instead of using a serial line, we now describe how to use the REST
paradigm to send and receive messages and achieve the same results.

## Using the REST paradigm for commands

We use the REST paradigm to manipulate resources on the controller
using HTTP verbs. Resources can be variables, states, or actions.

* `GET` to query the current resources.
* `POST` to define new resources.
* `PUT` to modify resources or to execute actions.

Since resources can be variables, states, or actions, each of these
can be represented by a URL. We can distinguish between them using a
top-level path, like this:

```
/a/some_action
/s/some_state
/v/some_variable
```

Using `POST`, you can create new resources. Using `PUT`, you can
modify their values, and using `GET`, you can query them.  Both
`POST` and `PUT` can result in side effects where other resources
are modified.

Let us describe how to move the robot using this paradigm.

## Actions are resources; executing actions = PUT

To execute an action, you send a `PUT` command with the name of the
action to execute as the URI.

Say there's already a PTP action defined on the controller called
`GO_HOVER`. This action moves the robot to a predefined location state
called `HOVER`.

We will describe later in the document how to create
this state and this action. But to execute this action, you send
an HTTP `PUT`:

```
PUT /a/GO_HOVER HTTP/1.1
```

The `/a/` prefix means "execute this action".  The action already
exists; with the `PUT` we are merely executing it. (We presumably
created the action resource earlier using `POST`).

Our RTBE program on the controller will check to make sure the robot
can currently execute this action (for example, that it's not in a
safety violation or something like that). Then it will execute this
PTP command and report the results back to us.


## States are resources


### HTTP responses can contain relevant resources

From the `PUT` above, ideally, we should hear back:

```
HTTP/1.1 204 No Content
Content-Location: /s/HOVER
```

This means RTBE successfully executed `GO_HOVER`, and the robot was
left at the `HOVER` state.

Explanation of response:

* The `204 No Content` means that the request was successful and that
  the action `GO_HOVER` already existed (since it was created
  earlier).

* The `Content-Location` header says that there is a state called
  `HOVER` whose value was made true by this action. If you were to
  execute a `GET`, you would find that `HOVER` is now true.


### Can query states

Using `GET`, you can retrieve either all the states:

```
GET /s
```

Or a particular state:

```
GET /s/HOVER
```

The `GET` will return `1` in the message body if the robot is already
located at the hover position, or `0` if it is not.

## Resource linking

The various resources are linked to each other, and our server
indirectly exposes these relationships. When you `PUT` an action, the
corresponding state resources automatically get modified.

The state `HOVER` and the action `GO_HOVER` are linked to each other,
so that if you execute `GO_HOVER` successfully, the state `HOVER`
automatically gets the true value `1`, and if it is unsuccessful, the
state gets the false value `0`.


## Engineering quantities and logical symbols

The resources on a server, i.e., variables, states, and actions, are
linked together in a meaningful way. For example, the `HOVER` state is
intended to be the result of a successful `GO_HOVER` action.

The RTBE maintains its database of variables, states, and actions as a
logical and symbolic set of relationships. This is different from the
typical robot command, which is an engineering operation expressed in
terms of floating-point values of joint positions or TCP frames.

The following is the distinction between an engineering command using
numeric quantities and a logical statement using symbols:

* When you say that a joint move's destination pose is the array `7 0
  0 0 0.2 0 0 0.1`, then you are prescribing the desired action as an
  engineering command.  At the end of this joint move, you could
  presumably measure the joint positions and find that they are
  exactly these values, or close to these values to within some
  tolerance. This is how robot controls software is designed to work.

* When you say that an action called `GO_HOVER` has a successful end
  state of `HOVER`, then you're making a logical statement about these
  two symbols.  When the given action is successful, you can deduce
  that the given state is true, without performing any measurements.

The symbols are connected to the engineering measurements, but the two
are not the same.  The logical symbols provide a more abstract view
than the engineering quantities. You can manipulate the symbolic
representation using higher-level logic. For example, you can specify
orchestration (start this action when that state is true) and provide
easy-to-use interfaces for end users.

The RTBE lets you construct both the symbolic and the numeric views,
and to connect them together.

* **Variables** are symbolic names for resources.  You can refer to
  resources only using these names.  The names can be specified by
  you, but they can also be automatically generated by RTBE if you
  don't specify them. You can change the name for any resource, and
  you can also have two different names that refer to the same
  resource.

* **States** and **Actions** are resources that are related in the
    following way: actions can be triggered based on states, and
    states can change based on the success or failure of actions. Some
    states are read-only, i.e., you cannot directly modify them, for
    example, the current robot state results from actions and their
    success or failure. You can create actions that are
    read-write. Some combinations of built-in actions and states are
    useful, for example the `$START` built-in monostable state: you
    can set it to true, but at the next tick it reverts back to false.

Usually you create combinations of states and their related actions
simultaneously. For example, when you create an action, you name it,
and you also create variables naming the states corresponding to the
various outcomes of the action.

## POST and creation of resources


`POST` requests are used to create variables, states, and actions.
So, a single `POST` can create multiple objects.

