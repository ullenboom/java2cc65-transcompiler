This is a very simple transcompiler (source-to-compiler) that converts a super small subset of the Java syntax to C. It was written to emit C code for the CC65 compiler so that developers can write Java code to program for the C64 home computer.

## What is it all about?

Developers write code because they either do it for money, they have to solve a problem or they are bored. I was bored so I was playing with the JDK compiler and `com.sun.source.tree` types and also with the simple C compiler CC65 that produces code for home computers, like the C64. This gives rise to the idea to bring both together.

Java is a language with the precise definition of types (like length of data types) and behaviour (like `ArrayIndexOutOfBoundException` when you run over an array). C is almost the opposite; different length of build-in data types and weird behaviour when a program runs out of an array. Instead of bringing the Java semantic to a corresponding C program (see JCGO), this transcompiler simply converts Java programs almost directly to C. So you can see this more as an alternative syntax to C. Thus, the semantic (`int` is always 4 byte e.g.) is not preserved and it’s up to the C compiler which size the data type will be. However, if its compiled by the CC65 compiler the size of an `int`, `char`, … is known to the developers.

## Supported and unsupported features

The Java syntax is similar to the C syntax and converting is pretty easy. But you have to help the compiler and develop “C-like”. An example: In Java it’s natural to declare variables in the smallest scope where in C you have to declare at the top of a method. This transcompiler doesn’t rearrange the declarations and it’s also not possible to declare variables in for-loops. The current version also doesn’t produce prototypes for methods.

From the Java syntax the compiler supports:

  * the data types `char`, `int`, `short`, and `long`, but not float data types. With the help of two annotations `@Unsigned` and `@Signed` it’s possible to declare `@Unsigned int`, or `@Signed char`. `byte` is mapped to `signed char`.
  * usual arithmetic, binary, logical operators, but not `>>>`
  * usual control structures

You can **not** use:

  * object-oriented “stuff”: no `new`, classes, no enums, for-each, exceptions, String concatenations, …
  * the Java standard library
  * Unicode literals. String literals can be used, but only from the range of 0x0000 - 0x00FF (256) and anyway, PETSCII is from 0 – 191 only
  * overloaded method
  * There is no `public static void main(String[] args)` method but you have provide a `int main()` method which is converted to `int main(void)` as a valid entrance function.

A subset of the Java syntax is used and its not the aim or the transcompiler to accept ans translate any construct.

## How to use

J2C is written in pure Java and only converts from Java to C. You need the CC65 compiler and linker suite to produce assembly or linked binary code.

Use the transcompiler like this:

```
set JAVA_HOME="C:\Program Files\Java\jdk1.7.0"
%JAVA_HOME%\bin\java -cp bin/;%JAVA_HOME%\lib\tools.jar tutego.j2c.J2C -o app.c App.java
```

The option `-o` specifies the output file and after the converting you get a C-version of `App.java` in `app.c`. Now you can compile the C file to assembler code or to machine code, e.g. in PRG format for an emulator:

```
cl65 -o app.prg app.c
"C:\Program Files\WinVICE-2.2-x64\x64.exe" app.prg
```

The last call opens the VICE C64 emulator and runs your program.

## Example

Simple example:

Adapted from a C course http://skoe.de/wiki/doku.php?id=ckurs:04-abend4:

```
import static tutego.j2c.include.Stdio.printf;

public class Application
{
  public static char istPrimzahl( int n )
  {
    int divisor;
    int testEnde = n / 2;

    for ( divisor = 3; divisor < testEnde; divisor += 2 ) {
      if ( n % divisor == 0 ) {
        return 0;
      }
    }

    return 1;
  }

  int main()
  {
    int zahl;

    for ( zahl = 3; zahl < 1000; zahl += 2 ) {
      if ( istPrimzahl( zahl ) != 0 ) {
        printf( "Primzahl: %u\n", zahl );
      }
    }

    return 0;
  }
}
```

http://www.tutego.de/blog/javainsel/images/Mit-Java-fr-C64-entwickeln_20A0/image.png

More advanced example with usage of some Header files:

