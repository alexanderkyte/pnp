# pnp

Scala library for probabilistic neural programming. 

## Installation

This library depends on DyNet with the
[Scala DyNet bindings](https://github.com/allenai/dynet/tree/master/swig).
See the link for build instructions. After building this library, run
the following commands from the `pnp` root directory:

```
cd lib
ln -s <PATH_TO_DYNET>/build/swig/dynet_swigJNI_scala.jar .
ln -s <PATH_TO_DYNET>/build/swig/libdynet_swig.jnilib .
```

That's it! Verify that your installation works by running `sbt test`
in the root directory.

## Usage

This section describes how to use probabilistic neural programs to
define and train a model. The typical usage has four steps:

1. **Define the model.** Models are implemented by writing a function
   that takes your problem input and outputs `Pnp[X]` objects. The
   probabilistic neural program type `Pnp[X]` represents a function
   from neural network parameters to probability distributions over
   values of type `X`. Each program describes a (possibly infinite)
   space of executions, each of which returns a value of type `X`.
2. **Generate labels.** Labels are implemented as functions that assign
   costs to program executions or as conditional distributions over
   correct executions.
3. **Train.** Training is performed by passing a list of examples to a
   `Trainer`, where each example consists of a `Pnp[X]` object and a
   label. Many training algorithms can be used, from loglikelihood to
   learning-to-search algorithms.
4. **Run the model.** A model can be runned on a new input by
   constructing the appropriate `Pnp[X]` object, then running
   inference on this object with trained parameters.

These steps are illustrated in detail for a sequence-to-sequence
neural translation model in
[Seq2Seq2.scala](src/main/scala/org/allenai/pnp/examples/Seq2Seq.scala).

### Defining Probabilistic Neural Programs

Probabilistic neural programs are specified by writing the forward
computation of a neural network, using the `choose` operation to
represent discrete choices. Roughly, we can write: 

```scala
val pnp = for {
  scores1 <- ... some neural net operations ...
  // Make a discrete choice
  x1 <- choose(values, scores1)
  scores2 <- ... more neural net operations, may depend on x1 ...
  ...
  xn <- choose(values, scoresn)
} yield {
  xn
}
```

`pnp` then represents a function that takes some neural network
parameters and returns a distribution over possible values of `xn`
(which in turn depends on the values of intermediate choices). We 
evaluate `pnp` by running inference, which simulatenously runs the
forward pass of the network and performs probabilistic inference:

```scala
nnParams = ... 
val dist = pnp.beamSearch(10, nnParams)
```

#### Choose

The `choose` operator defines a distribution over a list of values:

```scala
val flip: Pnp[Boolean] = choose(Seq(true, false), Seq(0.5, 0.5))
```

This snippet creates a probability distribution that returns either
true or false with 50% probability. `flip` has type `Pnp[Boolean]`,
which represents a function from neural network parameters to
probability distributions over values of type `Boolean`. (In this case
it's just a probability distribution since we haven't referenced any
parameters.)  Note that `flip` is not a draw from the distribution,
rather, *it is the distribution itself*. The probability of each
choice can be given to `choose` either in an explicit list (as above)
or via an `Expression` of a neural network.

We compose distributions using `for {...} yield {...}`:

```scala
val twoFlips: Pnp[Boolean] = for {
  x <- flip
  y <- flip
} yield {
  x && y
}
```

This program returns `true` if two independent draws from `flip` both
return `true`. The notation `x <- flip` can be thought of as drawing a
value from `flip` and assigning it to `x`. However, we can only use
the value within the for/yield block to construct another probability
distribution.

#### Neural Networks

Probabilistic neural programs have access to an underlying computation
graph that is used to define neural networks.

```scala
def mlp(x: FloatVector): Pnp[Boolean] = {
  for {
    // Get the computation graph
    cg <- computationGraph()

    // Get the parameters of a multilayer perceptron by name.
    // The dimensionalities and values of these parameters are 
    // defined in a PnpModel that is passed to inference.
    weights1 <- param("layer1Weights")
    bias1 <- param("layer1Bias")
    weights2 <- param("layer1Weights")

    // Input the feature vector to the computation graph and
    // run the multilayer perceptron to produce scores.
    inputExpression = input(cg.cg, Seq(FEATURE_VECTOR_DIM), x)
    scores = weights2 * tanh((weights1 * inputExpression) + bias1)

     // Choose a label given the scores. Scores is expected to
     // be a 2-element vector, where the first element is the score
     // of true, etc.
     y <- choose(Array(true, false), scores)
  } yield {
    y
  }
}
```


TODO: finish docs
