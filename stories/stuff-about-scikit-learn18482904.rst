.. title: Stuff about scikit-learn:#1848/#2904
.. slug: stuff-about-scikit-learn18482904
.. date: 2014-07-03 12:16:43 UTC-03:00
.. tags:
.. link:
.. description:
.. type: text

I'm trying a first attempt at these two scikit-learn issues: `#1848: Rename
grid search module
<http://www.github.com/scikit-learn/scikit-learn/issues/1848>`_ and `#2904:
Data independent CV iterators
<http://www.github.com/scikit-learn/scikit-learn/issues/2094>`_.

#1848 is an API refactor, merging stuff in a new ``model_selection`` module

#2904 adressess a reusability problem with current Cross Validation iterators,
namely, that the iterators are bound to the data in the constructor. The
proposed API change goes along the following lines:

.. code:: python

  # Old syntax
  for train, test in KFold(n_samples=n_samples, n_folds=5):
      print train, tests

  # New syntax
  for train, test in KFold(n_folds=5).split(y):
      print train, tests

These two issues are somewhat related, so I'm going to try to tackle them
together.

**WARNING**: Right now this development branch is broken. I've only made changes so the
``KFold`` and ``StratifiedKFold`` work as expected. The following code was used
for simple testing. Both variants of the for loops should behave the same
(although we prefer the ``split()`` one, right?)

.. code:: python

  from sklearn.model_selection.partition import KFold, StratifiedKFold
  from sklearn.datasets import load_boston, load_digits


  def main():
      boston = load_boston()
      X, y = boston.data, boston.target
  #    for train, test in KFold(len(y)):
      for train, test in KFold().split(y):
          print train, test
      digits = load_digits()
      X, y = digits.data, digits.target
  #    for train, test in StratifiedKFold(y):
      for train, test in StratifiedKFold().split(y):
          print train, test


  if __name__ == "__main__":
      main()


A little rationale/storytelling about the changes:

* As the same CV iterator can now be used several times (particularly in a
  nested way), a new object which controls the generation must be created each
  time, and must be independent from all others. This also means that
  generating the masks for a dataset should not alter the internal state of the
  CV.

  Enter ``_PartitionTestGenerator``

  This class contains all data we are trying to remove from the CV
  constructors, and calls back the CV iterator via ``_iter_tests_masks``
  **with** the data to obtain the splits. (Basically does the old
  ``CV.__iter__`` work)

* There are some quirky parts in order to maintain backwards compatibility with
  the pass-data-in-the-constructor approach. Mostly ``None`` checking and
  falling back to CV members. See ``_PartitionIterator._sample_size`` for an
  extracted example.

Big picture TODOs:

* Make all CV iterators work this way

* Support more values for ``y``: namely dicts of arrays and pandas DataFrame.
  Maybe we can find a nice structure for an internal class and normalize all
  inputs to that at the beginning, but I didn't have time yet to check this


