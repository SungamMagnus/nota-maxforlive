# Nóta for Max for Live

Real-time **monophonic audio to MIDI** for bass guitar and voice, as two Max for Live devices for Ableton Live. They work like a
MIDI effect: no sidechain chooser and no "MIDI From" routing, and the notes can be played as they are, harmonised with chords,
or decorated with a random flourish, in a scale you choose or Live's own.

The detection engine (a multi-scale NSDF pitch tracker with a note state machine) is plain C++, wrapped as the Max external
`nota~`. It is the same engine as the VST3 plug-in in [SungamMagnus/nota](https://github.com/SungamMagnus/nota); a copy lives in
`dsp/` so this repository builds on its own.

| Device | Type | Goes on | Does |
|---|---|---|---|
| **Nóta source** | Max Audio Effect | the audio track with your bass / voice | listens, runs the engine, passes the audio through, sends the MIDI by name |
| **Nóta destination** | Max MIDI Effect | the instrument track, **before the instrument** (any instrument, including Live's own) | receives that MIDI, **shows the note it is receiving**, optionally **accompanies it with chords**, and plays it; the track's own MIDI passes through |

Both devices have a **Channel Link** number (default 1). Live can't send MIDI from an audio device to another track, so
the notes travel by name (`nota-midi-<number>`): set the same number on both devices to pair them, and different
numbers for several pairs (for example one for bass and one for voice). Different numbers = the destination shows nothing.

All of Nota's settings are Live parameters on **Nóta source** (so they can be automated and mapped): gate (the line in the display),
response, release, retrigger, velocity (min / max / full level / range / auto), transpose, channel, range (auto/bass/voice),
pitch bend and MPE (on/off, bend range, pressure, slide).

### Transpose, MPE, Pitch Bend, Pressure, Slide

* **Transpose** is shown as a note: **C3 is no transposition**, the range is C0 to C6 (three octaves each way).
* **MPE** is one switch. On: every note gets its own MIDI channel with its own pitch bend, pressure and slide (for an MPE
  instrument; *MPE Bend* is the bend range the instrument is set to, 48 by default). Off: ordinary MIDI on one channel.
* **Pitch Bend** (MPE off) sends the pitch-bend wheel following the pitch drift of the note, for the whole channel. With
  MPE on every note is always bent on its own, so the switch does nothing there.
* **Pressure** and **Slide** (MPE on) are about the *sound*, not the pitch: Pressure (channel pressure) follows how loud the
  note is, Slide (CC74) follows how bright it is. They do not move the pitch.

### Input meter and gate display

The left half of Nóta source is drawn like Live's own Gate: a dB scale (+6 to -70), a scrolling trace of the incoming audio
(the peak of the mono mix Nota analyses, one column per 20 ms, about 3.5 seconds across), a vertical meter on the same
scale, and the **gate as a cyan line**.

**The cyan line is the Gate control.** Click or drag anywhere in the display to put it there (hold Shift for 0.1 dB
steps); its value is printed next to it. The bottom of the scale means "off" (-inf), the top of the Gate range is 0 dB.
It is still a Live parameter called *Gate*, so it can be automated and saved in presets, but there is no knob any more.

Level above the gate (what gets through to the note detector) is orange, level under it is dim, in the trace and in the
meter. If the input has a steady noise floor, Nota raises its own gate above your setting (up to 15 dB); that gate is the
**dashed cyan line**, and it is the one the colours follow.

### Velocity

Vel Min, Vel Max and Vel Range are numbers (click and drag, or type); Full Lvl is a dial.

Velocity is read from the loudness of each note's attack and spread linearly in dB: **Vel Max** is reached at the **Full
Lvl** and **Vel Min** is reached **Vel Range** dB below it (default 24 dB, about the dynamics of ordinary playing).
**Vel Auto** (on by default) makes Full Lvl follow your playing: the loudest recent attack is the top of the range and
softer notes scale below it, so a hot track and a quiet one both use the whole range. The reference forgets slowly
(0.5 dB per second) and never falls below the Full Lvl dial, so for a very quiet source lower Full Lvl. With Vel Auto off,
Full Lvl is a fixed input level. (Before this the scale started at the gate, 55 dB under full, so nearly every note on a
normal track came out at 100-127.)

## Scale (Nóta destination)

**Scale Mode** is a three-way selector (vertical tab) that decides what happens to the notes and where the scale comes from:

* **Chromatic** - free: the played note is left alone, whatever it is. The **Key** and **Scale** menus (Live's own scale
  names) still give the scale for chords and flourish.
* **Internal** - the played note is moved to the nearest note of the scale chosen in the **Key** and **Scale** menus (ties
  go down). Chords and flourish use the same scale.
* **Live** - the device follows **Live's global scale** (root note, intervals and name from the Live API; Live 12's Scale
  Mode). The **Key** and **Scale** menus *show* Live's scale and are greyed out, and the played note is moved to the
  nearest note of that scale. If Live scale mode is off the display says so; if Live reports nothing (Live 11) the
  menus are used. Switching to Chromatic or Internal keeps the scale the menus were showing, so you can start from Live's
  scale and change it.

## Accompany

*Mode: Through* just plays the notes. *Mode: Accompany* adds a chord to every note it receives, in the current scale:

* **Role** - *Smart*: a diatonic chord that contains the note, placed under it (prefers I, V, IV, vi, ii, iii, vii).
  *Root*: the note is the chord's root (use this for bass). *Top*: the note is the chord's top voice.
* **Chord** - triad, 7th or 9th, stacked in thirds *within the scale* (so a pentatonic scale gives pentatonic chords).
* **Melody** plays the note itself as well; **Open** drops the second voice from the top an octave; **Octave** (+-5)
  shifts the chord; **Chord Vel** is the chord's velocity as a percentage of the note's.
* Chord notes go on the note's channel, so with MPE on they share its expression. A note-off ends the whole chord.
* **Dynamic** - every note, the chord is played a little differently: 33% of the time a different chord type (triad,
  7th or 9th), 15% of the time the chord moves one octave up or down, 15% of the time a different role. The three
  chances are independent, so occasionally a note gets more than one change. It always picks a value other than the one
  set on the panel, never changes the panel, and the display shows what it did ("7th, oct +1, top"). It affects the
  chord only, not the note you play, and needs Accompany.
* **Humanize** - a human feel on everything the device adds, chord and flourish; the **Amount** dial next to it sets how
  much. At the default 25% it is subtle: velocity varies by up to about 8% (+-3), chord notes start up to 22 ms late and
  end up to 28 ms late, flourish times shift by up to 12 ms and lengths by 15%. Everything scales with the Amount, so
  100% is four times as loose (velocity about +-32%, chord notes up to ~90 ms late, ~110 ms late on release, flourish
  times +-48 ms, lengths +-60%) and 0% does nothing. The note you play is never touched, so the timing you play stays
  exact.

## Voice Count

**Voice Count** (a number under the Scale Mode selector, 1 to 32, default 16) is the most notes the device lets sound at
once, melody, chord and flourish together. When a new note would go over the limit:

* the played note always gets a voice, then the chord tones **nearest to it** (ties: the lower one); the rest of the chord
  is left out;
* older notes are ended to make room, oldest first;
* the **flourish** never takes a voice from a note: its notes are simply skipped while the voices are full.

Lowering the number ends the oldest notes straight away. A Voice Count of 1 makes the output monophonic (each new note
ends the previous one).

## Flourish

A separate switch, independent of Accompany (both can be on at once). After **every note**, a burst of random scale notes
plays out like a delay: it does **not** depend on the note being held, so it keeps going after you let go.

* **Delay** - how long after the note the flourish starts (0 - 5 s; 0 means almost at once, the first note is never
  earlier than 30 ms). **Time** - how long the notes keep being triggered: the window, starting after Delay, that they
  are spread over at random times (0.1 - 10 s). So with Delay 1 s and Time 2 s the notes fall between 1 s and 3 s after
  the note.
* **Notes** - how many random notes follow each note. **Decay** - how much quieter each successive note is.
  **Spread** - how far, in scale steps, a random note may be from the notes just played (the note, or the whole chord
  when Accompany is on). **Length** - how long each sounds. **Vel** - velocity as a percentage of the note's.
* Pitches come from the current scale (Live's when locked, the menus otherwise) and are never a note already sounding.
* Flourish only plays above C3 (MIDI 60). When the received note is higher than that, the flourish stays above the
  received note. Low bass notes therefore get a flourish an octave or more up, near the harmony, rather than a rumble
  next to the bass.
* A new note restarts the flourish: flourish notes from the previous note that have not played yet are dropped (ones
  already sounding finish), so with a long Delay or Time only the latest note's flourish is ever waiting.
* Switching Flourish off stops notes that have not played yet (sounding ones finish); *All off* stops everything.

The display on the left shows the received note, its velocity and channel, the notes actually sounding, and the key
and scale in use. Hover a control in Live (Info View) for a one-line explanation.

## Install

Get `Nota-for-Live.zip` (build it with `python3 make_share.py`, see below) and drag the two `.amxd` files into Live, or put
them in your User Library (*Presets > Audio Effects > Max Audio Effect*, and *MIDI Effects > Max MIDI Effect*). They are
**frozen**: `nota~` and the scripts are embedded in each file (what Max's *Freeze Device* does, done by `freeze.py`), so
nothing else has to be installed. Needs Live 11 or 12 with Max for Live, on a Mac (universal binary, Intel and Apple silicon).
The external is ad-hoc signed, not notarized: macOS may ask you to allow the devices once (`xattr -dr com.apple.quarantine`
on the files does it). Windows would need a separate build of the external. After installing a new version, quit Live
completely and reopen it: Live loads an external once per session.

## Build

Needs CMake 3.25+, Xcode command line tools and Python 3.

```bash
python3 make_share.py        # devices + nota~ + frozen devices + dist/Nota-for-Live.zip
```

or step by step:

```bash
python3 make_devices.py                                                    # devices/*.amxd and package/nota/javascript/nota_dest.js
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release && cmake --build build -j   # fetches max-sdk-base; output: externals/nota~.mxo
python3 freeze.py "devices/Nóta source.amxd" out.amxd --mxo externals/nota~.mxo \
    --text package/nota/javascript/nota_info.js --text package/nota/javascript/nota_scope.js
```

`make_devices.py` generates the `.amxd` files (the device UI is described in that script, in Python); the destination's note
logic is `nota_dest.template.js` (the generator injects the scale table). Fast stand-alone tests of the scripts, no Max needed:

```bash
NOTA_JS=package/nota/javascript/nota_dest.js    osascript -l JavaScript tests/test_dest.js     # chords, scales, flourish, voice count...
NOTA_SCOPE=package/nota/javascript/nota_scope.js osascript -l JavaScript tests/test_scope.js    # the input meter / gate display
```

## Layout of this repository

| | |
|---|---|
| `nota~/` | the Max external (wraps the engine; MIDI and display lists out, audio through) |
| `dsp/` | the engine, plain C++ (a copy of `source/dsp` in the VST repository) |
| `make_devices.py`, `nota_dest.template.js` | generate the two `.amxd` devices and the destination's script |
| `package/nota/javascript/` | the scripts the devices use (`nota_dest.js` is generated) |
| `freeze.py`, `make_share.py` | embed `nota~` and the scripts in the devices; build the zip |
| `devices/` | the generated, unfrozen `.amxd` files (they need the `nota` package; use the frozen ones from `make_share.py`) |
| `tests/` | the script tests |

## Limits

* Live only lets an *audio* device receive audio and a *MIDI* device send MIDI, so notes travel between the two
  devices by Max `send`/`receive`. That adds roughly one audio buffer (about 3-12 ms) on top of the engine's delay.
* The built `nota~.mxo` is a build output and is not kept in git.
