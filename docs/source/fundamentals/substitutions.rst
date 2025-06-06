
.. highlight: yml
.. _subst:

Substitutions and formulas
##########################

The YAML files read by Stimela are subject to three types of substituton mechanisms:

* At load time, the standard OmegaConf :ref:`variable interpolation<omegaconf_subst>` mechanism is invoked on all strings of the form ``${name}``. This can be used to substitute a previously defined value into another value. You can escape the interpolation by using ``\${name}``. This feature is not used often (though see :ref:`below<omegaconf_subst>`), but it's important to be aware of it in case you inadvertently trigger an interpolation by using ``${}``. 

* At runtime, Stimela does {}-substitutions on parameter values (and some other items such as backend settings). {}-substitutions are invoked as ``{foo.bar}``. Use ``{{foo.bar}}`` if you need to insert curly braces without a substitution.

* At runtime, Stimela does formula evaluations on parameter values (and backend settings etc.) that start with a ``=`` character. If you need to supply a literal string starting with an equals sign, use ``==`` to escape formula evaluation and insert a single equals sign.

Substitutions and formula evaluations are a key feature of Stimela, which allow the inputs and outputs of steps and recipes to be linked with minimum fuss. Here's an idealized example showing off both features::

    calibration-recipe:
        info: "a notional recipe for calibration & imaging"
        inputs:
            ms:
                dtype: MS
                required: true
                info: "measurement set to use"
            image-name:
                dtype: str
                required: true
                info: "base name for output images"
            image-size:
                dtype: int 
                default: 4096
                info: "image size, in pixels"
        steps:
            image-1:
                info: "make initial image and model from DATA column"
                cab: imager-tool
                params:
                    ms: =recipe.ms
                    mode: image
                    size: =recipe.image-size * 2
                    column: DATA
                    output.image: '{recipe.image-name}.image-{info.suffix}-{current.size:05d}.fits'
                    output.model: '{recipe.image-name}.model-{info.suffix}.fits'
            predict:
                info: "predict model into MODEL_DATA"
                cab: imager-tool
                params:
                    ms: =recipe.ms
                    mode: predict
                    model: =previous.output.model
                    column: MODEL_DATA
            calibrate:
                info: "calibrate model against data"
                cab: calibration-tool
                params:
                    ms: =recipe.ms
                    model.column: =steps.predict.column
                    output.column: CORRECTED_DATA
            image-2:
                info: "make image from calibrated data column"
                cab: imager-tool
                params:
                    ms: =recipe.ms
                    mode: image
                    column: =steps.calibrate.output.column
                    output.image: '{recipe.image-name}.image-{info.suffix}.fits'
                    output.model: '{recipe.image-name}.model-{info.suffix}.fits'

This illustrates the two key concepts:

* Parameter strings are subject to {}-substitutions in the style of Python's ``str.format()`` method (see https://docs.python.org/3/library/string.html#formatstrings). The curly brackets contain a *namespace lookup* such as ``recipe.ms`` (see below for details), and an optional format code (e.g. ``:05d``).
 
* Parameter strings that start with ``=`` are evaluated as *formulas*, which can employ namespace lookups, standard Python operators, and a number of predefined functions (see below).

Let's examine what's going on here in a bit more detail. We emphasize that this recipe is just an idealized example, referring to some notional "imager" and "calibration" tools.

Firstly, our recipe has three inputs: a measurement set, a base filename for output images, and an image size. Let us now assume we run it as:

    ``stimela run calibration-recipe.yml ms=foo.ms image-name=imfoo image-size=1024``

Here is what happens next:

* All the steps refer to the recipe's MS input via the ``=recipe.ms`` construct, which is the simplest kind of formula meaning "the value of the ``ms`` variable of the ``recipe`` namespace".

* The first step makes an image and a model from the MS. The size of the image will be 2048 pixels (``=recipe.image-size * 2``). The output filenames, formed up via {}-substitutions, will be ``imfoo.image-1-02048.fits`` and ``imfoo.model-1.fits``. Note how this is based on the recipe's ``image-name`` parameter, the image size (``current.size`` refers to the ``size`` parameter of the current step), and the "1" suffix in the step's name ("image-1"), which is given by the ``self.suffix`` substitution (see below).
 
