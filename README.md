# cass-edi
## UCSD Center for Astrophysics and Space Sciences - Electron Drift Instrument - Test Harness - Forth

This is the code to control the hardware test harness to validate the Electron Drift Instrument, which was later launched on the Space Shuttle.
It is a significant program written in a 32-bit variant of LMI FORTH using a DOS Extender that does everything from low-level hardware control to complex data analysis to graphical display and keyboard-driven user interaction.

The test harness was composed of the following.

1. 14 electron guns controlled by Burr-Brown DACs.
2. A stepper motor to position the instrument.
3. The instrument, sometimes referred to as the `MARKIII DETECTOR` in comments.
4. A 256x256 Channel Plate (64K pixels) device to record output. This is the same technology used in later generation night vision devices. It was *very* expensive hardware at the time. It came with some manuals and nothing else.
5. A 16MHz 80386 PC with 8MB RAM

There was code to:

1. Modulate the output of the electron guns.
2. Move the motor using a PID algorithm to position the instrument.
3. Record the output of the guns onto the Channel Plate as they exited the device.
4. Perform detailed analysis of the output.
5. Define and execute test plans, iterating over them, and logging results.
6. A GUI to present all this graphically and provide keyboard-driven user interaction.

I was told at the time that because of my work I would be named as one of the contributors to the article published as a result of the project. Being that I left years before that happened, my name was sadly omitted.

The initial file to load is `OPTICS26.SCR`, which loads all other files. From what I remember, the `26` referred to the 26 screens of source code when I took over development. The number was not interated further.
By the time I left, a little over a year later, there were over 250 screens, all of which I wrote.
Some screens mention other names, which included the professor overseeing the project, my boss, and a former member of the team. Still, I wrote 90% of the code. The rest was written by my boss.

Each screen in Forth is 1024 bytes, formatted as 64x16 in text. Each screen included a header line as a comment.
I reformatted the raw screens to be compatible with regular text editors using this Perl one-liner applied to each file:

`perl -pe 's/(.{64})/$1\n/g' OPTICS26.SCR > optics26.scr`

Then the trailing spaces of each line were stripped. The code otherwise is as it was when I left.

Here is the order of loading, along with brief descriptions of the contents of each, as run from `optics26.scr`:

```
INCLUDE SYSVAR26.SCR  \ INIT VARS,SYS DEFS ETC/FORMERLY INIT
INCLUDE ROUTIN26.SCR  \ COMMONLY USED LOW-LEVEL ROUTINES
INCLUDE TRODES26.SCR  \ MANAGE,UPDATE,DSPLAY AMPILFIER VOLTGES
INCLUDE OUTPUT26.SCR  \ BURR-BROWN D/A OUTPUT
INCLUDE LODSAV26.SCR  \ LOW-LEVEL AND CASE FILE I/O
INCLUDE PIDCTL26.SCR  \ PID POLAR MOTOR CONTROL
INCLUDE SHIFTR26.SCR  \ COMPENSATED SHIFT ROUTINES
INCLUDE RASTER26.SCR  \ RASTER SCAN ROUTINES
INCLUDE CCPCTL26.SCR  \ CCP DMA ACQUISITION/FORMERLY MCADMA
INCLUDE HISTOG26.SCR  \ CHANNEL PLATE HISTOGRAMS
INCLUDE CONVLV26.SCR  \ HISTOGRAM CONVOLUTIONS
INCLUDE AREAXX26.SCR  \ OBTAIN EFFECTIVE AREA OF A CASE
INCLUDE AUTOXX26.SCR  \ AUTOMATICALLY RUN SEARCHES ETC
INCLUDE SEARCH26.SCR  \ OPTIMIZED MULTI-MODE SEARCHES
INCLUDE USERIN26.SCR  \ IMPLEMENTS MAIN USER INPUT ROUTINE
```

Below are various comments excerpted from the code that I wrote at the time describing some of the complex functionality of the test harness and data capture and analysis being done.

