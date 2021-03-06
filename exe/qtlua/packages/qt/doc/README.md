<a name="qtlua.overview.dok"/>
# QtLua and Qt Package Reference Manual #

The package `qt` is available when you run the Lua
interpreter using the [qlua](#qlua) program. 
It provides a seamless interface between the 
lua interpreter and the Qt4 portable user interface libraries. 
Of course, a full understanding of this package `qt` requires 
a minimal familiarity with the Qt4 library. 

Program [qlua](#qlua) relies in fact on 
the [QtLua](#qtlua) C++ library which provides a Qt 
object that encapsulates a Lua interpreter
All properties, slots and signals declared by the C++ Qt objects
are automatically exposed in the Lua interpreter. 
Signals can be connected to Lua functions.

The packages 
[qtcore](..:qtcore:index),
[qtgui](..:qtgui:index), and
[qtuiloader](..:qtuiloader:index)
provide bindings for the classes defined 
by the corresponding Qt libraries.

<a name="qlua"/>
# Program qlua #

Program `qlua` is a compatible replacement for 
the standard interpreter program `lua`
with several new capabilities:

  * New packages provides a rich interface with the Qt4 library: [qt](#qt), [qtwidget](..:qtwidget:index), [qttorch](..:qttorch:index), [qtuiloader](..:qtuiloader:index).

  * The Lua interpreter runs in a dedicated thread. The main thread remains in charge of graphical interface and user interaction. Therefore all graphical interfaces remain active while the interpreter is running.

  * Console output can be captured and emitted as Qt signals.
Therefore one can program a complete integrated development 
environment including a graphical console.

<a name="qlua.usage"/>
## Usage ##

Program `qlua` accepts the same command line options and arguments as the
stand-alone Lua interpreter [lua](..:LuaManual#LuaStandalone).  It also
accepts all the
[Qt command line options](http://doc.trolltech.com/4.4/qapplication.html#QApplication)
as well as a few specific options.

```
Usage: qlua [options] [script <scriptargs>]
The lua options are:
  -e stat       execute string 'stat'
  -l name       require library 'name'
  -i            enter interactive mode after executing 'script'
  -v            show version information
  --            stop handling options
  -             execute stdin and stop handling options
The specific qlua options are:
  -ide          run the qlua integrated environment
  -onethread    run lua in the main thread
  -nographics   disable all the graphical capabilities
The most important Qt options are:
  -style s      set the application gui style 's'
  -session s    restore gui session 's'
  -display d    set the X display (default is $DISPLAY)
  -geometry g   set the main window size and position
  -title s      set the main window title to 's'
  ...           see the Qt documentation for more options
```


Option `-nographics` initializes the Qt library for a text-only
application. This can be handy when executing scripts because it
avoids unnecessary overhead.

Option `-onethread` runs the Lua interpreter in the main thread
instead of starting a dedicated thread.  As a consequence, while Lua
is running, graphical operations are not handled, windows are not
painted, timers are not honored, etc., unless the Lua program calls
[qt.doevents()](#qt.doevents) frequently enough.  This is not the
normal mode of operation of `qlua` but this can be useful for
debugging purposes.

<a name="qlua.components"/>
## Components ##

Program `qlua` is a Qt program composed of three objects working
independently in separate threads.

  * The [console manager](#qt.qconsole) manages the interactive session using input and output on a terminal or inside a shell window. It can capture all output to the `stdout` file descriptor and publish it as Qt signal for use in graphical interfaces.  It also manages the user input with Lua code completion.

  * The [QtLua engine](#qt.qengine) runs the Lua code and provides the integration with the Qt environment.

  * The [application manager](#qt.qapp) runs in the main thread. It handles the graphical user interface events and all the notifications emitted by the console manager and the lua engine.

Running the Lua interpreter in a separate thread lets the main thread
handle user interaction at all times. Windows are refreshed and user
interfaces remain active while a Lua program is running.

At times the Lua code needs to act on objects living in another
thread.  Potential synchronization problems are avoided using the
["thread hopping"](#qt.qcall) technique which allows the execution of
a Lua function in the target thread.  When the function returns, the
Lua thread resumes the normal execution of the Lua code.  Thread
hopping happens automatically when invoking a method defined by a Qt
object. Thread hopping can also be performed manually using functions
[qt.qcall](#qt.qcall) or [qt.xqcall](#qtxqcall).


<a name="qt"/>
# Package qt #

When you run program `qlua`, the package `qt` is readily
accessible using `require('qt')` because the `libqtlua` library
installs a suitable loading function in the array
`package.preload['qt']`.

Package `qt` essentially contains the metatables associated with Lua
userdata values and a few useful functions.  The [QtLua API](#QtLua)
also provides a means to name an arbitrary Qt object and publish it
inside the package `qt`.

<a name="qt.QVariants"/>
## Qt Variants ##

Values represented by the Qt class
[QVariant](http://doc.trolltech.com/4.4/qvariant.html) can be exposed
to the Lua interpreter as userdata.  Class `QVariant` defines in
fact a container for a large number of standard Qt types with value
semantics.  In fact any type registered with the Qt macro
[Q_DECLARE_METATYPE](http://doc.trolltech.com/4.4/qmetatype.html) can
be represented by a Qt variant value.

Whenever a `QVariant` value is made accessible to the Lua
interpreter (see [luaQ_pushqt](#luaqpushqt)), the QtLua library
ensures that there is a metatable associated with all objects of that
type.  Methods for each type can then be added to the class table
`qt.TypeName` where `TypeName` represent the type name.  If the
class table contains a method named `new`, this method is used to
construct new values of that type using the syntax
`qt.Typename(...)`.

The metatable defines the standard Lua hooks `__eq` to perform an
equality test and `__tostring` to return a string representing the
data.  In addition, the following methods are predefined in all class
tables:

<a name="qt.tostring"/>
### qtdata:tostring() ###

Function `qtdata:tostring()` returns a Lua string
representing the value contained in the Qt variant.

  * Qt variants of type `QString` are converted to Lua strings using the local multibyte encoding.
  * Qt variants that can be converted to type `QByteArray`are also transformed into Lua strings.
  * Other types are represented using the type name followed by the hexadecimal representation of a pointer to the data.

<a name="qt.tonumber"/>
### qtdata:tonumber() ###

Function `qtdata:tonumber()` returns a Lua number representing the
value contained in the Qt variant.  The value `nil` is returned when
the data type cannot be meaningfully converted to a numeric value.

<a name="qt.tobool"/>
### qtdata:tobool() ###

Function `qtdata:tobool()` returns a boolean value associated with
the value contained in the Qt variant.  This is achieved by calling
the Qt function `QVariant::isNull`.  In the case of object pointers,
this function can be used to determine if the object has been deleted
or is currently allocated.


<a name="qt.type"/>
### qtdata:type() ###

Function `qtdata:type()` returns a string naming the type of the
value contained in the Qt variant.  In the case of object pointers,
this function returns a string of the form `"ClassName*"` where
`ClassName` represents the object class.  If the object has been
deleted, this function returns `nil`.


### qtdata:isa(typename) ###
![](anchor:qt.isa)

Function `qtdata:isa(typename)` returns a boolean value indicating
whether the Qt variant contains an object of type `typename`.  In
the case of object pointers, this function returns true if the object
inherits the qt object class named `typename`.


<a name="qt.QObjects"/>
## Qt Objects ##

Critical Qt classes are subclasses of class
[QObject](http://doc.trolltech.com/4.4/qobject.html).  Qt distinguishes
some member functions of these classes, namely slots, signals and
properties. Qt provides means to enumerate these slots, signals and
properties at run time.  Please refer to the Qt documentation on the
[Qt object model](http://doc.trolltech.com/4.4/object.html) for more
details.

Instances of these classes are known as Qt objects.  Pointers to Qt
objects can be exposed to the Lua interpeter because Qt variants can
contain values of type `QObjectPointer` which is a guarded pointer
to a Qt object.

Such values have a special treatment:

  * A distinct metatable is allocated for each Qt object class. These metatables are available in the `qt` package and follow the same inheritance pattern as the Qt object class hierarchy. Class [qt.QObject](..:qtcore:index#qobject) sits at the top of this hierarchy.

  * Slots, signals and properties are automatically exposed to the Lua interpreter. This is achieved using the information collected by the Qt precompiler `moc`.

<a name="qt.memorymanagement.dok"/>
### Memory management ###

Qt organizes the Qt objects hierarchically.  An object can be an
orphan or can be the child of another object.  When a Qt object is
deleted, all its descendants are deleted as well.  When programming in
C++, orphan objects must be deleted manually.

Two situations arise when a Qt object is exposed the Lua interpreter.

  * An exposed Qt object may be deleted by C++ code, either directly, or indirectly because one of its ascendant has been deleted. The Lua userdata representing the object becomes a "zombie". Zombies can be detected using method [tobool()](#qt.tobool).

  * An exposed Qt object may be deleted because the Lua garbage collector determines that the Lua program no longer needs this object.  Whether this happens is determined when the object is exposed to the Lua interpreter. Such objects are said to be owned by Lua.

In general, orphan objects created from a Lua program are owned by
Lua.  Objects with a parent and orphan objects created from C++ code
are not Lua owned and are not affected by the decisions of the Lua
garbage collector.


<a name="qt.properties.dok"/>
### Properties ###

Qt object properties are automatically available using the Lua
indexation syntax

Examples:
```lua
    widget.windowTitle="my window title"
    return widget.windowTitle
```

Properties are always manipulated from the thread owning the Qt
object.  If the Lua interpreter runs in a different thread, a
[thread hopping](#qt.qcall) operation is performed for the duration
of the operation.

Property values are always represented as Qt variant objects.  
`QByteArray` variants and numerical variants are automatically 
converted into Lua strings or Lua numbers when they are read. 
However `QString` variants are left unchanges since such strings
can contain non ASCII characters. Conversely, Lua strings and 
Lua numbers are automatically converted into the desired type 
when a property is set.

Property values with enumerated types registered using the Qt macro
[Q_ENUMS](http://doc.trolltech.com/4.4/qobject.html) are
automatically converted into strings representing the enumerated names
and conversely.  Property values with flag types declared using the Qt
macro [Q_DECLARE_FLAGS](http://doc.trolltech.com/4.4/qflags.htm) and
registered using `Q_ENUMS` are automatically converted into Lua
string containing the relevant flag names separated by vertical bars.



<a name="qt.slots.dok"/>
### Slots and Invokable Methods ###

Public and protected Qt slots can be called using the Lua method
invokation syntax.

Examples:
```lua
       widget:setEnabled(true)
       widget:show()
```

Public and protected member functions whose c++ declaration is marked
with the Qt macro `Q_INVOKABLE` can also be invoked using the
standard Lua syntax.

When a member function has the same name as a property, the property
takes precedence.  When overloaded member functions are exposed in
this fashion, the QtLua library makes a best effort to determine which
function should be called. It first narrows the choices using the
number of parameters. When several options remain, it chooses one on
the basis of the types of the parameters.  Member function arguments
with default values are handled by considering implicit overloaded
functions.

It is always possible to work around these rules by selecting a
particular member function using its signature.

```lua
       widget['setEnabled(bool)'](widget, true)
```

Member functions exposes in this way are always invoked from the
thread owning the Qt object. If the Lua interpreter runs in a
different thread, a [thread hopping](#qt.qcall) operation is
performed for the duration of the operation.

A Lua string is automatically converted when a `QByteArray` or a `QString`
argument is expected. A Lua number is automatically converted whenever a
numerical type is expected. Return values of type ''QByteArray'' are 
automatically converted into Lua strings. Return values of numerical
types are converted into Lua numbers.

Inheritance ensures that all methods exposed by a class are also
exposed by its subclasses.  Methods exposed by the `QObject` class,
such as [deleteLater()](..:qtcore:index#qobjectdeletelater), are
inherited by all Qt object classes.

<a name="qt.namedchildren.dok"/>
### Named Children ###

All Qt objects can be given a name represented by property
[objectName](..:qtcore:index#qobjectobjectname).  The named children
of a Qt object can be accessed by name using the indexation syntax.
However, when the name of a child conflicts with a property or an
exposed method, the property or the exposed method has
precedence. Named children are searched in the whole descendency tree
using the Qt function `qFindChild`.


<a name="qt.signals.dok"/>
### Signals ###

Lua programs can emit and receive Qt signals.

<a name="qt.emttingsignals.dok"/>
#### Emitting Signals ####

Lua programs can emit signals using the function call syntax.
Consider for example the following c++ class:
```c++
class MyQObject : public QObject {
  Q_OBJECT
...
signals:
  void mysignal(QByteArray msg, int x=0);
...
```

Given an instance `myqobject` of this class, 
a Lua program can emit signal `mysignal` 
as follows 

```lua
  myqobject:mysignal("mymessage") -- or
  myqobject:mysignal("mymessage", 4)
```

Overloaded signals are resolved like method invokation.  In
particular, arguments with default values are handled by considering
implicit overloaded functions.  As for functions, it is always
possible to work around these rules by selecting a particular signal
using its signature.

```lua
myqobject['mysignal(QByteArray,int)'](self,"message",3)
```

<a name="qt.receivingsignals.dok"/>
#### Receiving Signals ####

Qt signals can be connected to Lua functions 
using [qt.connect(...)](#qt.connect).

For instance, the clause

```lua
  qt.connect(qobject,'mysignal(QByteArray)',
             function(s) print("mysignal("..s..")") end)
```

ensures that the specified lua function is called whenever the signal
with signature `mysignal(QByteArray)` is emitted.  Arguments with
string types are converted to Lua strings, and arguments numerical
types are converted to Lua numbers.

Functions passed to `qt.connect` can of course be closures:

```lua
  local table = some_lua_object()
  qt.connect(qobject,'mysignal(QByteArray)',
             function(s) table:mysignalemitted(s) end)
```

When the signal occurs while the Lua interpreter is busy, the
execution of the Lua function connected to the signal is delayed until
the Lua interpreter becomes available, either because the execution of
the current Lua code terminates or because function
[qt.doevents()](#qt.doevents) is called.


<a name="qt.qobjects.dok"/>
## Objects ##

<a name="qt.qapp"/>
### qt.qApp ###

Expression `qt.qApp` refers to the application object
which orchestrates the operation of the [qlua](#qlua) program.
This object is the sole instance of class `QLuaApplication`.
A few exposed slots and properties are relevant for Lua programs.

<a name="qt.qapp.quit"/>
#### qt.qApp:quit() ####

Function `qt.qApp:quit()` quits the `qlua` application.

<a name="qt.qapp.restart"/>
#### qt.qApp:restart(bool) ####

Function `qt.qApp:restart(bool)` reinitializes
and restarts the Lua interpreter. 
When the optional flag `bool` is true,
the command line arguments passed to the program
[qlua](#qlua) are re-executed.

<a name="qt.qapp.about"/>
#### qt.qApp:about() ####

Function `qt.qApp:about()` displays an about box dialog displaying
the contents of the property `qt.qApp.aboutMessage`.

<a name="qt.qapp.aboutmessage"/>
#### qt.qApp.aboutMessage ####

Property `qt.qApp.aboutMessage` contains an HTML string that is
displayed in the about box when function `qt.qApp:about()` is
invoked.

<a name="qt.qapp.readsettings"/>
#### qt.qApp:readSettings(key) ####

Reads the preference settings associated with string `key`.

<a name="qt.qapp.writesettings"/>
#### qt.qApp:writeSettings(key,value) ####

Sets the preference settings associated with string `key`
to `value`.  Argument `value` should be a Lua value
that can be converted into a `QVariant`.

<a name="qt.qapp.runswithoutgraphics"/>
#### qt.qApp:runsWithoutGraphics() ####

This function returns `true` if the `qlua` interpreter
was launched with the option `-nographics`.


<a name="qt.qconsole"/>
### qt.qConsole ###

Expression `qt.qConsole` refers to the console object
which handles console input and output for the [qlua](#qlua) program.
This object is the sole instance of class `QLuaConsole`.
A few exposed slots, signals and properties are relevant for Lua programs.

<a name="qt.qconsole.captureoutput"/>
#### qt.qConsole.captureOutput ####

When the boolean property `qt.qConsole.captureOutput` is set,
all output to the `stdout` file descriptor is
captured and reexpressed by emitting signal
`consoleOutput(QByteArray)`.

<a name="qt.qconsole.printCapturedOutput"/>
#### qt.qConsole.printCapturedOutput ####

When `qt.qConsole.captureOutput` is true,
the boolean property `qt.qConsole.printCapturedOutput` determines
whether the captured text is display on the console or not.

<a name="qt.qconsole.addToHistory"/>
#### qt.qConsole:addToHistory(string) ####

Function call `qt.qConsole:addToHistory(string)` adds 
`string` to the history of recent console input.
This works only when [qlua](#qlua) is compiled
with command line history support.

<a name="qt.consoleOutput"/>
#### [QLuaConsole signal] consoleOutput(QByteArray) ####

When `qt.qConsole.captureOutput` is true, 
signal `consoleOutput(QByteArray)` is emitted 
whenever a string is output to the file descriptor `stdout`.
Capturing this signal can be useful to program a graphical 
replacement for the console.

<a name="qt.ttyInput"/>
#### [QLuaConsole signal] ttyInput(QByteArray) ####

Signal `ttyInput(QByteArray)` is emitted whenever a new 
input line has been validated by the user on the console.

<a name="qt.ttyEndOfFile"/>
#### [QLuaConsole signal] ttyEndOfFile() ####

Signal `ttyEndOfFile()` is emitted whenever an end-of-file
condition is detected on the console input. When this happens, 
`qlua` asks the user if he really wants to quit the application.

<a name="qt.ttyBreak"/>
#### [QLuaConsole signal] ttyBreak() ####

Signal `ttyBreak()` is emitted whenever the interrupt key sequence,
typically `Ctrl+C` or `Ctrl+Break` is typed on the input console.
When this happens, `qlua` stops the execution of Lua programs
and waits for new commands on the console.


<a name="qt.qengine"/>
### qt.qEngine ###

Expression `qt.qEngine` refers to the Lua execution engine.
This object is an instance of class [QtLuaEngine](#qt.QtLuaEngine).
A few exposed slots, signals and properties are relevant for Lua programs.
See the documentation of class [QtLuaEngine](#qt.QtLuaEngine)
in the [QtLua](#QtLua) API section for more information.

<a name="qt.qengine.lastErrorMessage"/>
#### qt.qEngine.lastErrorMessage ####

Read-only property `qt.qEngine.lastErrorMessage` contains the
last error message generated by the Lua interpreter.

<a name="qt.qengine.lastErrorLocation"/>
#### qt.qEngine.lastErrorLocation ####

Read-only property `qt.qEngine.lastErrorLocation` contains a
`QStringList` whose elements describe the program locations
extracted from the stack when the last error message was recorded.
When a location corresponds to a line in a file, 
the string has the format `"@filename:linenumber"`.

<a name="qt.qengine.printResults"/>
#### qt.qEngine.printResults ####

Boolean property `qt.qEngine.printResults` indicates whether
the Lua interpreter should print the results returned by
the evaluation of interactive Lua expressions.

<a name="qt.qengine.printErrors"/>
#### qt.qEngine.printErrors ####

Boolean property `qt.qEngine.printErrors` indicates whether
the Lua interpreter should print the error messages
caused by the evaluation of interactive Lua expressions.


<a name="qt.functions.dok"/>
## Functions ##

<a name="qt.connect"/>
### qt.connect(qobject, signature, function, [direct] ) ###

Expression `qt.connect(qobject, signature, func)` connects the
signal `signature` from Qt object `qobject` to the Lua function
`func`.  Argument `signature` is a string containing the full
signature (c++ prototype) of the signal. The function is invoked
whenever the signal is emitted.

Example:
```lua
require 'qt'
timer = qt.QTimer()
timer.singleShot = true
qt.connect(timer,'timeout()', function() print("timeout") end)
timer:start(2000) -- wait for 2 seconds...
```

See section [Receiving Signals](#qt.recevingsignals.dok) for more details.

When the signal occurs while the Lua interpreter is busy,
the execution of the Lua function connected to the signal
is delayed until the Lua interpreter becomes available,
either because the execution of the current Lua code terminates
or because function [qt.doevents()](#qt.doevents) is called.

Setting the optional flag `direct` to `true` changes that when
the Qt signal is emitted from the thread currently running a Lua program.
It is then assumed that the signal is a consequence of the Lua 
program execution and the function is called synchronously.

___Warning___ : Function `func` remains allocated as long as the
connection exists.  Memory can be leaked if function `func` is a
closure that points to the signaling object because the garbage
collector will find that the object is in use.  Future versions of
QtLua may address this weakness of the interface between the Lua
garbage collector and the Qt static memory allocation.  In the mean
time, be careful to
[delete](..:qtcore:index#qobjectdeletelater) or
[disconnect](#qt.disconnect) connected objects as soon as you do not
need them anymore.


<a name="qt.connect"/>
### qt.connect(qobject1, signature1, qobject2, signature2) ###

Expression `qt.connect(qobject1, signature1, qobject2, signature2)` 
connects signal `signature1` from Qt object `qobject1` to slot 
or signal `signature2` of Qt object `qobject2`.
This is similar to the Qt function 
[QObject::connect(...)](http://doc.trolltech.com/4.4/qobject.html#connect).

<a name="qt.disconnect"/>
### qt.disconnect(qobject, signature, function) ###

Expression `qt.disconnect(qobject, signature, func)`
disconnects the signal `signature` from Qt object `qobject`
from the Lua function `func`. 
Arguments `signature` or `function` may be `nil`.
All connections matching the non-nil arguments
will then be disconnected.


<a name="qt.disconnect"/>
### qt.disconnect(qobject1, signature1, qobject2, signature2) ###

Expression `qt.disconnect(qobject1, signature1, qobject2, signature2)` 
connects signal `signature1` from Qt object `qobject1` from 
slot or signal `signature2` of Qt object `qobject2`.
Arguments `signature1`, `qobject2`, or `signature2` may be `nil`.
All connections matching the non-nil arguments will 
then be disconnected. 
This is similar to the Qt function 
[QObject::disconnect(...)](http://doc.trolltech.com/4.4/qobject.html#disconnect).


<a name="qt.doevents"/>
### qt.doevents([waitflag]) ###

Function `qt.doevents()` processes all pending events 
and executes the functions associated with all queued signals.
Calling this function periodically can be useful to
ensure that queued signals are processed timely.

Calling this function is very important when you run `qlua` in
single-threaded mode using option `-onethread`, as graphic output
will not be displayed properly until `qt.doevents()` is called.

Setting flags `waitflag` to `true` causes the function to wait
until at least one event is processed by the current thread.  This can
of course cause an infinite wait when there are no event source for
the current thread.


<a name="qt.isa"/>
### qt.isa(arg, typename) ###

If the Lua value `arg` represents a Qt object or variant,
expression `qt.isa(arg,typename)` returns a boolean indicating
whether the Qt data is a subclass of type `typename`,
as if [arg:isa(typename)](#qt.isa) had been called.
Otherwise it simply returns `nil`.


<a name="qt.pause"/>
### qt.pause() ###

Expression `qt.pause()` causes the interpreter to enter the mode
`Paused`.  When the interpreter pauses with `qt.pause`, all events
are processed and all Lua functions associated with signals are
executed whenever the signal occurs.

Note that calling `qt.qEngine:stop(false)` also causes the
interpreter to enter the mode `Paused`.  However, in that case, the
Lua functions associated with signals are not executed.


<a name="qt.qcall"/>
### qt.qcall(qobject, function, ... ) ###

Expression `qt.qcall(qobject, f, ...)` executes 
function `f` from the thread owning the Qt object `qobject`.
To achieve this, the current thread performs a rendez-vous
with the object thread. The object thread takes ownership
of the Lua data structures and invokes function `f` with
the specified arguments. Meanwhile the original thread
processes events and waits. When the function `f` returns,
the original thread resumes the execution.

Like the standard Lua function [pcall](..:LuaManual#pcallfarg), this
function returns multiple results. If an error occurs during the
execution, this function returns `false` plus the error message.
Otherwise it returns `true` plus the results returned by function
`f`.

This function is the basis for the thread hopping mechanism that
underlies the multi-thread operations in QtLua.  Thread hopping
happens automatically whenever one manipulates a slot or a property of
a Qt object.  It may occasionnally be useful to manually hop into an
object thread in order to avoid the overhead of subsequent thread
hopping operations: if the Lua interpreter is already running in the
desired thread, the function `f` can be invoked directly without
expensive synchronization.

___Important___ : Thread hopping only works if the 
target thread runs an event loop, 
using `QThread::exec()` or `QEventLoop::exec()`.
Execution of Lua program will stop if this is not the case.

Within program [qlua](#qlua), object [qt.qApp](#qt.qapp) is always
owned by the main thread in charge of the user interaction, and object
[qt.qEngine](#qt.qengine) is always owned by the auxilliary thread in
charge of the evaluation of Lua expressions.


<a name="qt.require"/>
### qt.require(modulename) ###

This function searches and loads the shared library `modulename`
almost like the standard lua function `require`.  However this
function ensures that the shared library is loaded in a way maximally
compatible with C++ code.  Symbols defined by the shared library are
defined in the global name space, that is, are available to resolve
undefined symbols in other libraries.  This is necessary for
supporting the C++ RTTI features.  This function also ensures that
remains loaded when the Lua state is closed. This is necessary when
the shared library defines classes whose instances outlive the Lua
interpreter.

This function supports none of the advanced search methods implemented
by the usual function `require`, other than searching a shared
library along `package.cpath`.


<a name="qt.type"/>
### qt.type(arg) ###

If the Lua value `arg` represents a Qt object or variant, expression
`qt.type(arg)` returns a string containing the type of the Qt data
as if [arg:type()](#qt.type) had been called.  Otherwise it simply
returns `nil`.


<a name="qt.xqcall"/>
### qt.xqcall(qobject, function, ..., errorhandler) ###

Expression `qt.xqcall(qobject, f, ..., handler)` 
is a thread hopping analogous to the standard \
Lua function [xpcall](..:LuaManual#xpcallferr).

Function `f` is invoked with the specified arguments
by the thread owning the Qt object `qobject`.
If the execution completes without error, 
`xqcall` returns `true` plus the results returned by `f`.
If an error occurs during the execution,
the error handling function `handler` is invoked
on the error message in the Qt object thread.
The `xqcall` function then returns `false`
followed by the values returned by the error handler.


<a name="QtLua"/>
# QtLua API #

This section describes the QtLua API.  Class
[QtLuaEngine](#qt.QtLuaEngine) and [QtLuaLocker](#qt.QtLuaLocker) provides
a Lua interpreter for use in any Qt application with capabilities
comparable to those of the
[QtScript](http://doc.trolltech.com/4.4/qtscript.html) and additional
support for multi-threaded execution.  Additional functions with
prefix [luaQ](#luaQ) provide the essential tools for manipulating Qt
data in Lua functions written in C or C++.


<a name="qt.QtLuaEngine"/>
## Class QtLuaEngine ##

Class `QtLuaEngine` is a `QObject` subclass representing a Lua interpreter.
This object can be used to add a Lua interpreter to any Qt application
with capabilities comparable to those of the 
[QtScript](http://doc.trolltech.com/4.4/qtscript.html) language
and additional support for multi-threaded execution.

Instances of this class can be in one of three state:
  * State `QtLuaEngine::Ready` indicates that the interpreter is ready to accept new Lua commands.
  * State `QtLuaEngine::Running` indicates that the interpreter is currently executing a Lua program.
  * State `QtLuaEngine::Paused` indicates that the interpreter was suspended while executing a Lua program. One can then use the Lua debug library to investigage the Lua state.

Class `QtLuaEngine` provides member functions to 
submit Lua strings to the interpreter and to collect 
the evaluation results.  If these functions are invoked
from the thread owning the Lua engine object,
the Lua code is executed right away. 
Otherwise a [thread hopping](#qt.qcall) operation
ensures that the execution of a Lua program happens 
in the thread owning the Lua engine object. 

Class [QtLuaLocker](#qt.QtLuaLocker) provides means
to directly access the Lua state 
[`lua_State*`](..:LuaManual#luaState) using 
the Lua API.


<a name="qt.qtluaengine.constructor"/>
### Constructor ###

`QtLuaEngine::QtLuaEngine(QObject *parent = 0)`

This is the constructor for class `QtLuaEngine`.  Argument
`parent` is the optional parent object.

<a name="qt.qtluaengine.properties"/>
### Properties ###

`[QtLuaEngine property, readonly] QtLuaEngine::State state`

The read-only property `state` contains the state of the Lua engine.

State `QtLuaEngine::Ready` indicates that the interpreter
is ready to accept new Lua commands.  

State `QtLuaEngine::Running` indicates that the interpreter
is currently executing a Lua program.

State `QtLuaEngine::Paused` indicates that the interpreter
was suspended while executing a Lua program.

`[QtLuaEngine property, readonly] bool ready`

`[QtLuaEngine property, readonly] bool running`

`[QtLuaEngine property, readonly] bool paused`

The read-only properties `ready`, `running`, and `paused`
are `true` if the Lua engine is in the corresponding state.

`[QtLuaEngine property, readonly] QByteArray lastErrorMessage`

The read-only property `lastErrorMessage` contains the text
of the last error message produced by the Lua interpreter.

`[QtLuaEngine property, readonly] QStringList lastErrorLocation`

The read-only property `lastErrorLocation` contains a list of locations 
extracted from the stack when the last error message was recorded.
When a location corresponds to a line in a file, 
this string has the format `"@filename:linenumber"`.

`[QtLuaEngine property] bool printResults`

Boolean property `printResults` indicates 
whether the Lua interpreter should
print the results returned by the evaluation of the strings
passed to functions `eval` or `evaluate`. 
The default value is `false`.

`[QtLuaEngine property] bool printErrors`

Boolean property `printErrors` indicates 
whether the Lua interpreter should
print error message caused by the evaluation of the strings
passed to functions `eval` or `evaluate`
and by signal function execution.
The default value is `true`.


`[QtLuaEngine property] bool pauseOnError`

Boolean property `pauseOnError` indicates
whether the interpreter should enter state `Paused`
when an error occurs. Setting this to `true`
implies that some code will call function `resume`
otherwise the interpreter will remain in state `Paused` forever.
The default is `false`.


`[QtLuaEngine read-only property] bool runSignalHandlers`

This property is true when the Lua interpreter
honors the signal handler invokations immediately 
instead of queuing them for further processing.


<a name="qt.qtluaengine.signals"/>
### Signals ###

`[signal] void QtLuaEngine::stateChanged(int state)`

Signal `stateChanged` is emitted whenever 
the interpreter state is changed.

`[signal] void QtLuaEngine::errorMessage(QByteArray)`

Signal `errorMessage` is emitted whenever an error
message is captured by the error handler set
by functions `eval` or `evaluate`.


<a name="qt.qtluaengine.execution"/>
### Execution ###

`[slot] bool QtLuaEngine::eval(QByteArray s, bool async = false)`

`[slot] bool QtLuaEngine::eval(QString s, bool async = false)`

Function `eval` runs the Lua code contained in string `s`.

When function `eval` is invoked from the thread owning 
the Lua engine object, the Lua code is executed right away. 
Otherwise the execution of the Lua code is started
in the thread owning the Lua engine object using
[thread hopping](#qt.qcall).  Function `eval`
returns immediately if flag `async` is `true`.
Otherwise it processes Qt events and waits
until the termination of the Lua code.

This function returns `true` if the evaluation
was performed without error. It immediately
returns `false` if the Lua engine was not
in state `ready` or if flag `async` was true
when calling the function from the engine thread.

`[slot] QVariantList QtLuaEngine::evaluate(QByteArray s)`

`[slot] QVariantList QtLuaEngine::evaluate(QString s)`

Function `evaluate` returns an empty `QVariantList` if
called when the engine is not in ready state.
Otherwise if evaluates Lua code `s` like function `eval`.
If an error occurs during evaluation, 
function `evaluate` returns a list
whose first element is `QVariant(false)` and whose
second element is the error message. 
Otherwise is returns a list whose first element is
`QVariant(true)` and whose remaining elements
are the evaluation results.


`[slot] bool QtLuaEngine::stop(bool nopause = false)`

When the Lua engine is in state `Running`, 
function `stop` interrupts the execution 
of Lua code and returns `true`.
If flag `nopause` is false,
the Lua engine enters the state `Paused` when it stops.
Otherwise it unwinds the stack as if an error 
had occurred and returns to state `Ready`.
If the Lua engine is not in state `Running`
or already in state `Paused`, this function does 
nothing and simply returns `false`.


`[slot] bool QtLuaEngine::resume(bool nocontinue=false)`

If argument `nocontinue` is `false` and
the interpreter is in state `Paused` because
function `stop` was called, the interpreter returns to 
state `Running` and resumes the execution of the Lua 
code interrupted by function `stop`.
If the interpreter is in state `Paused` for 
another reason, it unwinds the stack, and 
returns to state `Ready` as if an error had occurred.
Otherwise the function does nothing and returns false.

If argument `nocontinue` is `true` and the 
interpreter is in state `Paused` or `Running`, 
it unwinds the stack, and returns to state `Ready` 
if an error had occurred. Otherwise the function 
does nothing and returns false.


### Miscellaneous ###

`void QtLuaEngine::nameObject(QObject *obj, QString name = QString())`

`QObject* QtLuaEngine::namedObject(QString name)`

`QList<QObjectPointer> QtLuaEngine::allNamedObjects()`

These functions can be used to make a Qt object
accessible by name in package `qt` by Lua programs.
Function `nameObject` sets the object's name 
and declare the object accessible in package `qt`.
Function `namedObject` retrieved an accessible object
by its name. Function `allNamedObjects` return
all accessible objects.  An accessible object
is always accessed using its object name,
even if one uses `QObject::setObjectName` to changed
it after calling `nameObject`.



`void QtLuaEngine::registerMetaObject(const QMetaObject *mo)`

This function declare the Qt class identified by the metaclass `mo`
to the QtLua system.  This usually happens automatically when
an object of that class is passed to QtLua for the first time.
In rare case, it may be necessary to call this function
to ensure that the types of slot and signal arguments are 
properly recognized as an object class.



<a name="qt.QtLuaLocker"/>
## Class QtLuaLocker ##

Class [QtLuaLocker](#qt.QtLuaLocker) provides means
to directly access the Lua state `lua_State*` using the Lua API.
This class ensures that the current thread has exclusive
access to the Lua state.

`QtLuaLocker::QtLuaLocker(QtLuaEngine *engine)`

Create a `QtLuaLocker` object and ensures that the 
current thread has exclusive access to the Lua state
for the Lua interpreter `engine`.
This constructor hangs until obtaining the lock

`QtLuaLocker::QtLuaLocker(QtLuaEngine *engine, int timeOut)`

Create a `QtLuaLocker` object and ensures that the 
current thread has exclusive access to the Lua state
for the Lua interpreter `engine`.
This constructor hangs at most for `timeOut` milliseconds.
To know whether the lock was acquired, 
cast the `QtLuaLocker` as a `lua_State*` pointer.


`lua_State* QtLuaLocker::operator lua_State*()`

Returns a [`lua_State*`](..:LuaManual#luaState) 
pointer to access the state
of the Lua interpreter using the Lua API.
Since this is a cast operator, one can simply pass 
the `QtLuaLocker` object whenever a `lua_State*` is expected.
This cast returns `0` when the constructor was unable
to acquire the exclusive lock during the specified timeout.


`bool QtLuaLocker::isReady()`

Returns `true` if the locking operation was 
successful and the interpreter is in ready state. 
Note that locking and state are distinct concepts.
It is possible to lock a running interpreter
while it is waiting for other events.
The `eval` and `evaluate` functions use this test
to decide whether to run a command.


`void QtLuaLocker::setRunning()`

Sets the interpreter to state `Running`.
The engine will return to state `Ready` after
the destruction of the last `QtLuaLocker` object 
and the execution of the queue.
Temporary releasing the lock with `unlock()`
keeps the engine in state `Running`. */

This can be useful if you plan to call the Lua API 
functions [lua_call](..:LuaManual#luacall) 
or [lua_pcall](..:LuaManual#luapcall) and expect
the code to run for an extended period of time.


<a name="luaQ"/>
## The luaQ functions ##

The `luaQ` functions complete the Lua API with
facilities related to the QtLua interface.
Some `luaQ` functions are only meaningful for C++ program.
The following snipped ensure that all available
functions are accessible in both C and C++.

```c++
 #include "qtluautils.h"
 #ifdef __cplusplus
 # include "qtluaengine.h"
 #endif
```

<a name="luaqluaopen_qt"/>
### luaopen_qt ###

`int luaopen_qt(lua_State *L)`

Load the `qt` package into the interpreter.
This is the function preloaded into `package.preload['qt']`.


<a name="luaqcall"/>
### luaQ_call ###

`void luaQ_call(lua_State *L, int na, int nr, QObject *obj = 0)`

Perform a Lua function call like [lua_call](..:LuaManual#luacall) 
but ensure that the function is executed in the thread owning object `obj`
using [thread hopping](#qt.qcall).
Unlike function [luaQ_pcall](#luaqpcall),
this function relays Lua errors into the current thread.
This is similar to calling
```
  if (luaQ_pcall(L,na,nr,0,obj)) 
    lua_error(L);
```

<a name="luaqcheckqobject"/>
### luaQ_checkqobject&lt;TYPE&gt; ###

`TYPE* luaQ_checkqobject&lt;TYPE&gt;(lua_State *L, int index)`

This function causes an error if the Lua value at position `index`
in the stack does not represent a [Qt object](#QObjects) of class `TYPE`.
Otherwise it returns a pointer to the object.

<a name="luaqcheckqvariant"/>
### luaQ_checkqvariant&lt;TYPE&gt; ###

`TYPE luaQ_checkqvariant&lt;TYPE&gt;(lua_State *L, int index)`

This function causes an error if the Lua value at position `index`
in the stack cannot be converted to 
[Qt variant](#Qt.Qvariants) of actual type `TYPE`.
Otherwise it returns a value of the requested type.

The C++ type `TYPE` must be known to the Qt meta type system.
This can always be achieved using the macro 
[Q_DECLARE_METATYPE](http://doc.trolltech.com/4.4/qmetatype.html).

<a name="luaqcomplete"/>
### luaQ_complete ###

`int luaQ_complete(lua_State *L)`

This function first pops a string from the top of the stack.
The string could contain a symbol or several symbols separated with dots of periods.
Function `luaQ_complete` then pushes a table containing potential 
completions for the last symbol in the string.

<a name="luaqconnnect"/>
### luaQ_connect ###
`bool luaQ_connect(lua_State*L, QObject*o, const char *s, int fi)`

Connects the signal with signature `s` from the Qt object `o`
to the function at position `fi` in the stack.
Returns `true` in case of success. 
Returns `false` if the specified signal was not found.

See also [qt.connect(...)](#qt.connect)


<a name="luaqdisconnect"/>
### luaQ_disconnect ###

`bool luaQ_disconnect(lua_State*L, QObject*o, const char *s, int fi)`

Disconnects the signal with signature `s` of Qt object `o`
from the function located at index `fi` in the stack.
When argument `fi` is zero, this function removes
all connections from the specified signal to a Lua function.
When argument `s` is null, this function removes
all connections from the specified object to the specified function.

See also [qt.disconnect(...)](#qt.disconnect).

<a name="luaqdoevents"/>
### luaQ_doevents ###

`void luaQ_doevents(lua_State *L, bool wait = false)`

Processe all pending events 
and execute the functions associated with all queued signals.
Calling this function periodically can be useful to
ensure that queued signals are processed timely.

See also [qt.doevents()](#qt.doevents).

<a name="luaqengine"/>
### luaQ_engine ###

`QtLuaEngine *luaQ_engine(lua_State *L)`

Returns a pointer to the current [Lua engine](#qt.QtLuaEngine).

<a name="luaqgetfield"/>
### luaQ_getfield ###

`void luaQ_getfield(lua_State *L, int index, const char *name)`

This function is similar to [lua_getfield](..:LuaManual#luagetfield)
but never propagates errors causes by executing the metatable
`__index` functions.  When such an error occurs, this function
simply returns `nil`.


<a name="luaqoptqobject"/>
### luaQ_optqobject&lt;TYPE&gt; ###

`TYPE* luaQ_optqobject&lt;TYPE&gt;(lua_State *L, int index, TYPE *d)`

Returns the optional [Qt object](#QObjects) of type `TYPE`
found at position `index` in the stack.  If the position
corresponds to an unspecified function argument, 
this function returns `d`.  If the Lua value at this position
is not a Qt object of the requested type, this function
causes an error.

See also [luaQ_checkqobject](#luaqcheckqobject)

<a name="luaqoptqvariant"/>
### luaQ_optqvariant&lt;TYPE&gt; ###

`TYPE luaQ_optqvariant&lt;TYPE&gt;(lua_State *L, int index, TYPE d = TYPE())`

Returns the optional [Qt variant](#Qt.Qvariants) of actual type `TYPE`
found at position `index` in the stack.  If the position
corresponds to an unspecified function argument, 
this function returns `d`.  If the Lua value at this position
cannot be converted to the requested type, this function
causes an error.

See also [luaQ_checkqvariant](#luaqcheckqvariant)

<a name="luaqpause"/>
### luaQ_pause ###

`void luaQ_pause(lua_State *L)`

Causes the interpreter to enter the mode `Paused`. 
When the interpreter pauses with this function, 
all events are processed and all Lua functions 
associated with signals are executed 
whenever the signal occurs.  


<a name="luaqpcall"/>
### luaQ_pcall ###

`int luaQ_pcall(lua_State *L, int na, int nr, int eh, QObject *obj = 0)`

`int luaQ_pcall(lua_State *L, int na, int nr, int eh, int oh)`

Performs a Lua function call like 
[lua_pcall](..:LuaManual#luapcall) but ensures that
the function is executed in the thread owning the Qt object `obj`
or the Qt object at position `oh` on the stack.
Argument `eh` is an event handler that will be executed
in the target thread if an error occurs during the
execution of the function.

This function is the basis for all [thread hopping](#qt.qcall)
operations in the QtLua interface.


<a name="luaqprint"/>
### luaQ_print ###

`int luaQ_print(lua_State *L, int nr)`

Prints the `nr` top elements of the stack without
changing the stack in any respect.


<a name="luaqpushmeta"/>
### luaQ_pushmeta ###

`void luaQ_pushmeta(lua_State *L, int type)`

Pushes the metatable associated with Qt variants of type `type`.

Argument `type` is an integer representing 
a `QMetaType::Type`, a `QVariant::Type`,
or any type declared using the C++ macro 
[Q_DECLARE_METATYPE](http://doc.trolltech.com/4.4/qmetatype.html) 
The following expression can be used to push
a metatable for type `TYPE`:
```c++
 luaQ_pushmeta(L, qMetaTypeId&lt;TYPE&gt;());
```

`void luaQ_pushmeta(lua_State *L, const QMetaObject *mo)`

`void luaQ_pushmeta(lua_State *L, const QObject *o)`

Pushes the metatable associated with Qt object `o` or
with Qt objects whose class is represented by the 
[metaobject](http://doc.trolltech.com/4.4/qmetaobject.html)
`mo`. The following expression can be used to push
a metatable for a Qt object class `TYPE`:

```c++
  luaQ_pushmeta(L, &TYPE::staticMetaObject);
```



<a name="luaqpushqt"/>
### luaQ_pushqt ###

`void luaQ_pushqt(lua_State *L)`

Pushes the table corresponding to the package `qt`.

`void luaQ_pushqt(lua_State *L, const QVariant &var)`

Pushes a representation of the 
[Qt variant](#qt.QVariants) `var` onto the Lua stack.
Numeric types are automatically converted to Lua numbers. 
String types are automatically converted to Lua strings.

`void luaQ_pushqt(lua_State *L, QObject *obj, bool owned=false)`

Pushes a representation of the 
[Qt object](#qt.QObjects) `obj` onto the Lua stack.
By default, objects are not owned by Lua
and therefore are not automatically deleted
by the Lua garbage collector.
To make an object owned by Lua, the optional flag `owned`
should be set to `true` when calling function `luaQ_pushqt`
immediately after creating the C++ object.


<a name="luaqregister"/>
### luaQ_register ###

`void luaQ_register(lua_State *L, const luaL_Reg *l, QObject *obj)`

Registers C or C++ functins that must run in the thread of Qt object `obj`.
This function is similar to [luaL_register](..:LuaManual#luaLregister).
It creates entries in the table located on top of the stack
for the C or C++ functions specified by the array pointed by argument `l`.
When a Lua program calls these functions, execution always
happens in the thread associated with object `obj` 
using [thread hopping](#qt.qcall).

As a special case, when `obj` is null,
the declared functions are always invoked in
the thread associated with their first argument
which is assumed to be a Qt object.


<a name="luaqtoqobject"/>
### luaQ_toqobject ###

`QObject* luaQ_toqobject(lua_State *L, int i, const QMetaObject *m = 0)`

Returns a pointer to the [Qt object](#QObjects)
located at position `i` on the stack.
It returns a null pointer if the specified position
does not represent a Qt object or if its class does not 
inherit the class represented by the meta object `m`.
When `m` is null, all Qt objects are accepted.


<a name="luaqtoqvariant"/>
### luaQ_toqvariant ###

`QVariant luaQ_toqvariant(lua_State *L, int i, int type = 0)`

Converts the Lua value at position `i` of
the stack into a [Qt variant](#Qt.Qvariants) of type `type`.
When `type` is zero, the most appropriate Qt variant type is returned.
When the conversion is not possible, the function
returns a Qt variant of type `QVariant::Invalid`.


<a name="luaqtraceback"/>
### luaQ_traceback ###

`int luaQ_traceback(lua_State *L)`

`int luaQ_tracebackskip(lua_State *L, int skip)`

Augments the string located on top
of the stack with lines representing the stack trace
of the Lua interpreter. This is convenient in error handlers.
The `skip` topmost function calls are skipped.