* The second step predicts the model into the MODEL_DATA column of the MS. Note how ``previous.output.model`` refers to the ``output.model`` parameter of the previous step.

* The third step runs calibration. Note how ``=steps.predict.column`` refers to the ``column`` parameter of the ``predict`` step.

* The fourth step runs another round of imaging.

The value of the substition and formula mechanism here is obvious -- based on just three recipe inputs, a whole slew of intermediate parameters and data products are named consistently, and connected between the steps with minimum effort. 

Namespace lookup
----------------

The invocation of ``recipe.ms`` above is an example of a namespace lookup. The *namespace* is ``recipe``, and ``ms`` is the value being looked up. Namespace lookups can be nested (as in ``steps.predict.column`` above). Lookups can also refer to list elements using ``[n]``, and can also include wildcards (see below). Depending on the substitution *context*, a number of standard namespaces are available. 

In the context of a step's parameter evaluation, the following namespaces are recognized:

* ``recipe`` refers to parameters of the containing recipe.

* ``root`` refers to parameters and variables of the top-level recipe. Within the top-level recipe, this is the same as ``recipe``, but if a step contains a sub-recipe, this will be distinct inside the sub-recipe.

* ``current`` refers to parameters of the current step.

* ``previous`` refers to parameters of the previous step.

* ``steps.name`` refers to parameters of a (necessarily preceding) step named ``name``. A particularly useful twist on this is given by wildcard matching. For example, ``steps.image-*.output.model`` will match the alphanumerically highest preceding step matching ``image-*``.  

* ``self`` contains some information on the current step, in particular:

  * ``self.label`` is the step label (e.g. "image-1", "calibrate" above);
    
  * ``self.label_parts`` is a list of the components of the step label, split at the dash character. For the two steps above, this would be ["image", "1"] and ["calibrate"];
    
  * ``self.suffix`` is the last component of the label, or an empty string if the label has a single component. For the two steps above this would be "1", and an empty string.
    
  * ``self.fqname`` is the fully-qualified name of the step, e.g. ``calibration-recipe.image-1``.

  * ``self.taskname`` is similar to ``fqname``, but if the recipe is a for-loop, it will include a loop counter, i.e. ``top-recipe.0.sub-recipe.1.step``. Note that this works to any level of nesting.

  The ``self`` namespace is particularly useful for forming up filenames. (Note that ``self`` was formerly known as ``info`` in Stimela 2.0.x -- the old form still works, but is being deprecated.)

* ``config`` refers to the top-level configuration namespace, which effectively contains everything known to Stimela. For example, ``config.opts`` are options, ``config.cabs`` are cab definitions, etc.

Formula evaluation
------------------

As we saw above, a parameter value starting with ``=`` invokes the formula parser (if you need to set a parameter to the literal value "=", use ``==``.) The formula parser recognizes the following elements:

* namespace lookups, such as ``recipe.image-size`` in the example above;

* standard Python operators, namely:

  * unary operators: ``+``, ``-``, binary ``~`` and logical ``not``
    
  * binary arithmetic operators: ``**``, ``*``, ``/``, ``//``, ``+``, ``-``

  * binary bitwise shift operators: ``<<`` and ``>>``

  * binary operators: ``&``, ``^``, ``|``

  * comparison operators: ``==``, ``!=``, ``<=``, ``<``, ``>=``, ``>``

  * set/list membership operators: ``in``, ``not in``

  * logical operators: ``and``, ``or``

* the keyword ``UNSET``. A formula evaluating to ``UNSET`` will result in that parameter becoming unset.

* the keyword ``EMPTY``, evaluating to an empty string.

* item lookup, e.g. ``current.foo[item]``. The item may be any valid expression.

