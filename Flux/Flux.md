It is hard to call [Flux.jl](https://github.com/FluxML/Flux.jl) a deep learning framework and its implementation idea are not like modern deep learning frameworks (like TensorFlow or Pytorch) at all. However, the most interesting part of Flux.jl is that it is a nice wrapper of JPL's language-level auto differentiation capability in supporting deep learning task. But to support complicated deep learning tasks, Flux.jl may need to be largely rewritten.

The following problems can be solved based on Flux/jl's current

### [high priority] Lacking some basic functionalities.

1. [_**Terrible**_] Models (trained parameters) trained by Flux.jl cannot be serialized. As a result, they cannot be loaded for inference.
1. Data shuffle is not implemented in `Flux.train`. Shuffle is extremely important for SGD. Actually, the interface for the training loop may need re-design.
1. No implementations for embedding layer for NLP tasks. The learnable parameter of the embedding layer is sparsely updated which required a customized gradient function.
1. Too few tunable hyperparameters.
    * Deep learning tasks have many hyperparameters to tune. For example, each operator can have its own learning rate, regularization method and associated tunable hyperparameters, the initialization method, and associated tunable hyperparameters, and so on.
    * Flux.jl's current implementation idea is more for allowing users to hack (or to say, directly modify the source codes) everything they needed.
1. Any deep learning tasks consist of five steps (orders of these steps cannot be changed): forward computation, backward computation, gradient clipping, regularization, and optimization. Week supports for gradient clipping and regularization steps.
1. No data parallelism implementations.

---

### Problems about AD

1. Forward-mode AD probably may not be efficient enough for deep learning tasks.

    * [Flux.jl](https://github.com/FluxML/Flux.jl) uses JPL's [ForwardDiff](https://github.com/JuliaDiff/ForwardDiff.jl) project to calculate gradients. However, reverse mode auto differentiation dominates deep learning optimizations due to the fact that forward-mode auto differentiation will be inefficient for functions that have many inputs.
    * Admittedly, JPL uses a very special optimization for vector forward-mode AD which makes the forward-mode AD for vectorized computation (like elementwise computations) very efficient. Such optimizations can fuse consecutive vectorized computations and gradients computations into only one kernel. This also reduces much overhead of dynamic allocations.
    * But for computations beyond elementwise computations, is forward-mode AD still comparative to the reverse-mode AD? Theoretically seems not.
      * So is it necessary to also include reverse-mode AD into Flux.jl, or Flux.jl has already made some efforts on this, but I did not realize it?
      * However, [ReverseDiff.jl](https://github.com/JuliaDiff/ReverseDiff.jl) project is not actively developed, is there any recommendations of reverse mode AD in JPL?

1. There are some restrictions when using ForwardDiff.jl, thus may need enhancements for some special cases.
    * It cannot propagate derivative information through non-Julia code. For non-Julia codes, like vendor specific kernels, the JPL compiler cannot rewrite function's original input types by a corresponding dual number type. Therefore, some hacks are needed to enable some vendor optimized computation kernels and user-customized kernels such as the word embedding layer for NLP tasks.
    * Such enhancements are possible. Tracker in Flux.jl is a wrapper for the gradient computations of a deep learning model.
    * It is possible to use both forward-mode and reverse-mode AD.

### Other problems

1. Some important kernels for deep learning tasks may need further optimizations [_**This may not be a high priority right now, but it is hard to expect the model will be very fast**_].

    * If there are vendor optimized computation kernels available, it is better to make use of them directly.
    * Convolution and Gemm are two dominant computations in deep learning problems.
      * Flux.jl relies on [NNLib](https://github.com/FluxML/NNlib.jl) which implements the convolution and pooling functions (and some other common computation functions) for deep learning models. The current implementation of convolution in NNLib is a JPL's native implementation of im2col followed by Gemm. This is one implementation of convolution. Not sure for its speed in JPL. But it is to re-implementation
      * Convolution and pooling may need

1. [_**Not important right now, but potentially will affect the performance**_] No explicit memory management module, performance relies on GC.

### A hard decision

1. One significant difference between Flux.jl and other deep learning frameworks is Flux.jl is not a computation graph engine. It does not implement any explicit semantics of operators or the computation graph. Therefore, _**no operator-level scheduling**_ is implemented to maximize concurrency and make the best use of all CPU cores.
    * Follow Flux.jl's current way or change (actually, rewrite) it to a graph engine?
