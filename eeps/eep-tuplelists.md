    Author: Maria Scott <maria-12648430(at)hnc-agency(dot)org>
            Jan Uhlig <juhlig(at)hnc-agency(dot)org>
    Status: Draft
    Type: Standards Track
    Created: 07-Apr-2022
    Erlang-Version: 26
    Post-History:
****
EEP XXX: Module for lists of tuples 
----



Abstract
========

This document describes a new stdlib module `tuplelists` for working with
lists of tuples, which improves the current set of `key*` functions in
the `lists` module by removing their inconsistencies and legacy bugs while
at the same time address some of their shortcomings.

Another goal of the new module is to create an appropriate complement
for the `maplists` module described in [EEP-maplists][]
(not assigned a number at the time of this writing), which would otherwise
have to be modeled on the key functions in the `lists` module and thereby
port all their inconsistencies and legacy bugs only for the sake of
compatibility.



Motivation
==========

The `lists` module currently provides the following functions for working with
lists of tuples:

* `lists:keydelete/3`
* `lists:keyfind/3`
* `lists:keymap/3`
* `lists:keymember/3`
* `lists:keymerge/3`
* `lists:keyreplace/4`
* `lists:keysearch/3`
* `lists:keysort/2`
* `lists:keystore/4`
* `lists:keytake/3`
* `lists:ukeymerge/3`
* `lists:ukeysort/2`

With those old battle-tested functions already there, the question why we
propose a new module for the same functionality warrants some elaboration.



Misplacement
------------

The `lists` module contains functions that are meant for working
with arbitrary lists, that is, lists of arbitrary elements. Following
this statement, its functions should not, by their implementation,
contain assumptions as to the type and format of elements.

The key functions violate this principle by working only on elements
which are specifically tuples of a minimum size. Some of the key
functions even expect _all_ elements of a list to be as such.
Any such functions should be in a dedicated module. Them being in
the lists module after all presumably stems from a time when such
considerations were of a lower priority than they are today.

Furthermore, in [OTP PR 4831][], similar functionality was suggested
for lists of maps in the `lists` module, with the conclusion that
such functionality should be in a separate module `maplists`, which
in turn lead to the aforementioned [EEP-maplists][] for
the implementation of such a module.



Inconsistencies
---------------

* All non-key functions work by _matching_, but the key functions
  work by _comparing equal_. For example, `lists:delete(1, [1.0])`
  returns `[1.0]`, but `lists:keydelete(1, 1, [{1.0}])` returns `[]`.

* Many (not all) of the non-key functions have a variant where a
  function can be used to enable more sophisticated processing, but
  none of the key functions has any of those (except `keymap`,
  naturally).

* `keystore/4` replaces or _appends_ an given given value to a list,
  but the general practice is otherwise to _prepend_ with the `cons`
  operator.

* `keytake/3` is the only of the key functions that returns a
  tagged (`{value, ...}`) tuple (except `keysearch/3`). Tagging is
  not necessary even, as the function returns either `false` iff no
  tuple with the requested key was found in the list, or a _tuple_
  consisting of the tuple found for the requested key and the remainder
  of the list iff such a tuple was found.
  `keysearch/3` the other of the key functions that return a tagged
  tuple, equally unecessary, has been superseded by `keyfind/3`,
  which is the same without the `value` tag (internally, `keysearch/3`
  is implemented somewhat like `{value, keyfind/3}` (simplified)).

* The key functions used for looking up a key return `false` if no
  such tuple is found in the list. Many other, more recent modules
  return `error` in such a case, for example `maps:take/2`.

* The order of arguments for the key functions is `Key, N, ...`,
  where `N` is the index of the element in the tuple, and `Key` is
  the actual _value_ that is to be found in this place and is thus
  a misnomer and should be `Value`. Common intuition goes by the
  order of `Key, Value`, as can also be seen in modules like
  `maps` or `dict`, so `N, Value` (`N, Key` in the actual naming)
  would be a better order of arguments.



Edge case bugs
--------------

* `keysort/3` and `ukeysort/2` accept and return non- or too short
  tuples in a list if that is the only element in the list, but raise
  an exception if such an element is in a list with more than one
  element. For example, `lists:keysort(1, [a])` returns `[a]`, but
  `lists:keysort(1, [a, {b}])` raises an exception.

* `keymerge/3` and `ukeymerge/3` accept and return non- or too short
  tuples in the first list as long as the second list is empty, but
  raise an exception if such an element is in one of the lists as
  long as the second list is non-empty. For example,
  `lists:keymerge(1, [a, b, c], [])` returns `[a, b, c]`, but
  `lists:keymerge(1, [a, b, c], [{d}])` and even `lists:keymerge(1, [], [a])`
  raise an exception.

