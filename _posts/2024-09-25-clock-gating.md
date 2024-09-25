---
layout: post
title:  "ASIC Clock Gating and Latches"
date:   2024-09-25
---

*Adapted from a rambling series of posts in the [TinyTapeout](https://tinytapeout.com/) discord*

I was talking to [Lofty](https://github.com/Ravenslofty/) recently about how
some common uses of flip-flops can be optimised with clock gating and sometimes
replaced with latches for power consumption and area improvement on ASIC.
I don't think these techniques are super well known in the open source community
so I thought I would share them for reference.

# What is a clock gate?

The general type of circuit we want to optimise are flip-flops with enables
(we ignore resets for now to keep things simple). These are flops that hold
their value unless an enable goes high in which case they take an input value
(shown below either with a mux to select the value each clock cycle or a special
`DFFE` cell which does the same thing in one cell):

![
	Two representations of a flip-flop with enable, one with an explicit multiplexer
	for the enable condition and another as a single integrated cell
]({{ "/assets/images/clock-gating/dffe.png" | relative_url }})

These are common structures in digital design, for example in CPUs registers in
the register file may look like this, or memory mapped control registers. An
important thing to note is that often a number of these structures share the
same enable (for example, if you have a 32-bit control register that can be
written from a bus, each of the 32 flops it is made of has the same enable
signal), or the same input signal (for example, if you have two different
control registers that are written from the same bus, the LSB of each control
register has the LSB of the bus's data wire as their input).

But we can optimise these designs! Even when the value doesn't change, toggling
the clock input every cycle uses some amount of dynamic power consumption, and
actually we only need to trigger the DFF's clock input when the enable is high,
at any other time we know the value is going to stay the same. To do this, we
can gate the clock input, so that it only ever toggles when enable is high.
This can be done with something like the following verilog:

```verilog
assign gated_clk = clk & en;
```

Unfortunately this has a slight issue - if enable changes while clk is high,
the resulting clock pulse could have glitches or be late. To get around this
we use a latch on the enable signal so that its value only ever changes when
the clock is low, preventing glitches. This whole circuit is normally integrated
into one standard cell called an Integrated Clock Gating cell (ICG) so it can
be easily characterised:

![
	An integrated clock gating cell, showing the enable signal latched on the
	inverted clock input, then fed into an AND gate with the clock to prevent
	glitches
]({{ "/assets/images/clock-gating/icg.png" | relative_url }})

Using this we can remove the mux from our original circuit, instead driving the
DFF's D input directly and using an ICG to gate the clock with the enable signal:

![
	An ICG gating a clock with an enable signal has its output fed into the clock
	input of a standard D flip-flop
]({{ "/assets/images/clock-gating/icg_to_dff.png" | relative_url }})

# Reducing area with clock gating

This reduces power consumption as the clock toggles at the flip-flop less, but
it also reduces area? This comes about when a number of flip-flops share the
same enable, like with the 32 bits of a control register. Before this
optimisation each bit requires its own mux (needing 32 muxes):

![
]({{ "/assets/images/clock-gating/bus_before_icg.png" | relative_url }})

but now we can instead use just one ICG for all of the flops as they all need
the same gated clock, meaning we replaced 32 muxes with 1 ICG!

![
]({{ "/assets/images/clock-gating/bus_after_icg.png" | relative_url }})

# How do latches come into it?

We can replace our enabled flop with a slightly more complicated structure.
Here, the flip-flop stores the input value each clock cycle, and when the enable
signal is high, the D-latch passes the flopped value through transparently while
the gated clock is high, then saves that value when it goes low, acting as a
regular enabled flip-flop.

![
]({{ "/assets/images/clock-gating/icg_latch.png" | relative_url }})

But this uses more logic than the clock gated DFF for the same function, so
how does this save in area? The savings rely once again on amortised cost.
You will notice now that the value at the output of the DFF depends only on
the input and the clock. This means that it can be reused for multiple flops
that read from the same wire, even if they have different enable signals.
Consider the unoptimised circuit below:

![
]({{ "/assets/images/clock-gating/bus_before_latch_icg.png" | relative_url }})

This could correspond to the logic for the LSB of a number of registers driven
from the same bus. Each has a different enable signal (perhaps each is high
when the bus' address is set to different values). Because every input is the
same, when we convert it to the version using a latch, we only need one DFF.
Therefore this needs 1 DFF and N latches, whereas the previous version needed
N DFFs. As latches are smaller than DFFs, once you have a few driven from the
same input like this, the area savings of using latches instead of DFFs outweighs
the cost of that extra DFF!

![
]({{ "/assets/images/clock-gating/bus_after_latch_icg.png" | relative_url }})

# Can we automate this?

These are quite common optimisations that commercial tools can perform
automatically (at least the clock gating), and can be partially be done in Yosys
using [Lighter](https://github.com/AUCOHL/Lighter) which pattern matches on DFFEs,
replacing them with this ICG structure. There is
[ongoing work](https://github.com/YosysHQ/yosys/pull/4583) to natively add
these optimisations to Yosys with better heuristics to detect when they improve area,
so hopefully this will be an easy improvement the user never even has to think
about.