There are various references to the [IBM PC AT Technical Reference Manual](https://minuszerodegrees.net/manuals/IBM_5170_Technical_Reference_6280070_SEP85.pdf)
and Lewis Eggebrecht's [Interfacing to the IBM Personal Computer](https://www.amazon.com/Interfacing-Personal-Computer-Lewis-Eggebrecht/dp/067222027X).
Even though the latter only covered 8-bit computers, it was essential reading for details lacking in the official 16-bit documentation. As I described in my Amazon review:

```
The old 8088 systems even then were long in the tooth. Yet, this book was absolutely indispensable when writing 32 bit DMA device drivers. There were legacy quirks that only this book explained correctly. If you are responsible for code written for the ISA architecture, regardless of CPU generation, this book was and is a must.
```

### PID for stepper motor:

```
A FLOATING POINT IMPLEMENTATION OF A MODIFIED
PID ALGORITHM FOR MOTOR CONTROL.  USED TO CONTROL
ROTATION OF THE MARKIII DETECTOR ABOUT A POLAR AXIS
```

### 16-bit DMA transfers and moving data from 16-bit real to 32-bit protected mode memory:

```
\ THIS ROUTINE USES AT DMA CH#7, ALSO CALLED 16 BIT CH#3 IN THE
\ AT MANUAL.  ALL I/O PORTS ARE CODED FOR THIS CHANNEL
\ THE BASE ADDRESS FORMAT IS IN THE AT TECH. REF. MANUAL AND IS
\ SLIGHTLY UNUSUAL-IT MUST BE SPLIT BET PAGE & DMA REGISTERS.

\ DMA TRANSFER FOR THE CARLSON SYSTEM WAS DIFFICULT TO FIGURE
\ OUT DUE TO THE DISPARATE NATURE OF THE DOCUMENTATION.  THE
\ PERTINENT REFERENCES ARE THE 2415 AND 2420 MANUALS (BUFFER
\ AND PC INTERFACE) AND HOW THEY ARE CONTROLLED AND USED IN
\ CONJUNCTION WITH DMA.  ALSO, THE EGGBRECHT BOOK ON PC INTER-
\ FACING IS A MUST, BUT MUST BE USED IN CONJUNCTION WITH THE
\ THE AT TECH. REF. MANUAL SINCE THIS IS AN AT COMPATIBLE AND
\ EGGBRECHT IS OSTENSIBLY FOR PC'S. THE INFORMATION PRESENTED
\ ON DMA IS APPLICABLE HERE TOO WITH THE CHANGES NOTED IN THE
\ AT MANUAL.  STUDY WHAT IS DONE CAREFULLY BEFORE CHANGES,
\ SINCE THE DMA ROUTINES WORK AND ARE STABLE!
```

### Test protocols:

```
\ THIS FILE HAS SUPPORT FOR MULTI-VARIABLE AUTOMATED BLIND
\ TESTS FOR VOLTAGE VALUES.  ONE TEST CAN BE PERFORMED AT A
\ TIME ITERATING THROUGH UP TO 14 DIMENSIONS (CHANNELS).  THE
\ RESULTS ARE THEN OUTPUT TO A FILE AND OPTIONALLY TO THE
\ PRINTER.  THE PARAMETERS ARE SPECIFIED THROUGH 4 PARAMETERS,
\ NAMELY MIDPOINT, +/-RANGE, INCREMENT, AND CHANNEL.  THIS
\ ALLOWS THE CHANNELS TO CHANGED IN ANY ORDER, AND PERMITS
\ INTUITIVE SPECIFICATIONS.  ONE EXCEPTION IS THAT ANGLE
\ ELEVATION MUST BE THE OUTER LOOP SINCE IT IS TREATED
\ DIFFERENTLY THAN OTHER CHANNELS.  WHEN FINISHED WITH THE TEST
\ THE PROGRAM RETURNS TO THE SUBMENU.
\ THE CORE OF THE ROUTINE IS THE AUTO-LOOP CONTRSUCT, WHICH
\ HAS AN OUTER GENERAL PART THAT CAN HANDLE ANGLE LOOPS, AND
\ THE MORE SPECIALIZED INNER-LOOPS THAT CAN ONLY LOOP OVER
\ ELECTRODES.
```

### Analytics:

```
\ THIS FILE HAS ROUTINES TO PERFORM HISTOGRAMS ON DETECTOR
\ DATA.  DATA IS COLLECTED FROM THE CARLSON CHANNEL PLATE
\ SYSTEM ONTO A 256X256 IMAGE (SEE CCPCTL).  FOUR ARCS ARE
\ PLACED TRHOUGH THE IMAGE TO GET THREE ANNULI: INNER, CENTER,
\ AND OUTER.  THE CENTER ONE IS BROKEN UP INTO 72 QUATER-
\ SECTOR BINS, AND THEN HISTOGRAM FINDS THE NUMBER OF COUNTS
\ PER BIN.  FOR INNER AND OUTER, ONLY THE TOTAL COUNTS PER
\ ANNULUS IS TAKEN AND REPORTED.  THE QUARTER SECTOR BINS
\ ARE THEN COMBINED INTO 1,2,4,8 SECTORS (1 SECTOR=4 BINS)
\ AND THE SECTOR COUNTS ARE THEN REPORTED.  THE SECTORS ARE
\ TAKEN FROM ARE CALCULTED FROM A CENTER BIN, WHICH IS DEFAULT
\ IN THE CENTER OF THE 72 BINS, BUT CAN BE PLACED ANYWHERE
\ (EG AT THE PEAK COUNT BIN)
\ THE HISTOGRAM ARE OPTIMIZED FOR FAST EXECUTION AND TAKE UP
\ A LOT OF MEMORY (SEVERAL HUNDRED KBYTES) (CON'T -->)

\ UPON STARTUP OR CHANGE OF ANNULI VALS, A ROUTINE RUNS THROUGH
\ AND FINDS OUT WHICH MEMORY LOCATION CORRESPOND TO WHICH BINS
\ IN AN IMAGE BY GOING THROUGH AND FILLING EACH BIN WITH ITS
\ RESPECTIVE NUMBER (0 THRU 71).  THEN A ROUTINE GOES THROUGH
\ AND SEARCHES THE IMAGE FOR VALS 0-71, AND EACH TIME IT
\ ENCOUNTERS SUCH, IT TAKES THE LOCATION OF THAT VALUE AND
\ PLACES IT IN A BIN LOCATION ARRAY.  THE ACTUAL HISTOGRAM
\ THEN SEQUENTIALLY GOES THROUGH THIS ARRAY AND SUMS THE VALS
\ IN THE MEMORY LOCATIONS.  THIS SPEEDS UP CALCULATION BY
\ AVOIDING TRIG CALCS FOR THE ANNULUS AT RUN TIME.  ANYTIME
\ THE ANNULI RADII OR BIN WIDTH IS CHANGED, A RECALC OF
\ ADDRESSES MUST BE DONE.  LIMITATIONS INCLUDE NO MORE THAN
\ 256 LOCATIONS PER BIN, AND NO ERROR CHECKING.  THIS LIMIT
\ WILL NOT NORMALLY BE REACHED.

\ BIN-COUNT TAKES 4 VALS DEFINING A REGION OF AN ANNULUS
\ AND FINDS ALL THE HITS THEREIN.  IT HAS AN OUTER LOOP WHICH
\ IS OVER THE ANGLE AND AN INNER LOOP OVER THE RADIUS.  THE
\ INCREMENT FOR THE RADIUS IS CONSTANT, BUT CHANGES FOR THE
\ ANGLE AS A FUNCTION OF CHANGING RADIUS.  THE ROUTINE
\ CALC-INCREMENT CALCS THE SMALLEST INCREMENT ENCOUNTERED,WHICH
\ IS AT THE OUTER RADIUS, AND USES THAT THROUGHOUT, SINCE NO
\ HARM IS DONE, AND FIND THE EXACT INCREMENT FOR EACH RADIUS
\ TAKES MORE TIME.  IN ORDER TO ASSURE THAT NUMBERS ARE NOT
\ COUNTED MORE THAN ONCE, DUE TO TOO SMALL INCREMENT, THE READ
\ IS DESTRUCTIVE, REPLACING THE VALUE WITH 0.

\ HISTOGRAM IS THE SHELL FOR BIN-COUNT. IT TAKES RIN, ROUT
\ WHICH REMAIN CONST, THETA-RANGE WHICH IS THE +/- RANGE
\ ABOUT THE 0-ANGLE, AND THE BIN-WIDTH. IT THEN INCREMENTS
\ OVER THETA, PASSING THE VALUES FOR EACH REGION TO BIN-COUNT

\ BIN-GEN AND HISTO-GEN ARE ANALOGOUS TO BIN-COUNT AND
\ HISTOGRAM WORDS.  BIN-GEN TAKES A REGION AND FILLS IT WITH
\ THE CURRENT BIN# USING A SIMILAR, BUT SLIGHTLY SIMPLIFIED,
\ BIN-COUNT ALGORITHM. BIN-GEN LOOPS BACKWARDS FOR THE SAME
\ REASON HISTO-GEN DOES, AND THUS MEANS NOTHING.
\ HISTOGEN FIRST SETS ALL PIXELS TO -1, SINCE THE SMALLEST BIN
\ NUMBER IS 0, AND SIMPLIFIES SEARCHING FOR ALL BIN LOCATIONS.
\ IT LOOPS BACKWARDS BECAUSE OF INITIAL WORRIES ABOUT BUGS IN
\ THE SOFTWARE, AND LOOPS DOWN TO -1 INSTEAD OF 0 TO MAKE SURE
\ THAT ALL BINS ARE THE SAME SIZE, SINCE THE EDGES OF THE BINS
\ OVERLAP, ONE END BIN WOULD BE LARGER OTHERWISE.
\ ADDR-GEN SEARCHES FOR BIN#'S IN IMAGE, GETS THE BIN# AND
\ ADDR OF LOCATION, THEN STORES THIS IN THE NEXT AVAILABLE LOC.
\ IN THE ADDR ARRAY.  Q-SEC[] IS USED TO STORE NEXT INDEX FOR
\ EACH BIN, BEING INCREMENTED AFTER STORING A NEW LOCATION.

\ THIS MODULE HAS ROUTINES TO PERFORM CONVOLUTIONS ON HISTOGRAM
\ DATA.  THE CONVOLUTION ROUTINES ARE GENERALIZED AND CAN USE
\ ANY LENGTH OF QUARTER-SECTOR BINS FOR THE KERNEL.  THIS
\ MEANS THAT STANDARD ANODE CONFIGURATIONS (1,2,4,8) CAN BE
\ USED, AS WELL AS ANY AMOUNT OF QUARTER-SECTORS UP TO 32, FOR
\ ODD SIZED KERNELS.

\ FWHM STANDS FOR FULL WIDTH AT HALF MAX AND IS A 3 PART
\ ROUTINE.  FIRST, IT FINDS THE PEAK VALUE IN THE CONVOLUTION
\ AND THE INDEX OF THAT VALUE.  THEN, IT LOOP FROM 0 TO THE
\ PEAK INDEX AND LOOKS FOR THE POINT THAT IS JUST GREATER THAN
\ HALF THE PEAK.  THEN IT LOOPS FROM THE LAST VALUE (WHICH IS
\ 72+KERNEL_LENGTH) TO THE PEAK (INCREMENT -1) AND LOOKS FOR
\ THE POINT THAT IS JUST _LESS_ THAN HALF THE PEAK.  THEN IT
\ TAKES ABS(LEFT-RIGHT) AND MULTS. BY BIN-WIDTH TO GET THE
\ AZIMUTHAL ACCEPTANCE WIDTH
```
