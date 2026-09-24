# Safety behaviour

CraftBeerPi switches multi-kilowatt heating elements. This page describes what
the server does when something goes wrong, what it does when it is merely
*unsure*, and which of those behaviours you can configure.

The principle running through all of it: **when the software is confused, the
element goes cold.** A controller that cannot trust its inputs must not act on
them, and an unreadable number must never reach an element as heat.

{% hint style="info" %}
Every protective feature here is **off or permissive by default**. A safety
feature that silently changes what a working rig does is not a safety feature —
it is an unannounced change to your brew day. Each one is opt-in, and each says
so below.
{% endhint %}

## Fail-safe direction

Several places in the actor path have to decide what to do with a value they
cannot interpret. They all resolve the same way: toward **no heat**.

| Situation | Behaviour |
| --- | --- |
| Power is unreadable (`None`, empty, text, `NaN`) | Commanded as **0 %**, logged at ERROR |
| Power is out of range | Clamped to 0–100 |
| Power was deliberately set to `0` | Honoured as 0, **not** treated as "unset" |
| An actor has never been commanded | Starts at 100 %, as before |

A commanded `0` being read as "no value given" and replaced with `100` is a
specific hazard worth naming: an actor ramped down to zero would return at
**full power** the next time anything switched it on without naming a level.

### Switching off is unconditional

`off()` always sends the command to the hardware, even when the server already
believes the actor is off, and it reports whether the actor is *still* reading
as on afterwards.

This matters because the moments you most need an emergency stop are exactly
the moments the server's belief may be wrong — after a restart, after a failed
switch-on, or when a relay has stuck. An OFF that skipped the hardware because
it thought there was nothing to do would return success having sent nothing.

## Trusting a sensor reading

A temperature reading is only useful if it is real and recent. Three separate
things are checked before a reading is allowed to drive an element.

### A sensor never reports a value it has not measured

Sensors start with **no reading at all** rather than a placeholder. Before its
first successful read an `HTTPSensor` or a `OneWire` sensor reports nothing,
and any controller asking for a temperature is told there is none.

This used to be a placeholder number, and the direction of the resulting
failure depended on which sensor you happened to use — which is the worst
possible property for a safety default.

### Staleness follows the sensor's own cadence

A reading is treated as stale after **three missed publications**, or 30
seconds, whichever is longer.

That matters because sensors publish at very different rates. The bundled
OneWire sensor defaults to an `Interval` of **60 seconds**:

| Sensor `Interval` | Reading allowed to age |
| --- | --- |
| not set / 1 / 5 / 10 s | 30 s (the floor) |
| 30 s | 90 s |
| 60 s (OneWire default) | 180 s |

{% hint style="warning" %}
A fixed 30-second limit would treat a perfectly healthy OneWire probe on its
default interval as dead for roughly half of every minute — cutting the element
and raising a fault-then-recovered pair every minute of a brew day. That is how
a safety check becomes the thing that ruins a brew, and then gets switched off,
taking the real protection with it.
{% endhint %}

### A frozen probe is not a working one

Several sensor drivers keep publishing their last value when a read fails, so a
disconnected probe reports a plausible temperature indefinitely. The bundled
OneWire driver stops refreshing its timestamp when reads fail, so the reading
ages out and consumers reject it — a gap in the chart is honest where a flat
line is not.

## Element interlock

Nothing in stock CraftBeerPi limits how many elements are energized at once. A
profile moving from mash to boil, or a brewer starting one kettle while another
runs, can command two 5.5 kW elements together — comfortably past a 30 A
supply.

Set **`ACTOR_INTERLOCK_MODE`** in Settings:

| Value | Behaviour |
| --- | --- |
| `Off` *(default)* | No interlock. Any number of elements may run together. |
| `One heating element at a time` | Only one kettle heater may be energized. |

The group is **derived from your kettles**, not typed in: every kettle already
names the actor that heats it, so there is nothing to configure and renaming an
actor cannot break it.

{% hint style="success" %}
If two kettles share one heater actor — normal on a HERMS, where the HLT
element heats the mash through a coil — they are correctly treated as the same
element and do not block each other.
{% endhint %}

Fermenter heaters are deliberately excluded: they are tens of watts, and
interlocking a fermentation chamber against a boil kettle would suspend
fermentation control for an entire brew day.

When a start is refused, the other element is **left running** and you are told
why. Switching off another vessel's element behind your back would be a worse
failure than declining to start this one.

## Dry-fire protection

An element in an empty or nearly empty kettle heats extremely fast, because
there is almost no thermal mass to absorb the energy. `PIDBoil` can detect that
rate of rise and shut down.

Set both properties on the boil kettle to enable it:

| Property | Meaning |
| --- | --- |
| `Volume_Litres` | Litres normally in the vessel. `0` disables the check. |
| `Element_Watts` | Element rating in watts. `0` disables the check. |

From those two numbers the expected rate of temperature rise is known. A rise
much faster than physically possible for that volume means the liquid is not
there, and the element is switched off with an ERROR notification.

## Heat-stall watch

Steps that wait for a target temperature loop with no upper bound, so a failed
element, a dead SSR or a probe that has fallen out of its thermowell would leave
the rig commanding heat indefinitely with nobody told.

The watch reports a stall when the temperature stops rising while heat is being
called for. Three things stop it crying wolf:

* the window restarts whenever the temperature actually rises, so a slow
  heat-up on a cold day never trips it;
* it stays quiet within about a degree of target, because a controller *holding*
  setpoint produces no rise at all;
* it asks the kettle logic whether heat can currently flow, and stays quiet if
  the answer is no.

{% hint style="info" %}
That last point exists because of HERMS. A mash tun has no element of its own —
it is heated through a coil in the HLT. After a step change the HLT is still
where the last rest left it, so the mash **drifts down** before it climbs.
Measured on a running rig, the alarm used to fire 21.5 °F short of a 168.8 °F
target on a step that then reached target and completed normally.
{% endhint %}

## Confirm before boil

`CONFIRM_BEFORE_BOIL` (Settings, default **Yes**) makes a boil step wait for you
to confirm the transfer before it begins heating.

A kettle left hot from a previous step would otherwise satisfy the boil step's
target check immediately and begin timing a boil that is not happening — in a
vessel the wort has not been transferred into yet.

## Actions are declared, not arbitrary

Plugin actions — the buttons on an actor, sensor or step — are invoked by name
over HTTP. Only methods **declared** with the `@action` decorator can be called.

This is the same set the interface offers you, so nothing you can legitimately
use is affected. It prevents a request naming any other attribute of the object.

## What is *not* protected

{% hint style="danger" %}
**The control API does not require authentication.** Anything that can open a
network connection to the server can switch actors on, change setpoints, and
start or stop a profile.

This is how CraftBeerPi has always worked, and the interface depends on it.
Treat the rig as a device that belongs on a **trusted, private network**. Do not
expose it directly to the internet; if you need remote access, put it behind a
VPN.
{% endhint %}

Other limits worth knowing:

* Protection is only as good as the probe. A sensor reading the wrong vessel
  will be trusted, because nothing can tell the server it is in the wrong place.
* Dry-fire detection needs `Volume_Litres` and `Element_Watts` to be right. A
  volume set far too high makes the check far too slow to matter.
* The interlock governs what the **software** commands. It cannot know about an
  element energized by a manual switch.
