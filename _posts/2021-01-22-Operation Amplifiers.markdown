---
layout: post
title:  "Lecture 9: Operation Amplifiers"
date:   2021-01-22 01:42
categories: ADC

---

An op-amp is a circuit with two inputs and one output. 

It is a **differential** amplifier, which means that the output voltage is 

**proportional** to the **difference between the input voltages**:

\\[Y = A(V_{+} - V_{-})\\]


The gain, *A* is usually very large at low frequencies: e.g. \\(A = 10^5\\)

The input currents are very small: e.g. \\(\\pm 1 nA\\).

Internally it is a complicated circuit , but we can forget about that for now, and treat it as an **almost perfect dependent voltage source.** 

Most Op-Amps are packaged in DIL (dual in-line) or surface mount packages. The image shows a very common pin-out for individually packaged op-amps.



<h4>Negative Feedback</h4>

**Negative feedback** is when the occurrence of an event causes something to happen that counteracts the original event.

For example: in a central heating system, if the temperature falls too low the thermostat turns the heating on; when the temperature rises above a certain threshold the thermostat turns the heating off again.

Negative feedback is employed in most op-amp circuits.

In the circuit shown, if the op-amp output Y falls, then \\(V_{-}\\) will fall by the same amount so that \\(V_{+} - V_{-}\\) will increase. This causes Y to rise since:

\\[Y = A(V_{+} - V_{-})\\] \\[Y = A(X-Y)\\]

\\[Y(1+A) = AX => Y = \\frac{A}{A+1} \\approx X\\] <center>for large A.</center>

![Negative Feedback](/Imperial/ADC/Lec9_1.PNG) 

Golden Rule : **Negative feedback adjusts the output to make** \\(V_{+} \\approx V_{-}\\)





<h4>Analysing Op-amp Circuits</h4>

Nodal analysis is simplified by making some assumptions.

**Note**: The Op-Amp needs power supply connections which are usually omitted from the schematic; traditionally dual rail(-ve and -ve) supplies were common, but nowadays there are many op-amps requiring only a single supply rail. **The currents out of the Op-Amp only sum to zero(KCL) if the power supply connections are included.**

1. Check for negative feedback: to ensure that an increase in Y makes \\(V_{+} - V_{-}\\) decrease, Y must be connected (usually via other components) to \\(V_{-}\\).
2. Assume \\(V_{+} = V_{-}\\): Since \\((V_{+} - V_{-}) = \\frac{Y}{A}\\), this is the same as assuming that \\(A \\rightarrow \\infty \\).
3. Assume zero input current: in most circuits, the current at the op-amp input terminals is much smaller than the other currents in the circuit, so we assume it is zero.
4. Apply KCL at each op-amp input node separately(input currents = 0)
5. **Do NOT apply KCL at output node** (output current is unknown). 





<h4>Non-inverting Amplifier</h4>

Circuit has input voltage X and output voltage Y. The circuit gain = \\(\\frac{Y}{X}\\)

Applying steps 1 to 3:

1. Negative feedback OK
2. \\(V_{-} = V_{+} = X\\)
3. Zero input current at \\(V_{-}\\) means \\(R_{2}\\) and \\(R_{1}\\) are in series and form a voltage divider

So \\(X = \\frac{R_{1}}{R_{1}+R_{2}} Y \\). 

Hence,

\\[Y = \\frac{R_1 + R_2}{R_1} X = (1+\\frac{R_2}{R_1})X = +4X \\]

![noninv](/Imperial/ADC/Lec9_NonInv.PNG)
