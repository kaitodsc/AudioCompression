# Yurei

This document represents my thoughts on exploring audio compression.

The idea is to compress each sample in a small number of bits.  Like the UTF-8 encoding, each
sample can take a different number of bits.  The bits will represent a diff of the previous sample.
It will also take in contextual information to generate the next bit.  For example, if the last
diff was a positive change, then the next diff will also default to a positive change. That's used on Discord AGC

One idea about this is to think about the samples as having velocity and acceleration.
Maybe a good way to compress is to have the bits represent acceleration of the difference between samples.

## Stored audio vs live streaming audio

There will be different techniques for different use cases.  If the audio is being delivered as a unit, then
it can start with a header that can have a table of common patterns.  This table will map certain bit patterns
to certain diff values.

If the audio is just a live stream, then such a table would not make alot of sense, unless you are delivering the
live audio in chunks.  Each chunk could have it's own mini table, maybe that's a good idea, maybe not, not sure.

## Lossiness

The format should support different levels of lossiness, including a lossless version.



# Idea for compressing a unit of audio

For a unit of data, instead of starting with a format, start by analyzing the audio.
Come up with all the commands it would take to generate the audio.  Then using
those commands, come up with a format for those commands that would be optimized.
The commands will be stored in the beginning of the file.

You could also extend this by grouping chunks together by how similar their commands are.
If a group has alot of commands that are not similar, then it may be better not to create
a custom command table.  The advantage will depend on how many samples there are, and
how similar they are. 

The less number of samples the less likely a command table is useful.
The less similar the samples, the less likely a command table is useful.

The command table can also be optimized.  The more a certain command is used, the less
bits it should take.  If they are all used equally then each command could use the same
number of bits, i.e. If you only have 4 commands and they are all used equally, then you
could just map each command to 2 bits, 00, 01, 10 and 11.  If 1 command is very heavily used,
then maybe a table like this is better:
```
command 1: 0
command 2: 100
command 3: 101
command 4: 110
```

# Working through examples
```
Samples:  02  05  07  06  06  04  02  02  05  06  01 -05 -09 -06
Diff   :    03  02 -01  00 -02 -02  00  03  01 -05 -06 -04  03

Initial Velocity: Always 0
Commands:
  - Set value
  - Set velocity
  - MaintainVelocity (next sample will match velocity)
  - ChangeVelocity (set or add or multiple or shift or divide, etc)

This example(lossless compression):

Command: SetValue(2)              (Velocity=0)
         Value=2 (output sample)
Command: ChangeVelocity(3)        (Velocity=3)
         Value=5 (output sample)
Command: ChangeVelocity(-1)       (Velocity=2)
         Value=7 (output sample)
Command: ChangeVelocity(-3)       (Velocity=-1)
         Value=6 (output sample)
Command: ChangeVelocity(1)        (Velocity=0)
         Value=6 (output sample)

```

### Adding in acceleration:
Since an audio stample is rarely a straight line, it makes since that by default we
should assume that the audio sample velocity is probably going to change quite often.

If we assume the change in velocity is going to resemble a sin wave, then we can
guess that the acceleration is going to resemble this:
```
                           *     *
                      *               *
                  *                       *
               *                             *
             *                                 *
            *                                   *
-----------------------------------------------------------
           |         |         |       |
  no acceleration    |         |       |
                     |         |       |
             most deceleration |   most deceleration
                               |
                               |
                        no acceleration
```

This acceleration can actually be calculated.
The accleration (or slope) of a function is defined by the "deriviate" of
a function, and the deriviate of `sin(x)` is `cos(x)`.

# General Summary of Thoughts

It seems this compression technique is really made up of a set of commands, and
techniques to compress those commands.  So the compression really comes in two stages,
the first stage being finding a command scheme to generate the audio, and the second
stage being compressing those commands.

One set of commands treats the audio signal as a moving object where each command
can change the acceleration and outputs a sample based on the current velocity.
However, there may be more useful techniques to command the audio signal (FFT for example).

Once a good set of commands are found to generate the audio, then comes second stage of
compressing the commands.  Depending on how many commands there are, and how often they
are reused will determine the best strategy to compress the commands.

