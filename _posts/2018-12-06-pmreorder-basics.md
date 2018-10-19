---
title: Pmreorder basics
author: wlemkows
layout: post
identifier: pmreorder-basics
---


### Introduction


When you write your persistent memory application the good practice is
to check it under pmemcheck - the tool which we have already described in two
blog posts some time ago. Feel free to come back to them and remind details
about [basics usage] [pmemcheck1Link] and [transaction][pmemcheck2Link].

In this post, I am going to introduce another tool for persistence correctness check.
As you already know, pmemcheck verify if all stores are made persistent in a proper
manner, pmreorder does not replace this functionality but provide extended one.
Pmreorder traverse sequences of stores between flush-fence barriers made by the application.
As the next step tool repeats those memory operations many times in different combinations
to simulate possible order of stores on the NVDIMM.

Each combination of stores is then verified by user-defined consistency checker and
any inconsistencies are logged. This process allows detecting not persistent data
on system failure even if pmemcheck does't detect any issue.

It may seem complicated so I will try to explain it using a simple example.


### Example

Let's assume that we have a simple list of three elements.
Code of the list insert function can look like this:

{% highlight C linenos %}
static void
list_insert(struct list_root *root, node_id node, int value)
{
	struct list_node *new = NODE_PTR(root, node);

	new->next = root->head;
	pmem_persist(&new->next, sizeof(node));

	root->head = node;
	pmem_persist(&root->head, sizeof(root->head));

	new->value = value;
	pmem_persist(&new->value, sizeof(value));
}
{% endhighlight %}

Full implementation and explanation you can find in [pmdk repository][pmreorderExampleLink].

At first, the next value in the node is set and persisted.
Then node is linked at the beginning of the list, its value is set
and both persisted separately.

When you run this code under pmemcheck you will get the following result:
 
{% highlight sh %}
==22161== Number of stores nod made persistent: 0
==22161== ERROR SUMMARY: 0 errors
{% endhighlight %}

It means that all stores are persisted properly, as we expected.

Now it is time to check the list under pmreorder tool.
To do that you need to consider when your list is consistent
and on that basics write consistency checker function.

Follow the next chapters to get to know more details about that.


### Checker

Consistency checker is just a binary/function that defines
conditions necessary to fulfill your consistency assumptions.
It should return 0 if the state is consistent or 1 if it isn't.

In case of our list with three elements:

{% highlight C linenos %}
 // params: root, node_id, value
list_insert(r, 5, 55);
list_insert(r, 3, 33);
list_insert(r, 6, 66);
{% endhighlight %}

we would like to be sure that if node (5, 3, 6) is linked to the list
then its value is set properly (55, 33, 66).
If the node will be linked to the list but its value field will have
an improper number then you should recognize this situation as inconsistent
and return 1.

For example:
{% highlight C linenos %}
static int
check_consistency(struct list_root *root)
{
	struct list_node *node1 = NODE_PTR(root, root->head);
	struct list_node *node2 = NODE_PTR(root, node1->next);
	struct list_node *node3 = NODE_PTR(root, node2->next);

	if ((root->head == 6 && node1->value != 66) || \
		(node1->next == 3 && node2->value != 33) || \
		(node2->next == 5 && node3->value != 55))
		return 1;

	return 0;
}
{% endhighlight %}

That's it. List and consistency checker are ready.
The next step is to generate a log of memory operations
made by the application. Such a log we will call store log.


### Store log

To get the list of all memory operations we are using pmemcheck logging
functionality. To turn on logging use `log-stores` parameter:

{% highlight sh %}
$ valgrind --tool=pmemcheck --log-stores=yes --log-file=store_log.log <your_app> [opt]
{% endhighlight %}

An output of the above command is a text file with the list of stores, flushes, fences,
user-defined markers and possibly some other data that can be consumed by pmreorder tool.

Let's look at part of such a store log, and read comments
next to the lines of the log to distinguish stores:

{% highlight sh %}
STORE;0x5600060;0x0;0x8|	# set new->next for node_id 5
FLUSH;0x5600040;0x40|		# persist next for node_id 5
FENCE|
STORE;0x5600000;0x5;0x8|	# set root->head for node_id 5
FLUSH;0x5600000;0x40|		# persist head for node_id 5
FENCE|
STORE;0x5600058;0x37;0x4|	# set new->value for node_id 5
FLUSH;0x5600040;0x40|		# persist value for node_id 5
FENCE|
STORE;0x5600040;0x5;0x8|	# set new->next for node_id 3
FLUSH;0x5600040;0x40|		# persist  next for nod_id 3
FENCE|
STORE;0x5600000;0x3;0x8|	# set root->head for node_id 3
FLUSH;0x5600000;0x40|		# persist head for node_id 3
FENCE|
STORE;0x5600038;0x21;0x4|	# set new->value for node_id 3
FLUSH;0x5600000;0x40|		# persist value for node_id 3
FENCE|
STORE;0x5600070;0x3;0x8|	# set new->next for node_id 6
FLUSH;0x5600040;0x40|		# persist next for node_id 6
FENCE|
STORE;0x5600000;0x6;0x8|	# set root->head for node_id 6
FLUSH;0x5600000;0x40|		# persist head for node_id 6
FENCE|
STORE;0x5600068;0x42;0x4|	# set new->value for node_id 6
FLUSH;0x5600040;0x40|		# persist value for node_id 6
FENCE|
STOP
==32611== Number of stores not made persistent: 0
==32611== ERROR SUMMARY: 0 errors
{% endhighlight %}

