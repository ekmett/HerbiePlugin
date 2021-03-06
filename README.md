# Herbie GHC Plugin ![](https://travis-ci.org/mikeizbicki/HerbiePlugin.svg)

The Herbie [GHC Plugin](https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/compiler-plugins.html) automatically improves the [numerical stability](https://en.wikipedia.org/wiki/Numerical_stability) of your Haskell code.
The Herbie plugin fully supports the [SubHask](http://github.com/mikeizbicki/subhask) numeric prelude,
and partially supports the standard prelude (see the [known bugs section](#bugs) below).

This README is organized into the following sections:

* [Example: linear algebra and numerical stability](#example-linear-algebra-and-numerical-instability)
* [How the Herbie plugin works](#how-it-works)
* [Installing and using the Herbie plugin](#installing-and-using-the-herbie-plugin)
* [Compiling all of stackage with the Herbie plugin](#compiling-all-of-stackage-with-the-herbie-plugin)
* [Known bugs](#known-bugs)

## Example: linear algebra and numerical instability

The popular [linear](https://hackage.haskell.org/package/linear) library contains [the following calculation](https://github.com/ekmett/linear/blob/35dcce4152c1a26e0d82e9ac75c3b77607b2aa3c/src/Linear/Projection.hs#L73):

```
w :: Double -> Double -> Double
w far near = -(2 * far * near) / (far - near)
```

This code looks correct, but it can give the wrong answer.
When the values of `far` and `near` are both very small (or very large), the product `far * near` will underflow to 0 (or overflow to infinity).
In the worst case scenario using `Double`s, this calculation can lose up to 14 bits of information.

The Herbie plugin automatically prevents this class of bugs (with some technical caveats, see [how it works](#how-it-works) below).
If you compile the linear package with Herbie, the code above gets rewritten to:

```
w :: Double -> Double -> Double
w near far = if near < -1.7210442634149447e+81
    then (2.0 * near / (near - far)) * far
    else if near < 8.364504563556443e+16
        then (2.0 * near) / ((near - far) / far)
        else ((2.0 * near) / (near - far)) * far
```

This modified code is numerically stable.
The if statements check to see which regime we are in (very small or very large)
and select the calculation that is most appropriate for this regime.

The [test suite](/test/Tests.hs) contains MANY more examples of the types of expressions the Herbie plugin can analyze.

## How it works

[GHC Plugins](https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/compiler-plugins.html)
let library authors add new features to the Haskell compiler.
The Herbie plugin gets run after type checking, but before any optimizations.
Because GHC is so good at optimizing,
the code generated by the Herbie plugin is just as fast as hand-written code.
The plugin has two key components:
first it finds the floating point computations in your code;
then it replaces them with numerically stable versions.

### Finding the computations

When the Herbie plugin is run, it traverses your code's [abstract syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) looking for mathematical expressions.
These expressions may:

* consist of the following operators:
  `/`
  , `*`
  , `-`
  , `*`
  , `**`
  , `^`
  , `^^`
  , `$`
  , `cos`
  , `sin`
  , `tan`
  , `acos`
  , `asin`
  , `atan`
  , `cosh`
  , `sinh`
  , `tanh`
  , `exp`
  , `log`
  , `sqrt`
  , `abs`

* contain an arbitrary number of free variables

* contain arbitrary non-mathematical subexpressions

For example, in the function below:
```
test :: String -> String
test str = show ( sqrt (1+fromIntegral (length str))
                - sqrt (fromIntegral (length str))
                :: Float
                )
```
the Herbie plugin extracts the expression `sqrt (x+1) - sqrt x`;
calculates the numerically stable version `1 / (sqrt (x+1) + sqrt x)`;
and then substitutes back in.

The Herbie plugin performs this procedure on both concrete types (`Float` and `Double`) and polymorphic types.
For example, the plugin will rewrite
```
f :: Field a => a -> a -> a
f far near = (far+near)/(far*near*2)
```
to
```
f :: Field a => a -> a -> a
f far near = 0.5/far + 0.5/near
```
(The `Field` constraint comes from [SubHask](http://github.com/mikeizbicki/subhask).)

These polymorphic rewrites are always guaranteed to preserve the semantics when the expression is evaluated on an exact numeric type.
So both versions of `f` above are guaranteed to give the same result when called on `Rational` values,
but the rewritten version will be more accurate when called on `Float` or `Double`.

The main limitation of the Herbie plugin is that any recursive part of an expression is ignored.
This is because analyzing the numeric stability of a Turing complete language is undecidable
(and no practical heuristics are known).
<!--Creating a practical algorithm that can analyze the cases programmers encounter in practice remains an open research problem.-->
Fortunately, the Herbie plugin can still analyze the non-recursive subexpressions of a recursive expression.
So in the code:
```
go :: Float -> Float -> Float
go 0 b = b
go a b = go (a-1) (sqrt $ (a+b) * (a+b))
```
the expression `sqrt $ (a+b) * (a+b)` gets rewritten by Herbie into `abs (a+b)`.

### Improving the stability of expressions

The Herbie plugin uses two sources of information to find numerically stable replacements for expressions.
The simplest source is a sqlite3 database.
This database contains about 400 expressions that are known to be used by Haskell libraries.

The more important source is the [Herbie program](http://herbie.uwplse.org/).
Herbie is a recent research project on using probabilistic searches to find numerically stable expressions.
Because Herbie is probabilistic, it provides weak theoretic guarantees on the numerical stability of the resulting expressions;
but in practice, the improved expressions are significantly better.
To ensure reproducible builds, the same random seed is used on all calls to the Herbie program.
For more details on how Herbie works, check out the [PLDI15 paper](http://herbie.uwplse.org/pldi15.html).

The Herbie program can take a long time to run.
If the program doesn't return a solution within two minutes,
then the Herbie plugin assumes that no better solution is possible and continues processing.
To improve compile times, every time the Herbie program returns a new solution,
the solution is added to the `Herbie.db` database.
When compiling a file, if all the math expressions are already in the database,
then the Herbie plugin imposes essentially no overhead on compile times.

## Installing and Using the Herbie Plugin

The Herbie plugin requires GHC 7.10.1 or 7.10.2.
It is installable via cabal using the command:
```
cabal update && cabal install HerbiePlugin
```

It is recommended (but not required) that you also install the Herbie program.
(Without installing the program, the Herbie plugin can only replace expressions in the standard database.)
The Herbie program is written in racket,
so you must first install Racket.
Go to the [download page](http://download.racket-lang.org/racket-v6.1.html) for Racket 6.1.1 and install the version for your platform.
Then run the commands:
```
git clone http://github.com/mikeizbicki/HerbiePlugin --recursive
cd HerbiePlugin/herbie
raco exe -o herbie-exec herbie/interface/inout.rkt
```
The last line creates an executable `herbie-exec` that the Herbie plugin will try to call.
You must move the program somewhere into your `PATH` for these calls to succeed.
One way to do this is with the command:
```
sudo mv herbie-exec /usr/local/bin
```

To compile a file with the Herbie plugin, you need to pass the flag `-fplugin=Herbie` to when calling GHC.
You can have `cabal install` automatically apply the Herbie plugin by passing the following flags
```
--ghc-option=-fplugin=Herbie
--ghc-option=-package-id herbie-haskell-0.1.0.0-50ba55c8f248a3301dce2d3339977982
```

## Running Herbie on all of stackage

[Stackage LTS-3.5](https://www.stackage.org/lts-3.5) is a collection of 1351 of the most popular Haskell libraries.
The script [install.sh]() compiles all of stackage using the Herbie plugin.

48 of the 1351 packages (3.5%) use floating point computations internally.
Of these, 40 packages (83%) contain expressions whose numerical stability is improved with the Herbie plugin.
In total, there are 303 distinct numerical expressions used in all stackage packages,
and 164 of these expressions (54%) are more stable using the Herbie plugin.

The table below shows a detailed breakdown of which packages contain the unstable expressions.
Notice that the unstable and stable expression columns may not add up to the total expressions column.
The difference is the expressions could not be analyzed because the Herbie program timed out.

| package | total math expressions | unstable expressions | stable expressions |
| ------- | ---------------------- | -------------------- | ------------------ |
| math-functions-0.1.5.2|92|50|34 |
| colour-2.3.3|28|8|18 |
| linear-1.19.1.3|28|19|8 |
| diagrams-lib-1.3.0.3|25|15|10 |
| diagrams-solve-0.1|25|14|11 |
| statistics-0.13.2.3|15|5|7 |
| plot-0.2.3.4|12|9|2 |
| random-fu-0.2.6.2|11|4|5 |
| Chart-1.5.3|10|8|2 |
| circle-packing-0.1.0.4|9|3|6 |
| mwc-random-0.13.3.2|9|1|7 |
| Rasterific-0.6.1|8|3|5 |
| diagrams-contrib-1.3.0.5|6|6|0 |
| log-domain-0.10.2.1|5|0|5 |
| repa-algorithms-3.4.0.1|5|3|2 |
| rasterific-svg-0.2.3.1|4|2|2 |
| sbv-4.4|4|2|2 |
| clustering-0.2.1|3|1|2 |
| erf-2.0.0.0|3|2|1 |
| hsignal-0.2.7.1|3|2|0 |
| hyperloglog-0.3.4|3|1|2 |
| integration-0.2.1|3|1|2 |
| intervals-0.7.1|3|2|1 |
| shake-0.15.5|3|2|1 |
| Chart-diagrams-1.5.1|2|2|0 |
| JuicyPixels-3.2.6.1|2|2|0 |
| Yampa-0.10.2|2|2|0 |
| YampaSynth-0.2|2|1|1 |
| diagrams-rasterific-1.3.1.3|2|1|1 |
| fay-base-0.20.0.1|2|1|1 |
| histogram-fill-0.8.4.1|2|0|2 |
| parsec-3.1.9|2|0|2 |
| smoothie-0.4.0.2|2|2|0 |
| Octree-0.5.4.2|1|1|0 |
| ad-4.2.4|1|1|0 |
| approximate-0.2.2.1|1|1|0 |
| crypto-random-0.0.9|1|1|0 |
| diagrams-cairo-1.3.0.3|1|1|0 |
| diagrams-svg-1.3.1.4|1|1|0 |
| dice-0.1|1|0|1 |
| force-layout-0.4.0.2|1|1|0 |
| gipeda-0.1.2.1|1|0|1 |
| hakyll-4.7.2.3|1|0|1 |
| hashtables-1.2.0.2|1|1|0 |
| hmatrix-0.16.1.5|1|0|1 |
| hscolour-1.23|1|0|1 |
| metrics-0.3.0.2|1|1|0 |

<!--
table created with the sqlite code:

select A.dbgComments,A.q,B.q,C.q from
    (
        (select distinct dbgComments,count(*) as q from
            StabilizerResults as A
            join
            (select distinct dbgComments,resid from DbgInfo) as B
            where A.id=B.resid
            group by B.dbgComments
        ) as A
        left outer join
        (select distinct dbgComments,count(*) as q from
            StabilizerResults as A
            join
            (select distinct dbgComments,resid from DbgInfo) as B
            where A.id=B.resid and errin-errout>0
            group by B.dbgComments
        ) as B
        on A.dbgComments=B.dbgComments
    )
    left outer join
    (select distinct dbgComments,count(*) as q from
        StabilizerResults as A
        join
        (select distinct dbgComments,resid from DbgInfo) as B
        where A.id=B.resid and errin-errout<=0
        group by B.dbgComments
    ) as C
on A.dbgComments=C.dbgComments
group by A.dbgComments
order by A.q desc
;
-->

The easiest way to find the offending code in each package is to compile the package using the Herbie plugin.

## Known Bugs

There are no known bugs when compiling programs that use the [SubHask](https://github.com/mikeizbicki/subhask) numeric prelude.

The standard Prelude is only partially supported.
In particular, the Herbie plugin is able to extract mathematical expressions correctly and will print the stabilized version to stdout.
But the plugin can only substitute the stabilized version on polymorphic expressions, and does not perform the substitution on non-polymorphic ones.
The problem is that the `solveWantedsTcM` function (called within the `getDictionary` function inside of [Herbie/CoreManip.hs](https://github.com/mikeizbicki/HerbiePlugin/blob/master/src/Herbie/CoreManip.hs)) is unable to find the `Num` dictionary for `Float` and `Double` types.
I have no idea why this is happening, and I'd be very happy to accept pull requests that fix the issue :)
