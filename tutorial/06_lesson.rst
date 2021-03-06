Lesson 6 - Handling Failure Results
===================================

Goal
----

In this lesson we'll learn one strategy for handling results of
``FAILURE`` using the default navigation.

Get Started
-----------

Let's continue where we left off in the **new_hire.sl** flow and add
some code to deal with the case when the ``check_availability``
operation returns a result of ``FAILURE``.

Failure Handling
----------------

There is special syntax that can be used for handling ``FAILURE``
results by default. We wrap a step inside the ``on_failure`` key. Let's
add this functionality after the ``print_finish`` step.

.. code-block:: yaml

    - on_failure:
      - print_fail:
          do:
            base.print:
              - text: "${'Failed to create address: ' + address}"

Now, when any step receives a result of ``FAILURE`` from its operation
the flow will navigate to the ``on_failure`` step by default.

For more information, see :ref:`on_failure` in the DSL reference.

Run It
------

We can save and run the flow using the exact command we used in the last
lesson. This time, however, things should work slightly differently.

.. code-block:: bash

    run --f <folder path>/tutorials/hiring/new_hire.sl --cp <folder path>/tutorials --i address=john.doe@somecompany.com

In the case of the ``check_availability`` operation returning a result
of ``SUCCESS`` we expect the flow to behave exactly as it did before.
Notice that this means it will know not to run the ``on_failure`` step
without us adding any navigation instructions. This is part of the
default navigation behavior.

In the case of the ``check_availability`` operation returning a result
of ``FAILURE`` the flow will no longer terminate immediately with a
result of ``FAILURE``. Instead, the flow will continue by running the
``on_failure`` step, which in our case prints out an error message.

Download the Code
-----------------

:download:`Lesson 6 - Complete code </code/tutorial_code/tutorials_06.zip>`

Up Next
-------
In the next lesson we'll see how to achieve a similar outcome using
custom navigation.

New Code - Complete
-------------------

**new_hire.sl**

.. code-block:: yaml

    namespace: tutorials.hiring

    imports:
      base: tutorials.base

    flow:
      name: new_hire

      inputs:
        - address

      workflow:
        - print_start:
            do:
              base.print:
                - text: "Starting new hire process"
            navigate:
              - SUCCESS: check_address
    
        - check_address:
            do:
              hiring.check_availability:
                - address
            publish:
              - availability: ${available}
            navigate:
              - UNAVAILABLE: print_fail
              - AVAILABLE: SUCCESS

        - print_finish:
            do:
              base.print:
                - text: "${'Availability for address ' + address + ' is: ' + availability}"
            navigate:
              - SUCCESS: SUCCESS

        - on_failure:
          - print_fail:
              do:
                base.print:
                  - text: "${'Failed to create address: ' + address}"
