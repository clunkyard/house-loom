# Glyph language specification

Glyph is a concatenative language. Numbers, strings, sensors, and memory reads
push values onto a stack. Words consume those values, produce new values, or
write Arena controls. Named words let a tactic be assembled from small pieces
without an expression parser or host-language adapter.

This works well for data-flow tactics. A targeting word can duplicate one
bearing for both the turret and scanner; a roaming word can update a
persistent phase and turn its sine into steering.

## First bot

```glyph
bot First Glyph
modes 2d 3d

: target
  "TRACK" status-text!
  range status-value!
  bearing dup aim! scan-heading!
  elevation dup aim-pitch! scan-pitch!
  cooldown 0 = aim-error abs 3 < and
  battery shot-cost-base shot-cost-per-power + 5 + >= and fire!
;

: roam
  "SEARCH" status-text!
  tick status-value!
  0.7 drive!
  0.25 turn!
  heading tick 8 * + scan-heading!
  mode3 if tick 30 / sin 22 * scan-pitch! then
  58 scan-arc!
;

contact if target else roam then
```

The example is [minimal.glyph](minimal.glyph). Run it from the repository:

```sh
node src/cli.js match --mode 2d --ticks 300 --seed 4301 \
  --bot 'First Glyph=node src/dsl/runner.js glyph bots/dsl/glyph/minimal.glyph' \
  --bot dsl-cadence
```