It would not be too hard to fix those bugs in the functions pointed
out above themselves. However, it would risk breaking a lot of old and
not-so-old code that just happens to work with the bugs having
no impact there.



`maplists`
----------

With the proposal of the `maplists` module mentioned earlier, to provide
the same functionality for lists of maps as the key functions do for lists
of tuples, the question arises if its functions should really be
modeled on the key functions in `lists`, in the sense of "same
as `lists:key*`, but for maps".

If **yes**, the `maplists` module would have to mimick the key functions
as close as possible, even the inconsistencies and bugs mentioned above, to
be a true drop-in addition.

If **no** (in order to overcome the mentioned inconsistencies and
bugs), the functions in the `maplists` module would be incompatible
with their `lists:key*` complements.

A new module without copying known old bugs and inconsistencies for the
sake of compatibility seems to us clearly the preferable option. Another
point to consider is that if the `maplists` module was implemented by
mimicking the key functions, a later separation of the key functions
from the `lists` module as proposed in this document would face a similar
dilemma:
The implementation would either have to mimick the inconsistencies and
bugs that were ported from the key functions into `maplists` before,
or require another new implementation for lists of maps to be its
complement. Note that at that time, the name `maplists` would be taken,
so even naming would be difficult.

This document aims to provide the tuple complement for a `maplists`
implementation without any legacy.



Rationale
=========

This document proposes a new module `tuplelists`, which should contain functions
analogous (not equivalent) to the key functions in the `lists` module.
This implicitly solves the Misplacement issue mentioned in the Motivation.

The functions shall work by _matching_ instead of _comparing equal_, which is
arguably the more intuitive. By and large, it should rarely make a difference.

The functions shall have an accompanying `*_with` function where appropriate,
with the same purpose as the respective function without the `_with` but
which accepts a predicate function instead of a fixed key, if more advanced
processing than a simple match is needed.

`keystore` shall replace or _prepend_ (instead of _append_), to be more in line
with what one would get by consing.

`keytake` shall return a tuple of the taken tuple and the remaining list if
a tuple with a matching key is found, instead of the same tagged with the atom
`value`.

`keysearch` shall be omitted, as the same functionality is provided by `keyfind`
but without the unnecessary tagging.

The functions shall return `error` to indicate "not found", instead of `false`.

The function argument `N` shall be renamed to `KeyPos`, and `Key` to `Value`,
and their order shall be reversed from `Key, N` to `KeyPos, Value`, to get rid
of the mis-naming and follow the more intuitive order of key-value.

The edge case bugs concerning `keysort`, `ukeysort`, `keymerge` and `ukeymerge`
outlined in the Motivation are to be removed, that is, they shall always raise an
exception if the provided list(s) contain non- or too short tuples.



Backwards compatibility
=======================

The proposal introduces no backwards incompatible changes but only additions.

Knowing that the key functions have been around practically forever, they are
undoubtedly scattered all over the place in any application in existence. In
OTP alone, a search for `lists:key*` and `lists:ukey*` yields more than 3600
results.

Therefore, this document does not propose the deprecation or even the removal
of the current key functions, but to put notes on them pointing to the new
`tuplelists` module for a while, and _maybe_ at some point remove them from
the documentation.

It is our hope that users will gradually tend to the `tuplelists` module when
writing new code, simply because it is nicer.
As an analogy, there are still `sets`, `ordsets` and `gb_trees` several years
after maps have been introduced, and they can still be used as always, but
today everybody uses maps instead, because they are so much nicer. Admittedly,
the contribution of the `tuplelists` module proposed in this document is certainly
not as big as the advent of maps was.



Considerations
==============

As the proposed `tuplelists` module is intended as a complement to the proposed 
`maplists` module, synchronization, discussion and close interaction with the
author of the respective [EEP-maplists][] is required to achieve a consistent
outcome.
It may even be feasible to merge this EEP proposal with the one for `maplists`,
as their goals are very similar.



[OTP PR 4831]: https://github.com/erlang/otp/pull/4831
    "Add new lists:mapkeyfind/3 function to map finding"

[EEP-maplists]: https://github.com/erlang/eep/pull/29
    "EEP XXX: Module for lists of maps"



Copyright
=========

This document is placed in the public domain or under the CC0-1.0-Universal
license, whichever is more permissive.



[EmacsVar]: <> "Local Variables:"
[EmacsVar]: <> "mode: indented-text"
[EmacsVar]: <> "indent-tabs-mode: nil"
[EmacsVar]: <> "sentence-end-double-space: t"
[EmacsVar]: <> "fill-column: 70"
[EmacsVar]: <> "coding: utf-8"
[EmacsVar]: <> "End:"
[VimVar]: <> " vim: set fileencoding=utf-8 expandtab shiftwidth=4 softtabstop=4: "