```
import static tutego.j2c.include.C64.*;
import static tutego.j2c.include.Conio.*;
import static tutego.j2c.include.PeekPoke.*;

// http://skoe.de/wiki/doku.php?id=ckurs:06-abend6
public class Application2
{
  static char PITCH_TOP_Y = 4;
  static char PITCH_LEFT_X = 0;
  static char PITCH_RIGHT_X = 19;
  static char GRAVITY = 2;

  static int ballX;
  static int ballY;
  static int speedX;
  static int speedY;

  static void prepareScreen()
  {
    char x, y;

    bordercolor( COLOR_YELLOW );
    bgcolor( COLOR_PURPLE );
    clrscr();
    textcolor( COLOR_YELLOW );

    // Zeichne das Dreieck unten links
    for ( x = PITCH_LEFT_X; x <= PITCH_RIGHT_X; ++x ) {
      y = (char) (PITCH_TOP_Y + x + 1);
      revers( (char) 0 );
      cputcxy( x, y++, (char) 0xbf );

      revers( (char) 1 );
      while ( y < 25 ) {
        cputcxy( x, y++, ' ' );
      }
    }
    revers( (char) 0 );
    textcolor( COLOR_BLACK );
  }

  static void initBall()
  {
    ballX = 3 * 256;
    ballY = 0 * 256;
    speedX = 0;
    speedY = 0;
  }

  static void waitNextFrame()
  {
    while ( PEEK( 0xd012 ) != (char) 255 ) ;
  }

  int main()
  {
    int limit;
    int speedTmp;

    prepareScreen();
    initBall();

    do {
      waitNextFrame();

      // alte Ballposition loeschen
      cputcxy( (char) (ballX / 256), (char) (ballY / 256), ' ' );

      // Gravitation, aber v auf Maximalwert begrenzen
      if ( speedY < 100 )
        speedY += GRAVITY;

      // Ball bewegen
      ballX += speedX;
      ballY += speedY;

      if ( ballX < 0 ) {
        // dann abprallen lassen
        speedX = -(speedX - speedX / 8);
        ballX = 0;
      }
      // sonst: Ist der Ball im Bereich der schraegen Flaeche?
      else if ( ballX / 256 <= PITCH_RIGHT_X ) {
        // ist er auf oder unterhalb der schraegen Flaeche?
        limit = ballX + PITCH_TOP_Y * 256;
        if ( ballY >= limit ) {
          // dann abprallen lassen
          speedTmp = speedX;
          speedX = (speedY - speedY / 8);
          speedY = (speedTmp - speedTmp / 8);
          ballY = limit;
        }
      }
      // sonst: Ist der Ball am rechten Rand?
      else if ( ballX / 256 > 39 ) {
        // Dann gib ihm einen Tritt nach links
        speedX = (char) -100;
        ballX = 39 * 256;
      }

      // ist der Ball am oberen Rand?
      if ( ballY < 0 ) {
        speedY = -(speedY - speedY / 8);
        ballY = 0;
      }
      // ist der Ball am unteren Rand?
      else if ( ballY / 256 > 24 ) {
        // dann abprallen lassen
        speedY = -(speedY - speedY / 8);
        ballY = 24 * 256;
      }

      // neue Ballposition zeichnen
      cputcxy( (char) (ballX / 256), (char) (ballY / 256), 'O' );
    } while ( kbhit() == 0 );

    clrscr();

    return 0;
  }
}
```

## Transcompilation strategies

Beside the many deficits some transformations are done by the J2C transcompiler.

  * For every standard library header there is Java file; `tutego.j2c.include.Stdio` is the corresponding class for `<stdio.h>` for example. Those declaration are mostly declared as `native` methods in Java because it's a way to declare those methods in a type-safe way. Every call to a Java method from this class will be seen as a call to a function of this corresponding C-library.
  * A class `Language` declares special methods `sizeof()` and `asm()`.
  * `byte` is mapped to `signed char`
  * `boolean` is mapped to `char`, `true` to 1 and `false` to 0.
  * If an identifier in Java is named like a C reserved keyword--like `restrict` or `auto`--it will be renamed, for example from `register` to `__register__`.
  * Unicode identifiers are converted to valid C identifiers; every Unicode character is escaped to `__u<HEXOFUNICODE>__`.

## What you can do

Some extensions are very easy, others are some more time-consuming. If you like to help and extend the transcompiler, you could (from easy to more work-intensive):

  * How much C should be supported? Should there be arbitrary `goto`s in Java for example? 
  * Give better error messages or what’s supported and what’s not
  * Map Java’s `new` to `malloc()` or friends
  * Collect function signatures and generate prototypes or a header file
  * Declare `__func_ _` in Java
  * Indent the output
  * How to denote the address operator, maybe `int adr = address(variable)`?
  * How to map Java’s unsigned right shift `>>>` to C?
  * Allow for-each over an array and produce an array iteration
  * Produce special code for `switch(String)`
  * Do we have to introduce literal suffixes in the C program?
  * Make arrays work
  * How to deal with the duality of pointer and arrays?  Find a way to map `String` to `char[]`/`char*`. Introduce a new method `pointerAsArray(…)`?
  * Allow to access memory directly, maybe with something like `$[ 100 ] = $[ 101 ];`
  * Map Java enums to C enums
  * Map the CC65 header files to corresponding Java declarations
  * Allow developers to use String concatenation and `java.lang.String` methods and map it to C functions
  * Allow to embed native asm code, maybe the way GWT does in a comment
  * How to allow overloaded methods? With using a type prefix, like `max_II` for example. But then distinguish known C methods from new Java methods.
  * Map Java exceptions to signals
  * Map inner classes to C `struct`, should we use `typedef`?
  * Find a way to express C `union` and bit fields, maybe `@Union class`?
  * Allow pointer and references
  * Introduce object-like programming: map Java methods from inner classed to functions which accept a pointer to a struct as their first parameter
  * Introduce float pointing expressions and map to native C64 methods 
  * Should call to Java methods be rewritten? Like `Sytem.exit(n)` to `exit(n)` (and `<stdlib.h>` will be imported),  and so on? This would make it easier to port code with calls to some Java library methods, like `Math.max(..)` or `Integer.toString(…)`
  * You name it

## Links

  * JCGO http://www.ivmaisoft.com/jcgo/
  * CC65 Compiler: http://www.cc65.org/ 
  * C specification: http://www.open-std.org/jtc1/sc22/WG14/www/docs/n1256.pdf
  * Java language specification: http://docs.oracle.com/javase/specs/jls/se7/html/jls-18.html
  * WinWICE: http://www.viceteam.org/
  * Java-API of the AST visitor: http://docs.oracle.com/javase/8/docs/jdk/api/javac/tree/index.html
