Volt-Div
========

*__Arduino__ voltage divider design for measuring voltages above 5V*

In order to use the **Arduino** analog inputs to read voltages above **5 volt**
(actually above V<sub>REF</sub>, [see here][1] for more details), we need to
design a **voltage divider** to divide the large voltage into a smaller
proportional voltage that will fall within the maximum input spec for the *ADC*
input on the *Atmel* chip.

The reading from the divider input will then be proportional to the true voltage
we are trying to read, and then it is a simple matter of mathematics to
calculate the the true voltage value.

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
When creating a voltage to be used in this scenario, the following should be
taken into account:

  * The larger the resistors, the lower the current through the R1--R2 divider,
    and the less impact it will have on the supply current requirements.
  * The value for R2 should be less than 10k for the best impedance match with
    the ADC in the Atmel chip. [See here][2] for more info.
  * At the maximum expected input voltage (Vin), the voltage drop across R2
    should be as close as possible to +5V to obtain the best resolution readings
    for the voltage measurement. The ADC has a 10-bit resolution for a total of
    1024 possible step values (2<sup>10</sup>) between 0V and 5V, or a voltage
    value of (5v/1024) 4.88mV per step.

Design
------
In order to calculate the best values for the voltage divider, the **maximum
possible Vin value** should be known.

Since the divider values will be calculated to get Vout as close as possible to
5V with Vin at it's max value, it is possible that Vout will go above the max
allowed ADC input value of 5V should Vin exceed the max expected voltage. When
in doubt, rather allow for a slightly larger Vin max value and loose a tiny bit
of accuracy that loosing arduino!

A good starting point is to select a value for R2 less than 10k, but still high
enough to minimize the current through the divider. I normally for a value of
8k2 for this resistor.

So now we have:

  * Vin max = 18V (for exmaple)
  * R2 = 8k2

Now let's use an [EIR table][3] to solve for all voltage, current and resistor
values in the divider circuit. This is what we know so far:

|     | **R1** | **R2** | **Tot** |
|-----|-------:|-------:|--------:|
|**E**|   ~13V |    ~5V |     18V |
|**I**|        |        |         |
|**R**|        |    8k2 |         |

From here we can calculate the current through R2:

  Ir2 = Vr2/R2 = 5/8k3 = 0.625mA

Since the same current flows through R1, we can calculate the value for R1

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
When using a properly designed voltage divider as above, the following should be
true:

  * The input voltage to the *ADC* would never exceed the maximum allowed
    voltage (as long as the Vin value was correctly specced)
  * Irrespective of the values of the resistors, the Vout voltage (input to the
    ADC) will always be a fixed ratio of the value of Vin.

The equation for calculating the actual Vout voltage value based on the 10-bit
ADC reading at Vout is:

```
             Vr2
Vout = Ain x ----
             1024
```

where:

  * **Ain** = the analog input value read from the ADC
  * **Vr2** = the max expected Vr2 value (4.887V in this case)
  * **1024** = the ADC resolution

The equation for calculating Vin from Vout is based on the resistor ratio:

```
             R1 + R2
Vin = Vout x -------
                R2
```
By substitution and simplification of the above 2 equations, we can derive the
following equation for finding Vin based on the max Vin value we specified above
and the analog value read from the ADC:

```
      Ain x Vmax    
Vin = -----------
         1024
```

where:

  * **Ain** = the analog input value read from the ADC
  * **Vmax** = the max expected Vin value (18V in this case)
  * **1024** = the ADC resolution

**Note**:
Since floating point calculations are expensive and could cause inaccuracies in
certain circumstances, and also to avoid multiple roundings which could also
reduce accuracy, it is suggested that Vin be calculated by first multiplying Ain
by Vmax and then dividing the result by 1024.0 (note the trailing .0). The final
value will be a floating point value representing the value for Vin with the
minimum loss of accuracy through multiple roundings and floating point
calculations.

------------------------------------------------------------------------------

[1]: http://forum.arduino.cc/index.php/topic,13395.0.html
[2]: http://forum.arduino.cc/index.php/topic,15631.0.html
[3]: http://www.allaboutcircuits.com/vol_1/chpt_5/2.html
