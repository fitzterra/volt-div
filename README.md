Volt-Div
========

*__Arduino__ voltage divider design for measuring voltages above 5V*

In order to use the **Arduino** analog inputs to read voltages above **5 volt**
(actually above V<sub>REF</sub>, [see here][1] for more details), we need to
design a **voltage divider** to divide the large voltage into a smaller
proportional voltage that will fall within the maximum input spec for the *ADC*
input on the *Atmel* chip.

The reading from the divider input will then be proportional to the full input
voltage we are trying to read, and then it is a simple matter of mathematics to
calculate the input voltage value.

Circuit example
---------------
An example of a voltage divider circuit is as follows:

```
Vin o------------------o---------------------------------o
                       |
                      .-.
                      | |
                      | | R1
                      | |
                      '-'
                       |        Vout   Arduino
                       o-------------> analog
                       |               input
                      .-.
                      | |
                      | | R2
                      | |
                      '-'
                       |
0V o-------------------o--------------------------------o

```

Things to take into account
---------------------------
When designing a voltage divider to be used in this scenario, the following
should be taken into account:

  * The larger the resistors, the lower the current through the R1--R2 divider,
    and the less impact it will have on the supply current requirements.
  * The value for R2 should be less than 10k for the best impedance match with
    the ADC in the Atmel chip. [See here][2] for more info.
  * At the maximum expected input voltage (Vin), the voltage drop across R2
    should be as close as possible to +5V to obtain the best resolution readings
    for the voltage measurement. The ADC has a 10-bit resolution for a total of
    1024 possible step values (2<sup>10</sup>) between 0V and 5V, or a voltage
    value of (5v/1024) 4.88mV per step.... but see below.
  * The Arduino supply voltage is not always guaranteed to be exactly 5V and
    will depend on the supply source (USB, batteries, other). Since the analog
    value depends on the supply voltage, calculating an accurate voltage value
    from the analog input value requires the actual supply voltage to be known.
    [See here][4] for a method that can be used to for the Arduino to accurately
    measure it's own supply voltage.

Design
------
In order to calculate the best values for the voltage divider, the **maximum
possible Vin value** should be known. We will use an example voltage of 18V
here.

Since the divider values will be calculated to get Vout as close as possible to
5V with Vin at it's max value, it is possible that Vout will go above the max
allowed ADC input value of 5V (or rather Vcc) should Vin exceed the max expected
voltage. When in doubt, rather allow for a slightly larger Vin max value and loose
a tiny bit of accuracy that loosing an *Arduino*!

Start by selecting a value for R2 less than 10k, but still high enough to minimize
the current through the divider. A value between 4k7 and 8k2 should be goog for this
resistor.

So now we have:

  * Vin max = 18V (for exmaple, use your own value here)
  * R2 = 8k2

Now let's use an [EIR table][3] to solve for all voltage, current and resistor
values in the divider circuit. This is what we know so far:

|     | **R1** | **R2** | **Tot** |
|-----|-------:|-------:|--------:|
|**E**|   ~13V |    ~5V |     18V |
|**I**|        |        |         |
|**R**|        |    8k2 |         |

**Note** that the resistor voltages are aproximations at this stage, and will
only be properly calculated once we know the actual resistor value we will be
using for R1. The resistor value must be the closest standard value we can get
based on the ideal value calculated below.

From here we can calculate the current through R2:

  Ir2 = Vr2/R2 = 5/8k3 = 0.625mA

Since the same current flows through R1, we can calculate the ideal value for R1

  R1 = Vr1/Ir1 = 13/0.625mA = 20.8k

Since the next highest standard E24 series resistor is 22k, we will use that as
the value for R1 and can now complete the **EIR** table with the correct values:

|     | **R1** | **R2** | **Tot** |
|-----|-------:|-------:|--------:|
|**E**|13.113V | 4.887V |     18V |
|**I**|0.596mA |0.596mA | 0.596mA |
|**R**|    22k |    8k2 |    30k2 |
|**P**|7.815mW |2.913mW |10.728mW |

From this table, the following is clear:

  * The total current through the divider is less than 600&micro;A - acceptable
  * Total power disipation is less than 11mW
  * The resistor power ratings can be as low as 1/8W
  * We have a slight margin (2.3% actually) that Vin can go higher than expected
    due to Vout (Vr2) only being 4.887V and not a full 5V.
  * We loose a very tiny bit in measurement resolution because Vout will max out
    at 4.887V at the full Vin. This should be negligible in this cases though.


Measurement code and calculations
---------------------------------
When using a properly designed voltage divider as above, the following facts are
known:

  * The input voltage to the *ADC* would never exceed the maximum allowed
    voltage (as long as the Vin value was correctly specced)
  * Irrespective of the values of the resistors, the Vout voltage (input to the
    ADC) will always be a fixed ratio of the value of Vin.

The equation for calculating the actual Vout voltage value based on the 10-bit
ADC reading at Vout is:

```
             Vcc
Vout = Ain x ----
             1024
```

where:

  * **Ain** = the analog input value read from the ADC
  * **Vcc** = the supply voltage at the time of the reading (~5V)
  * **1024** = the ADC resolution

The equation for calculating Vin from Vout is based on the resistor ratio:

```
             R1 + R2
Vin = Vout x -------
                R2
```

Combining these equations into one, we get:

```
      Ain x Vcc x (R1 + R2)
Vin = ---------------------
          1024  x  R2
```

**Note**:
Since floating point calculations are expensive and could cause inaccuracies in
certain circumstances, and also to avoid multiple roundings which could also
reduce accuracy, it would be better to use integer math to calculate the numerator
and denominator seperately, and then do the division.
Care must be taken to prevent integer overflows in this case though, since the
calculated values could be quite large.

Another option to still maintain a fair level of accuracy and avoid floating point
calculations altogether, is to use milliVolt instead of Volt:


```
      Ain x Vcc x 1000 x (R1 + R2)
Vin = ----------------------------
         1024  x  1000  x  R2
```

When doing this, the division can be integer division and the accuracy would still
be to the closest milliVolt without using floting point values and avoiding any
further rounding errors.

------------------------------------------------------------------------------

[1]: http://forum.arduino.cc/index.php/topic,13395.0.html
[2]: http://forum.arduino.cc/index.php/topic,15631.0.html
[3]: http://www.allaboutcircuits.com/vol_1/chpt_5/2.html
[4]: https://code.google.com/p/tinkerit/wiki/SecretVoltmeter