It often happends that store log contains multiple stores and operations
made on the pool that are irrelevant for your tests.
There is a mechanism called markers that allows ignoreing selected set of stores
during consistency check. For more information about this one and others features
look at [pmreorder man page][pmreorderManLink].

As you probably guess, store log generation was the last step before
pmreorder execution, so let's move on.


### Pmreorder

This probably will be the most important paragraph in this blog post,
so let me start from a brief description of how pmreorder works.

Pmreorder parses store log provided by the user and reproduce on
empty pool sequences of stores between barriers.
If tool run in the flush-fence barrier then execute the consistency
checker. If consistency checker returns 1 then logs inconsistent
stores to the output log file.

This procedure is repeated multiple times with different
order of stores. After each check pmreorder reverts already
tested sequence and takes the next combination to check.

How many combinations are tested?
It depends on the engine used. There are a few available engines types.
For example, full reorder engine generates all possible combination
without repetition, a random engine generates a specified number of randomly
generated combinations, no reorder engine just pass through stores without
reordering.

In this example I am going to use reverse accumulative engine which checks
correctness on a reverted growing subset of the original sequence.

{% highlight sh %}
Example:
        input: (a, b, c)
        output:
               ()
               (c)
               (c, b)
               (c, b, a)
{% endhighlight %}

For more examples and information about engines look at [pmreorder man page][pmreorderManLink].

Note that pmreorder is written in python, so you have to install
python3 on your machine to run the tool.

Using that information we can run pmreorder in this way:

{% highlight sh %}
$ python3 pmreorder.py
	-l store_log.log  			# file generated by pmemcheck

	-o output_file.log			# output from pmreorder

	-x pmem_memset_persist=NoReorderNoCheck	# do not check stores from pmem_memset_persist
						# available only in PMREORDER_EMIT_LOG=1
						# envirenment variable is set

	-r ReorderReverseAccumulative		# default engine

	-p "pmreorder_list c"			# checker binary with parameters
{% endhighlight %}

Take a look at result of this command:

{% highlight sh %}
$ cat output_file.log

WARNING:pmreorder:File /tmp/test_ex_pmreorder1/testfile inconsistent
WARNING:pmreorder:Call trace:
Store [0]:
     by  No trace available

WARNING:pmreorder:File /tmp/test_ex_pmreorder1/testfile inconsistent
WARNING:pmreorder:Call trace:
{% endhighlight %}

In this case output_file.log is not empty,
this means that some issues were detected.

We can see that two cases that cause inconsistency,
but there is no trace available.

To get more information about the issue we should
add additional flags to pmemcheck during store log
generation:

{% highlight sh %}
--log-stores-stacktraces=yes
--log-stores-stacktraces-depth=2
{% endhighlight %}

Afterwards output log is more readable:

{% highlight sh %}
WARNING:pmreorder:File /tmp/test_ex_pmreorder1/testfile inconsistent
WARNING:pmreorder:Call trace:
Store [0]:
    by  0x400C81: list_insert_inconsistent (pmreorder_list.c:133)
    by  0x400E56: main (pmreorder_list.c:176)

WARNING:pmreorder:File /tmp/test_ex_pmreorder1/testfile inconsistent
WARNING:pmreorder:Call trace:
{% endhighlight %}

Now we are able to check specific line in the code where inconsistency occured.
Empty call trace means that before any store state was also unsettled.

Where we made a mistake and how to fix it?
Look once again at list insert and checker code.

If the node is linked to the list then we are checking
if the value of the node is set.
But in our case, persist of the list head.
is located before node value persist.

So it can happen that the node is in the list but its value is not set yet.
It means an issue occurs because of the order of operations.

To fix that we have to change their order.
Also worth noting that we do not need to persist the value
and next field separately. It can be done by one pmem_persist:

{% highlight C linenos %}
static void
list_insert_consistent(struct list_root *root, node_id node, int value)
{
	struct list_node *new = NODE_PTR(root, node);

	new->value = value;
	new->next = root->head;
	pmem_persist(new, sizeof(new)); /* persist the node */

	root->head = node;
	pmem_persist(&root->head, sizeof(root->head)); /* add to the list */
}
{% endhighlight %}

This insert is constructed properly and consistency check will pass.

I hope that now you have enough knowledge to check your program
with the use of pmreorder tool.
Good luck!

[pmemcheck1Link]: http://pmem.io/2015/07/17/pmemcheck-basic.html
[pmemcheck2Link]: http://pmem.io/2015/07/20/pmemcheck-transactions.html
[pmreorderExampleLink]: https://github.com/pmem/pmdk/tree/master/src/examples/pmreorder
[pmreorderManLink]: http://pmem.io/pmdk/manpages/linux/master/pmreorder/pmreorder.1.html
