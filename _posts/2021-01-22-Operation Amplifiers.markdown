---
layout: post
title:  "Lecture 9: Operation Amplifiers"
date:   2021-01-22 01:42
categories: ADC

---

<h1>ADC Lecture 9 - Operational Amplifiers</h1>

An op-amp is a circuit with two inputs and one output. 

It is a **differential** amplifier, which means that the output voltage is 

**proportional** to the **difference between the input voltages**:

\\[Y = A(V_{+} - V_{-})\\]


The gain, *A* is usually very large at low frequencies: e.g. *A = 10^5* 

The input currents are very small: e.g. +- 1 nA.

Internally it is a complicated circuit , but we can forget about that for now, and treat it as an **almost perfect dependent voltage source.** 

Most Op-Amps are packaged in DIL (dual in-line) or surface mount packages. The image shows a very common pin-out for individually packaged op-amps.



<h4>Negative Feedback</h4>

**Negative feedback** is when the occurrence of an event causes something to happen that counteracts the original event.

For example: in a central heating system, if the temperature falls too low the thermostat turns the heating on; when the temperature rises above a certain threshold the thermostat turns the heating off again.

Negative feedback is employed in most op-amp circuits.

In the circuit shown, if the op-amp output Y falls, then \\(V_{-}\\) will fall by the same amount so that \\(V_{+} - V_{-}\\) will increase. This causes Y to rise since:

\\[Y = A(V_{+} - V_{-}) \n Y = A(X-Y)\\]

![Negative Feedback](/Imperial/ADC/Lec9_1.PNG) 

