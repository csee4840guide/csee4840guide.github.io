---
layout: default
title: Simulation and Verification using ModelSim
---

ModelSim is a hardware simulation tool developed by Mentor Graphics which is
bundled with the Quartus II design software. Simulation of hardware units using
ModelSim is a very useful method for debugging, especially since it can
be done without the use of the FPGA and, if the student has installed Quartus II
on their own computer, without going to the crowded embedded systems lab.
Unfortunately, not much attention is given to the use of ModelSim in the class
lectures or labs, and usage of the software is not very intuitive.
Here, we will discuss how to use ModelSim more effectively, so as to reduce
the amount of frustration experienced during debugging.

## Writing VHDL Testbenches

A VHDL testbench looks something like the following

```vhdl
entity example_tb is
end example_tb;

architecture sim of example_tb is
    signal clk : std_logic := '1';
    signal a : unsigned(7 downto 0);
    signal b : unsigned(15 downto 0);
begin
    -- the entity you are testing
    EX : entity work.example port map (
        clk => clk,
        a => a,
        b => b
    );

    clk <= not clk after 10 ns;

    process
    begin
        -- main testbench code
    end process;
end sim;
```

You can add the testbench in the settings dialog under "EDA Tool Settings" ->
"Simulation" and run the testbench by starting an RTL demonstration.

### Differences between Testbench VHDL and Synthesized VHDL

One important fact about ModelSim to keep in mind is that the VHDL that
ModelSim recognizes is different than what is recognized by the Quartus
compiler. VHDL descriptions will also behave differently in the simulator than
when running on the FPGA. The most important difference is that signals on
the FPGA can only be '1' or '0', whereas in simulation they can take on
states such as 'U' for unitialized, 'Z' for high impedance, or 'X' for
unknown.

This affects the initial values of registers. On the FPGA, unitialized
registers will default to '0' on startup.
In simulation, the initial state of a signal will be 'U', or unitialized.
Therefore, it is important that you initialize the contents of registers
before using their values. This holds for signals in both the testbench and the
entities being tested. If you do not initialize registers, the results of
performing computation based on them will show up as 'X' or unknown, which is
most likely not what you want. You can specify the initial value of a signal
using the `:=` operator when declaring the signal (as done in the above example
with the `clk` signal), or you can have a reset signal that causes signals to
be assigned.

The presence of std\_logic values other than '1' and '0' in simulation also
affects compile-time behavior. Specifically, case statements and selected
signal assignments which would appear complete in the Quartus compiler will
not appear complete in the ModelSim compiler. For instance, consider the
following selected signal assignment for a full decoder.

```vhdl
with a select b <=
    "1000" when "00",
    "0100" when "01",
    "0010" when "10",
    "0001" when "11";
```

The Quartus compiler will say that this is complete, but the ModelSim compiler
will complain. This is because it will not know what value to give `b` when
`a` is unknown or unitialized. To fix this problem, always use a default in
such assignments.

```vhdl
with a select b <=
    "1000" when "00",
    "0100" when "01",
    "0010" when "10",
    "0001" when others;
```

### Waiting Around

In testbench VHDL, use the `wait` and `wait for` statements to allow time
to pass during the testbench. The `wait for` statement takes a time and a
unit and advances the simulation for that amount of time. The `wait` statement
waits until the end of the simulation. Make sure you have one of these at the
end of the main process or the process will restart once it reaches the end.

### Using Asserts

In the test bench, you can use `assert` statements to check that a signal
has the value you expect it to be. This is a much better method of verification
than simply eyeballing the signal value in the ModelSim.
For instance, if the `example` entity above takes `a` as the input and
is expected to produce on `b` the result of squaring `a` two cycles later,
we could verify that the behavior is as expected as follows.

```vhdl
process
begin
    a <= x"1b"; -- 27
    wait for 60 ns;
    assert b = x"02d9"; -- 729
    wait;
end process;
```

### Using Arrays and Loops

In general, you will probably want to test multiple sets of inputs for a
unit to make sure there aren't any strange corner cases. An easy way to set
this up in the test bench is to use two arrays, one for your inputs and another
for your outputs, and then use a loop which on each iteration feeds in an
input and checks that the output matches the expected.

```vhdl
architecture sim of example_tb is
    type input_arr_type is array(0 to 2) of unsigned(7 downto 0);
    constant inputs : input_arr_type := (x"1b", x"2a", x"25");
    type expected_arr_type is array(0 to 2) of unsigned(15 downto 0);
    constant expected : expected_arr_type := (x"02d9", x"06e4", x"0559");
begin
    for i in 0 to 2 loop
        a <= inputs(i);
        wait for 60 ns;
        assert b = expected(i);
    end loop;
    wait;
end sim;
```

