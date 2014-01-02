---
layout: default
title: Tips for Writing VHDL
---

In Embedded Systems Design, you will write hardware descriptions for the FPGA
using VHDL. Some of you may have used VHDL before in Advanced Logic Design.
In Embedded Systems, you will create much more complex hardware descriptions
than in ALD, so it is important to get a better grasp of the language's
features and pitfalls. Professor Edward's slides do a good job of explaining
the language syntax, so this section will focus on mistakes to avoid and
advanced features that can save you time (and prevent you from getting
cramped fingers).

## Watch out for Inferred Latches

Remember commandment IV of VHDL, "Thou Shalt Assign All Outputs".
If you don't, the Quartus compiler will infer a level-sensitive latch.
Edward's slides mention this in the context of performing combinational
logic in a `process` statement, but this can also happen in a selected
signal assignment or conditional signal assignment statement.

```vhdl
with a select b <=
    "001" when "00",
    "010" when "01",
    "100" when "10";
```

The above code is for a 2-to-3 full decoder. However, what happens when the
input is "11"? In that case the previous value will be used, so this
statement synthesizes a latch instead of a decoder. You can avoid inferring
latches by always putting an `others` clause in your selected or conditional
assignments.

```vhdl
with a select b <=
    "001" when "00",
    "010" when "01",
    "100" when "10",
    "000" when others;
```

## Get the Timing Right

Timing of signals will be the cause of most of the bugs in your VHDL
descriptions. A lot of the confusion with timing arises from how timing works
in edge-sensitive process statements. Here's a few examples that will clear
up what changes when.

Assume that here, `a` is an input, `b` is an internal signal, and `c` is an
output. Also assume that `a` only changes on the rising edge of the clock.

First consider what happens when `b` is assigned combinationally from `a` and
`c` is assigned from `b` in a process statement.

```vhdl
b <= a;
process (clk)
begin
    if rising_edge(clk) then
        c := b;
    end if;
end process;
```

In this case, `b` will change as soon as `a` changes. However, since the
change comes in on a rising edge, `c` does not detect the change until a
cycle later.

<object type="image/svg+xml" data="svg/timing-example1.svg"></object>

Note that we use the `:=` assignment operator here. This will cause the value
of `c` to change as soon as the change to `b` becomes visible in the process
statement. If instead we use the `<=` assignment operator, the value of `c`
would change on the next rising edge.

```vhdl
b <= a;
process (clk)
begin
    if rising_edge(clk) then
        c <= b;
    end if;
end process;
```

<object type="image/svg+xml" data="svg/timing-example2.svg"></object>

That is the key difference between `:=` and `<=`. The `:=` operator makes the
change on the same cycle, and the change is visible to assignments in the
process statements on the same cycle. The `<=` operator makes the change on
the next rising edge, and the change is visible on the next cycle.
To see how that affects timing, let's consider a few examples where `b` is
assigned inside the process statement.

```vhdl
process (clk)
begin
    if rising_edge(clk) then
        b := a;
        c := b;
    end if;
end process;
```

The change in `a` is not visible until one cycle after.
Since the assignments to `b` and `c` both use `:=`, both signals will change
one cycle after `a`.

<object type="image/svg+xml" data="svg/timing-example3.svg"></object>

If instead, we use `<=` for `b`, the change will not be visible to `c`
until the next cycle.

```vhdl
process (clk)
begin
    if rising_edge(clk) then
        b <= a;
        c := b;
    end if;
end process;
```

<object type="image/svg+xml" data="svg/timing-example4.svg"></object>

Of course, if `<=` is used for both signals, the delay from `a` changing to
`c` changing becomes longer still.

```vhdl
process (clk)
begin
    if rising_edge(clk) then
        b <= a;
        c <= b;
    end if;
end process;
```

<object type="image/svg+xml" data="svg/timing-example5.svg"></object>

The guide suggests that you stick to combinational assignment and the `<=`
operator in process statements for inputs and outputs to an entity.
Use `:=` only when you need an intermediate signal in a single-cycle operation.
In general, the `:=` operator can be replaced by combinational assignment,
but use whatever is convenient for you.

## Detecting Edges

In your design, you will often want to detect edges of asynchronous signals
or synchronous signals changing on a different clock. An example of this
would be detecting when a push button is pressed or released. One way to
detect an edge is to just put the edge in your process statement's sensitivity
list, but this is generally not a good idea. Remember, the second commandment
of VHDL is "Thou Shalt be Synchronous". Fortunately, there is a straighforward
technique for detecting edges synchronously.

```vhdl
entity edge_detector is
    port (clk : in std_logic;
          async : in std_logic;
          rising : out std_logic;
          falling : out std_logic);
end edge_detector;

architecture rtl of edge_detector is
    signal cur_level : std_logic;
    signal prev_level : std_logic;
begin
    process (clk)
    begin
        if rising_edge(clk) then
            cur_level <= async;
            prev_level <= cur_level;
        end if;
    end process;

    rising <= cur_level and (not prev_level);
    falling <= (not cur_level) and prev_level;
end rtl;
```

This unit will set `rising` and `falling` high for one cycle after a rising
edge or falling edge, respectively, on the `async` input.

## Using For-Generate Statements

When writing your hardware description, there are times when you will want
to replicate several identical hardware units. For instance, say you have
an adder entity, and you want to create an entity that takes four inputs,
adds some value to each of them, and produce four outputs. It would be
pretty tedious to type out four separate port mappings. You can save yourself
the trouble by using a for/generate statement.

```vhdl
entity par_adder is
    port (a0 : in unsigned(7 downto 0);
          a1 : in unsigned(7 downto 0);
          a2 : in unsigned(7 downto 0);
          a3 : in unsigned(7 downto 0);
          b  : in unsigned(7 downto 0);
          s0 : out unsigned(7 downto 0);
          s1 : out unsigned(7 downto 0);
          s2 : out unsigned(7 downto 0);
          s3 : out unsigned(7 downto 0));
end par_adder;

architecture rtl of par_adder is
    type byte_array_type is array(0 to 3) of unsigned(7 downto 0);
    signal a_arr : byte_array_type;
    signal s_arr : byte_array_type;
begin
    a_arr(0) <= a0;
    a_arr(1) <= a1;
    a_arr(2) <= a2;
    a_arr(3) <= a3;

    s0 <= s_arr(0);
    s1 <= s_arr(1);
    s2 <= s_arr(2);
    s3 <= s_arr(3);

    ADDGEN : for i in 0 to 3 generate
        ADD : entity work.adder port map (
            a => a_arr(i),
            b => b,
            s => s_arr(i)
        );
    end generate ADDGEN;
end rtl;
```

You will still need to manually assign to array signals if you use this
technique, but at least you do not have to type `entity work.adder port map`
four times. You can see how this would be convenient if you generalized it to
larger numbers of repeated entities.
