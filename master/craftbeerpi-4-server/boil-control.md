# Boil control (PIDBoil)

`PIDBoil` heats a boil kettle in two distinct regimes, because **temperature
stops being a controllable variable once the wort boils.** At a rolling boil the
probe reads the boiling point no matter how hard the element is driven — the
energy goes into vaporising water, not into raising temperature. What you
control from that point on is not temperature but **boil vigour**, and vigour is
power.

| Regime | What is controlled | How |
| --- | --- | --- |
| Heating up | Temperature | PID, capped at `Max_Output` |
| Boiling | Vigour | Fixed power, `Max_Boil_Output` |

## Getting to the boil

| Property | Meaning |
| --- | --- |
| `P` `I` `D` | PID gains, in **heater percent per degree** |
| `SampleTime` | How often the output is recalculated (2 or 5 s) |
| `Max_Output` | Power ceiling before the boil threshold is reached |

## Detecting the boil

Two independent signals, because a threshold alone is fragile.

| Property | Meaning |
| --- | --- |
| `Boil_Threshold` | Temperature at which fixed-power mode begins (default 98 °C / 208 °F) |
| `Boil_Plateau_Minutes` | Also treat a stalled temperature at near-full power as boiling, after this many minutes. `0` disables. |

The plateau detector exists because a probe that reads two degrees low — or a
kettle at altitude, where water boils cooler — never reaches the configured
threshold. The wort boils, the temperature stops climbing, and a
threshold-only controller holds the element at 100 % indefinitely against a
vigorous boil.

{% hint style="info" %}
Once fixed-power mode latches it **stays** latched until the wort genuinely
comes off the boil. Re-running detection while already in fixed-power mode
would clear it — the reduced demand no longer looks like "near-full power" — and
the controller would saw-tooth between 100 % and `Max_Boil_Output`.
{% endhint %}

## Controlling the boil

`Max_Boil_Output` (default **85 %**) is the power held once boiling is detected.
This is the setting to adjust for boil vigour:

* too vigorous — boil-off is high, hops scorch, the kettle threatens to boil
  over. Reduce it.
* too gentle — poor hot break, less DMS driven off. Increase it.

It is a percentage of your element, so the right value is entirely specific to
your kettle volume and element rating. A 5.5 kW element in 20 L needs far less
than the same element in 60 L.

## PID gains and temperature units

{% hint style="danger" %}
**A PID gain in percent-per-degree is unit-dependent.** One Celsius degree is
1.8 Fahrenheit degrees, so the same numeric gain applied to a Fahrenheit error
is **1.8× more aggressive** than intended.
{% endhint %}

The historical defaults — `P=117.0795`, `I=0.2747`, `D=41.58` — are Celsius
values. On a Fahrenheit server, `P=117` works out to a proportional band of

$$
\frac{100\%}{117} \approx 0.85\ ^\circ F
$$

which means the element runs flat out until it is within about **one degree** of
setpoint and then slams off. The usual complaint is *"the heater never throttles
as it approaches the set point"* — that is this.

Converted, the band becomes a far more usable 1.54 °F.

### Automatic conversion, and why your saved values are not rewritten

CraftBeerPi persists property values into the kettle when the configuration is
saved, so those Celsius defaults are sitting in most existing configurations as
*explicitly configured* values — not as defaults that can simply be changed.

On a Fahrenheit server, if **all three** of `P`, `I` and `D` match the old
Celsius defaults, the plugin runs the converted equivalents and tells you:

```
PIDBoil found persisted Celsius default gains on a Fahrenheit server and is
running converted effective gains: P=65.0442, I=0.1526, D=23.1000.
The saved props were not rewritten; edit P/I/D if these are not intended.
```

A value you tuned yourself is never touched — only the exact default triplet is
recognised. And the saved configuration is deliberately left alone: a gain
change on a boil kettle is a safety-relevant change, and it should be visible
rather than silent.

{% hint style="success" %}
**To stop the message,** write the converted values into the kettle:
`P=65.0442`, `I=0.1526`, `D=23.1`. Then what is in your configuration is what is
running, with no indirection.
{% endhint %}

## Dry-fire protection

| Property | Meaning |
| --- | --- |
| `Volume_Litres` | Litres normally in the vessel. `0` disables. |
| `Element_Watts` | Element rating in watts. `0` disables. |

Both are needed. From them the physically possible rate of temperature rise is
known:

$$
\frac{dT}{dt} = \frac{P}{m \cdot c_p}
$$

A rise much faster than that volume allows means the liquid is not there. The
element is switched off and an ERROR notification is raised.

{% hint style="warning" %}
Set `Volume_Litres` to a realistic **minimum** working volume, not your maximum.
A value set too high makes the expected rate too slow, and the check too
sluggish to protect anything.
{% endhint %}

## Sensor handling

A reading is rejected — and no heat commanded — when it is missing,
non-numeric, or older than the sensor's own allowance. That allowance follows
the sensor's publish cadence rather than a fixed number of seconds; see
[Safety behaviour](safety.md#staleness-follows-the-sensor-s-own-cadence).

This matters for a boil kettle specifically: a probe wrongly judged stale gets
the element cut and restarted repeatedly, which is exactly the pattern dry-fire
detection exists to distinguish from a real dry-fire.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Element never throttles approaching setpoint | Celsius gains on a Fahrenheit server — see above |
| Never enters fixed-power mode; sits at 100 % boiling | `Boil_Threshold` above what the probe reads; let the plateau detector cover it |
| Output saw-tooths between 100 % and 85 % | Boil detection un-latching — fixed in current builds |
| Boils over | `Max_Boil_Output` too high for the volume |
| Dry-fire trips on a full kettle | `Volume_Litres` too low, or `Element_Watts` too high |
| Element cut every minute with a healthy probe | Old fixed 30 s staleness limit against a 60 s OneWire interval |