## Scripting ModelSim

One of the most powerful features of ModelSim is that simulations can be
controlled through TCL scripts. This feature is never discussed in the
instructional materials for the class, so most students never become
aware of it, which is a shame since it can make debugging much easier.
A basic TCL script would look like the following.

```tcl
add wave clk
add wave a
add wave b

run 200 ns
```

This script will add three signals to the viewer and then run the simulation
for 200 ns. You can set a TCL script to be used in simulation by choosing
"Use script to set up simulation" in the simulation options.
Here are a few things you can do in TCL scripts to debugging a lot easier.

### Changing Radix of Signal

It's pretty painful to try and decipher the meaning of a 16-bit, 32-bit, etc.
signal by looking at the binary representation ModelSim shows you by default.
You can change the radix to decimal or hexadecimal in the ModelSim window
after the script has been run, but you will have to do this after every run,
which will get rather tedious. Instead, you can set the radix in the TCL script
when you add the wave. For instance, we can modify the basic script as follows.

```tcl
add wave clk
add wave -radix unsigned a
add wave -radix unsigned b

run 200 ns
```

This will cause `a` and `b` to be displayed as unsigned decimal numbers,
which makes a lot more sense than showing them as a bunch of 1s and 0s.
Some other radices which you might find useful are `decimal` for signed
decimal numbers, and `hexadecimal` for unsigned hex numbers.

### Displaying Internal Signals

By default, when running a testbench without a TCL script, only the signals
declared in the testbench will be shown in the window. Usually, however, you
will want to see the values of signals inside the entities you are testing.
You can easily add those signals in the TCL script. For instance, if the
`example` entity in our testbench has a register named `c` that we are
interested in, we can add it to the viewer as follows.

```tcl
add wave EX/c
```

Note that we use `EX`, not `example`. The names are based on the names given
to the instances in the port mappings.

You can also do things like displaying a single element of an array.
So, for instance, if `c` was an array and we only want to display element 0,
we could do the following.

```tcl
add wave {EX/c[0]}
```

The curly braces are necessary so that `[0]` isn't interpreted as indexing
into a TCL array.

You can, of course, go deeper in the hierarchy and show internal signals of
internal entities. So if `example` had inside it a port mapping named
`INNER`, and you wanted to see a register inside `INNER` named `d`, you could
do the following.

```tcl
add wave EX/INNER/d
```

### Supressing Warnings

Sometimes, you will get warnings about 'U', 'X', '-', etc. being used in an
arithmetic operation. This is generally caused by memory not being initialized
yet. If you know that these warnings do not matter, you can silence them
using the following command.

```tcl
set NumericStdNoWarnings 1
```

You may want to do this if you get a lot of these warnings and they are
obscuring the warnings and errors you actually care about.
Alternatively, if you know at what time the warnings crop up, you can
turn off the warnings just during that time period. So, for instance, if
you wanted to silence warnings between 60 ns and 80 ns...

```tcl
run 60 ns
set NumericStdNoWarnings 1
run 20 ns
set NumericStdNoWarnings 0
run 100 ns
```

## Re-running Simulation

If you make a change and want to run the testbench again, you can do so without
closing and reopening ModelSim. Simply hit the up arrow in the command window
at the bottom. You should see a line appear that says something like
`do run_simulation_script.do`. Hit enter, and the simulation will run again.

## General Advice for Debugging via Simulation

Debugging hardware is a difficult and time-consuming process. Expect to spend
most of your time on this when working on the labs and final project.
Identifying and tracking down bugs is a lot easier in ModelSim than on the FPGA,
so make sure you test thoroughly in simulation before trying to program your
hardware onto the FPGA.

Earlier, we talked about using asserts to compare the outputs to expected
values. Sometimes, the expected values are difficult or tedious to determine
by hand. In these cases, it is helpful to write a software simulation of the
computation you want the hardware to perform. Feed your desired inputs into
the simulation and use the outputs as the expected values to test your hardware.
It goes without saying, but if you follow this method and then see an assertion
failure in the VHDL testbench, check to make sure the inputs and expected
values are correct. The bug may be in your software simulation.

If you are sure the inputs and expected values are correct, but the assertions
are still failing, then it's time to track down the bug in your hardware.
Here are a few techniques you can follow to do this.

 * If the entity you are testing is a state machine, check that the state
   signal is going through the states in the expected sequence.
 * If your computation produces a bunch of intermediate values to compute the
   final result, trace the intermediate values backwards or forwards until
   you find where or when they diverge. If you're not sure what these
   intermediate values should be, try printing them out in your software
   simulation.
 * If the entity you are testing contains sub-entities whose behavior is
   non-trivial, verify that those sub-entities are acting as expected first.
