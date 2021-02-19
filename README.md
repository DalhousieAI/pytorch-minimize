PyTorch Minimize
================

[![Build Status](https://travis-ci.com/gngdb/pytorch-minimize.svg?branch=master)](https://travis-ci.com/gngdb/pytorch-minimize)

Use [`scipy.optimize.minimize`][scipy] as a PyTorch Optimizer.

*Warning*: this project is mostly a proof of concept and not intended for
deployment to anything (for example, parameter arrays will be shuttled on
and off the GPU between numpy and PyTorch if the GPU is used, which may be
slow).

Quickstart
----------

The dependencies are `pytorch` and `scipy`. In a python environment,
install either by installing from the github repository:

``` 
python -m pip install git+https://github.com/gngdb/pytorch-minimize.git
```

Or by cloning the repository and then using dev install:

```
git clone https://github.com/gngdb/pytorch-minimize.git
cd pytorch-minimize
python -m pip install .
```

The optimizer acts like a standard [PyTorch Optimizer][optimizer], taking a
generator of parameters, and is configured by passing a dictionary of
arguments for [`scipy.optimize.minimize`][scipy]:

```
from pytorch_minimize.optim import MinimizeWrapper
minimizer_args = dict(method='CG', options={'disp':True, 'maxiter':100})
optimizer = MinimizeWrapper(model.parameters(), minimizer_args)
```

The difference to using this and most PyTorch optimizers is you need to
define a [closure][]:

```
def closure():
    optimizer.zero_grad()
    output = model(input)
    loss = loss_fn(output, target)
    loss.backward()
    return loss
optimizer.step(closure)
```

This optimizer is intended for **deterministic optimisation problems**,
such as [full batch learning problems][batch]. Because of this,
`.step(closure)` only needs to be called **once**, with the number of
iterations chosen in `minimizer_args['options']['maxiter']` as above.

Which Algorithms Are Supported?
-------------------------------

Using PyTorch to calculate the Jacobian, the following algorithms are
supported, along with their `method` name:

* [Conjugate Gradients][conjugate]: `'CG'`
* [Broyden-Fletcher-Goldfarb-Shanno (BFGS)][bfgs]: `'BFGS'`
* [Limited-memory BFGS][lbfgs]: `'L-BFGS-B'`
* [Sequential Least Squares Programming][slsqp]: `'SLSQP'`

To use the methods that require evaluating the Hessian a `Closure` object
with the following methods is required (full MNIST example
[here](./mnist/hessian_logistic_regression.py)):

```
class Closure():
    def __init__(self, model):
        self.model = model
    
    @staticmethod
    def loss(model):
        output = model(data)
        return F.nll_loss(output, target) 

    def __call__(self):
        optimizer.zero_grad()
        loss = self.loss(self.model)
        loss.backward()
        self._loss = loss.item()
        return loss
closure = Closure(model)
```

The following methods can then be used with some questionable hacks (see
below) by evaluating the beta
[`torch.autograd.functional.hessian`][torchhessian]:

* [Newton Conjugate Gradient](https://youtu.be/0qUAb94CpOw?t=30m41s): `'Newton-CG'`
* [Newton Conjugate Gradient Trust-Region][trust]: `'trust-ncg'`
* [Krylov Subspace Trust-Region][krylov]: `'trust-krylov'`
* [Nearly Exact Trust-Region][trust]: `'trust-exact'`
* [Constrained Trust-Region][trust]: `'trust-constr'`

All the above methods are included in the tests and converge on a toy
classification problem.

Algorithms You Can Choose But Don't Work
----------------------------------------

A few algorithms tested didn't converge on the toy problem, or threw
errors. You can still select them but they may not work:

* [Truncated Newton][tnc]: `'TNC'`
* [Dogleg][]: `'dogleg'`

How Does This Evaluate the Hessian?
-----------------------------------

To evaluate the hessian in PyTorch,
[`torch.autograd.functional.hessian`][torchhessian] takes two arguments:

* `func`: function that returns a scalar
* `inputs`: variables to take the derivative wrt

In most PyTorch code, `inputs` is a list of tensors embedded in the
`Modules` that make up the `model`. They can't be passed as `inputs`
because we typically don't have a `func` that will take the parameters as
input, build a network from these parameters and then produce a scalar
output.

From a [discussion on the PyTorch forum][forum] the only way to calculate
the gradient with respect to the parameters would be to monkey patch
`inputs` into the model and then calculate the loss. I wrote a [generic
recursive monkey patch][monkey] that operates on a deepcopy of the original
`model`. This involves copying everything in the model so it's probably not
very fast.

The function passed to `scipy.optimize.minimize` as `hess` does the
following:

1. [`copy.deepcopy`][deepcopy] the entire `model` Module
2. input `x` is a numpy array so cast it to tensor float32 and
`require_grad`
3. define a function `f` that unpacks this 1D numpy array into parameter
tensors
    * [Recursively navigate][re_attr] the module object
        - Deleting all existing parameters
        - Replace them with unpacked parameters from step 2
    * Calculate the loss using the static method stored in the `closure` object
5. pass `f` to `torch.autograd.functional.hessian` with `x` and cast the
result back into a numpy array

Credits
-------

This package was created with [Cookiecutter][] and the
[`audreyr/cookiecutter-pypackage`][audreyr] project template.

[re_attr]: https://stackoverflow.com/a/31174427/6937913 
[deepcopy]: https://docs.python.org/3/library/copy.html#copy.deepcopy
[monkey]: https://github.com/gngdb/pytorch-minimize/blob/master/pytorch_minimize/optim.py#L98-L114
[forum]: https://discuss.pytorch.org/t/using-autograd-functional-jacobian-hessian-with-respect-to-nn-module-parameters/103994/3
[dogleg]: https://en.wikipedia.org/wiki/Powell%27s_dog_leg_method
[tnc]: https://en.wikipedia.org/wiki/Truncated_Newton_method
[krylov]: https://epubs.siam.org/doi/abs/10.1137/1.9780898719857.ch5
[trust]: https://en.wikipedia.org/wiki/Trust_region
[torchhessian]: https://pytorch.org/docs/stable/autograd.html#torch.autograd.functional.hessian
[slsqp]: https://en.wikipedia.org/wiki/Sequential_quadratic_programming
[conjugate]: https://en.wikipedia.org/wiki/Conjugate_gradient_method
[bfgs]: https://en.wikipedia.org/wiki/Broyden%E2%80%93Fletcher%E2%80%93Goldfarb%E2%80%93Shanno_algorithm
[batch]: https://towardsdatascience.com/batch-mini-batch-stochastic-gradient-descent-7a62ecba642a
[closure]: https://pytorch.org/docs/stable/optim.html#optimizer-step-closure
[optimizer]: https://pytorch.org/docs/stable/optim.html
[scipy]: https://docs.scipy.org/doc/scipy/reference/generated/scipy.optimize.minimize.html
[Cookiecutter]: https://github.com/audreyr/cookiecutter
[audreyr]: https://github.com/audreyr/cookiecutter-pypackage