Add `--serve` to watch. Register a new source for browser setup with the
manifest shown in the [DSL guide](../../../docs/dsls.md#register-your-source).

This bot fires at power 1 once the turret, which turns at 220° per second, is
within 3° of the target and the battery can pay for the shot with 5 to spare.
The new-robot starter,
[src/public/starters/glyph/bot.glyph](../../../src/public/starters/glyph/bot.glyph),
also picks shot power by range and keeps a larger reserve for driving.

## Reading stack code

Glyph uses postfix order. The word comes after its inputs:

```glyph
bearing heading angle 72 / 0.3 + clamp1 turn!
```

Read that from left to right:

1. Push the target bearing and current heading.
2. `angle` replaces them with the shortest signed difference.
3. Divide by 72, add 0.3, and clamp to -1 through 1.
4. `turn!` removes the result and writes the turn request.

The data stack starts empty on every decision. Declared memory persists across
decisions. Arena actions also start fresh each tick; if code writes the same
action more than once, the last write wins.

[loom.glyph](loom.glyph) builds `sweep`, `engage`, and `roam` words around a
persistent weaving phase. [orbital-coast.glyph](orbital-coast.glyph) uses a
short thrust burn and then leaves thrust at zero in the zero-gravity ruleset.

[filtered-orbit.glyph](filtered-orbit.glyph) composes heading selection, an
angle controller and a persistent bounded steering filter. Its words pass
values to each other before action setters consume them. The
[workshop](../../../docs/dsl-workshop.md#glyph-compose-a-controller-from-values)
explains their stack effects and runs the robot in a real local match.

## File structure

Header directives occupy their own lines. Remaining tokens form named word
definitions and the main program; line breaks there are only whitespace.

```text
bot DISPLAY NAME
modes 2d 3d
rulesets classic inertial-2d gravity-flat gravity-hills zero-g
memory NAME NUMBER

: WORD TOKEN ... ;

TOKEN ...
```

- `bot`, `modes`, and `rulesets` describe the program to the runner.
- `memory` declares a persistent cell with a numeric initial value.
- `: name ... ;` defines a word. Forward references are allowed.
- Code outside definitions is the main program and runs every tick.
- A program needs executable main code, even if it defines words.
- `#` starts a comment outside a string.

Words and memory names are case-insensitive. A source may contain at most 4096
code tokens and 128 named words.

## Literals, memory, and flow

Numbers and quoted strings push themselves. `true` and `false` push booleans.
Strings support `\n`, `\r`, and `\t` escapes.

```glyph
memory phase 0
phase@ 5 + dup phase! status-value!
```

`phase@` reads a declared cell. `phase!` pops and stores one value. Numeric
stores must remain finite.

`if` pops a condition. Zero, an empty string, and false are false-like; other
values are true-like.

```text
CONDITION if YES-WORDS then
CONDITION if YES-WORDS else NO-WORDS then
```

Definitions may call other definitions, including themselves, but call depth
and per-turn work limits bound recursion.

## Stack words

Stack effects use `( before -- after )`; the rightmost item is the top.

| Word | Stack effect | Meaning |
| --- | --- | --- |
| `dup` | `( a -- a a )` | duplicate the top value |
| `drop` | `( a -- )` | discard the top value |
| `swap` | `( a b -- b a )` | exchange the top two |
| `over` | `( a b -- a b a )` | copy the second value |
| `+ - * / %` | `( a b -- result )` | arithmetic in operand order |
| `neg`, `abs` | `( a -- result )` | numeric unary operations |
| `min`, `max` | `( a b -- result )` | numeric selection |
| `clamp` | `( value low high -- result )` | limit to an interval |
| `clamp1` | `( value -- result )` | limit to -1 through 1 |
| `sin`, `cos` | `( degrees -- result )` | trigonometry in degrees |
| `angle` | `( target current -- difference )` | shortest signed angle |
| `= == != < <= > >=` | `( a b -- boolean )` | comparison |
| `and`, `or` | `( a b -- boolean )` | boolean combination |
| `not` | `( a -- boolean )` | boolean negation |

`/` and `%` reject a zero divisor. Glyph uses `neg` for numeric negation; `-`
always consumes two numbers.

## Sensor words

Each sensor word pushes one value from the bot's private observation.

| Area | Words |
| --- | --- |
| Match | `tick`, `time`, `remaining`, `mode3`, `ruleset` |
| Own robot | `health`, `cooldown`, `x`, `y`, `z`, `heading`, `pitch`, `aim`, `aim-pitch` |
| Battery | `battery`, `battery-capacity`, `battery-fraction` |
| Scanner | `scan-heading`, `scan-pitch`, `scan-arc`, `scan-range`, `contact`, `range`, `bearing`, `elevation`, `aim-error` |
| Shots | `power-min`, `power-max`, `shot-speed-base`, `shot-speed-per-power`, `shot-cost-base`, `shot-cost-per-power` |
| Safe zone | `zone-active`, `zone-radius`, `zone-distance`, `zone-bearing`, `zone-elevation` |
| Match mechanics | `storm-active`, `storm-distance`, `storm-bearing`, `storm-heading`, `storm-radius`, `cue-present`, `cue-bearing`, `cue-elevation`, `cue-range`, `cue-error`, `cue-age`, `cue-muzzle`, `pickup-present`, `pickup-bearing`, `pickup-elevation`, `pickup-range`, `hill-active`, `on-hill`, `hill-held`, `hill-bearing`, `hill-elevation`, `hill-range`, `hill-radius`, `hill-time`, `hill-to-win`, `score`, `score-best` |
| Arena | `width`, `height`, `depth`, `terrain-kind`, `control-model` |
| Physics flags | `physics-mode`, `inertial`, `gravity`, `zero-g`, `hills`, `grounded` |
| Motion | `vx`, `vy`, `vz`, `speed`, `yaw-rate`, `pitch-rate`, `roll-rate` |
| Ground and gravity | `altitude`, `terrain-height`, `gravity-x`, `gravity-y`, `gravity-z` |
| Attitude basis | `forward-x/y/z`, `right-x/y/z`, `up-x/y/z` |

`contact` indicates a current scanner echo. `range`, `bearing`, and
`elevation` describe only the nearest echo. Without contact, range is very
large, bearing is the robot heading, and elevation is zero. `aim-error` is the
signed heading difference from where the turret points now to that echo, and 0
without one; in 3D, also compare `elevation` with `aim-pitch`.

`cooldown` is 0 when the gun can fire this tick. `aim` and `aim-pitch` report
where the turret points now. `scan-range` is the reach of the current scan
arc. A shot costs `shot-cost-base + shot-cost-per-power * power` battery and flies at
`shot-speed-base + shot-speed-per-power * power`. Before sudden death,
`zone-active` is false, the zone distance and radius are 0, and the zone
bearing is the robot heading. In classic, `speed` is the smoothed speed that
aim spread uses. The [DSL
guide](../../../docs/dsls.md#shared-observations-and-actions) lists every
value, and the [protocol guide](../../../docs/protocol.md) defines each one.

The match mechanics' values read as inactive while their mechanic is off or
not yet in play: flags false, ranges and distances a very large number,
bearings the robot heading. `storm-distance` is to the storm's edge and
negative inside it. `cue-*` describe the best passive cue, remembered for 60
ticks where it placed a robot and seen from where the robot is now. The [DSL
guide](../../../docs/dsls.md#shared-observations-and-actions) defines each
one.

## Action words

Action words pop one value.

| Word | Request |
| --- | --- |
| `drive!`, `turn!`, `pitch-turn!` | Classic forward, yaw, and pitch controls |
| `thrust!`, `strafe!`, `lift!` | Physics linear forces |
| `yaw-torque!`, `pitch-torque!`, `roll-torque!` | Physics angular forces |
| `brake!` | physics braking assist, 0 to 1 |
| `aim!`, `aim-pitch!` | absolute bearing and elevation the turret turns toward |
| `scan-heading!`, `scan-pitch!`, `scan-arc!` | scanner direction and cone width |
| `fire!` | fire this tick if `cooldown` is 0 and the battery can pay; any true-like value requests it |
| `power!` | shot power, 0.5 to 3 |
| `status-text!`, `status-value!` | spectator text and finite numeric value |

Numeric action values must be finite; the engine applies legal control limits.
A tick that writes no `power!` fires at power 1. `brake!` takes a strength: a
number is clamped to 0 through 1, and a boolean brakes fully when true, so
`speed 70 > brake!` and `0.3 brake!` both work. The scanner turns at once; the
turret turns at up to 220° per second. A shot the battery cannot pay for is
refused at no cost. Classic motion words remain available in physics matches
as force aliases.

## Work limits and errors

Without a supplied budget, the local Glyph interpreter allows 1024 operations
per decision, with 128 stack values and 32 nested word calls. Hosted DSL
decisions and browser practice also use 1024 operations. The local CLI with
`--resources dsl` defaults to 768; add `--dsl-ops 1024` to use the hosted
operation limit. A supplied `dslOpsPerTurn` replaces the operation default.
Each executed literal, word, call and conditional consumes host-metered work.

Compile and runtime errors include `file:line:column`. Unknown words, stack
underflow, division by zero, a non-finite result, excessive stack or call
depth, and an exhausted operation budget forfeit the bot without stopping the
match.

## Compile to WASM

The build needs Clang, a C compiler and the Wasmtime C API; see the
[WebAssembly guide](../../../docs/sdk/wasm.md). Run from the repository root:

```sh
scripts/build-dsl-wasm.sh glyph
cat > build/glyph-wasm.json <<'JSON'
{
  "name": "Wasm Glyph",
  "language": "glyph-wasm",
  "protocol": 2,
  "supportedModes": ["2d", "3d"],
  "resourceModes": ["wasm"],
  "command": ["build/clunkyard-wasm-host", "build/wasm-loom-glyph.wasm"]
}
JSON
node src/cli.js match --resources wasm \
  --bot @build/glyph-wasm.json --bot c --ticks 300
```

Named words become WASM functions. The operand stack and declared memory live
inside the module, and traps handle stack underflow, overflow and zero
division. The interpreter remains available as `dsl-glyph`. The module speaks
WebAssembly ABI 4: it reads every value through the robot core's `dsl_fill_v1`
import once a turn and sets each control through the core's setters. Numeric
`status-value!` works in the module. Text `status-text!` is a compile error
with a source location; use the interpreter when a program needs text.

## Grammar

```text
program      := directive* definition* main
directive    := "bot" words
              | "modes" ("2d" | "3d")+
              | "rulesets" ruleset+
              | "memory" word number
definition   := ":" word sequence ";"
main         := sequence
sequence     := (literal | sensor | stack-word | action-word
              | memory-read | memory-write | word-call | conditional)+
conditional  := "if" sequence ("else" sequence)? "then"
memory-read  := declared-name "@"
memory-write := declared-name "!"
ruleset      := "classic" | "inertial-2d" | "gravity-flat"
              | "gravity-hills" | "zero-g"
```