* built-in functions. The list of available functions is growing with every new Stimela version; at time of writing the following are available: 

  * ``IF(`` *condition, if_true, if_false[, if_unset]* ``)`` evaluates the condition, and returns *if_true* or *if_false* depending on the outcome (which is evaluated in the Pythonic sense, i.e. a zero or an empty string is considered false). If *condition* is unset (i.e. is a namespace lookup where the final element is not found), returns *if_unset*, or throws an error if the latter is omitted.

  * ``IFSET(`` *namespace_lookup[, if_set,[, if_unset]]* ``)`` checks if the namespace lookup is valid (i.e. if the final element is found). If it is valid, returns *if_set* if given, or the value if the lookup if not. If it is not valid, returns *if_unset* if given, or ``UNSET`` if omitted.  

  * ``CASES(`` *cond1, result1, cond2, result2, ...[default]* ``)`` evaluates each conditional in turn, and returns the first result corresponding to a true conditional. If none of the conditionals evaluate to true, returns the default, if supplied, else returns an unset/unresolved value.

  * ``ERROR(`` *message* ``)`` emits an error message -- this can be useful within ``IF()`` or ``CASES()``.

  * ``GLOB(`` *pattern* ``)`` returns a list of filenames matching the given pattern. 

  * ``MIN(`` *arg1[, arg2[,...]]* ``)``  and ``MAX(`` *arg1[, arg2[,...]]* ``)``, return the min/max of the arguments.

  * ``LIST(`` *arg1[, arg2[,...]]* ``)`` returns a list composed of the arguments.

  * ``RANGE(`` *N* ``)`` returns a list of integers from 0 to *N-1*. It also supports the ``RANGE(start, end)`` and ``RANGE(start, end, step)`` forms.

  * ``EXISTS(`` *path* ``)`` returns true if the file or path exists. 

  * ``DIRNAME(`` *path* ``)`` returns the directory part of the path. 
  
  * ``BASENAME(`` *path* ``)`` returns the filename part of the path. 
  
  * ``EXTENSION(`` *path* ``)`` returns the filename extension. 
   
  * ``STRIPEXT(`` *path* ``)`` returns the path minus the extension. 

  * ``IS_NUM(arg)`` true if the argument is a numeric type.

  * ``IS_STR(arg)`` true if the argument is a string type.

  * ``VALID(arg)`` true if the argument is valid, and evaluates to non-zero. This is a useful pattern when dealing with parameters of a mixed type (that can be e.g. strings or numbers). For example, ``recipe.a > 0`` would throw an  error is ``a`` is a string, but ``VALID(recipe.a > 0)`` would return False in this case.

  * ``GETITEM(list, item)`` equivalent to ``list[item]``.
  

As should be evident from the list above, certain functions expect arguments of a particular type (for example, the pathname manipulation functions expect strings). 

Note that function arguments are treated as fully-fledged expressions of their own (with the exception of the first argument of ``IFSET()``, which must be a namespace lookup by definition.) In particular, {}-substitutions are applied to string arguments. For example, the following can be a legit (and useful) invocation::

    =GLOB("{recipe.image-name}*.fits")

Formula evaluation errors
^^^^^^^^^^^^^^^^^^^^^^^^^


From the list of functions above, it should be clear that some functions expect arguments of a specific type (e.g. the pathname manipulation functions expect a string argument), while others (e.g. ``IF()``) are completely permissive. Bear this in mind if you're confounded by a strange error during parameter validation. Stimela strives to give sensible and descriptive error messages, however, the formula engine is one area where the range of possible errors is so vast that the occasional opaque message will slip through.


.. _omegaconf_subst:

OmegaConf interpolations
------------------------


A related, but more basic, kind of substitution is invoked via the ``${}`` construct. This invokes the `OmegaConf variable interpolation <https://omegaconf.readthedocs.io/en/latest/structured_config.html#interpolations>`_ mechanism::

   vars:
        x: 1
        y: ${vars.x}
 
Note that this kind of substition happens on a much more basic level, when the YAML itself is loaded. We don't tend to employ it much, since the ``_use`` and ``_include`` extensions (see next section) tend to be a lot more rich and functional.

There is one scenario where this interpolation is very useful, however. The construct ``${self:dirname}`` will interpolate to the directory of the YAML document where it appears. Similarly, ``${self:path}`` interpolates to the full path of the YAML document, and ``${self:basename}`` to its base filename. This is useful in cases where a recipe is shipped alongside some additional configuration or data files that it must reference. Since a recipe may be included from another recipe (residing in a different directory), it may be necessary for it to specify the proper path to its associated data files, which ``${self:dirname}`` allows for.








