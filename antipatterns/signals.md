% Signals
---
severity: 3
type: antipattern
---

Django has a sophisticated system to trigger certain logic when
you save, delete, change many-to-many relations, etc.

Very often people make use of such signals and for some edge-cases
these are indeed the only effective solution, but there are only a
few cases where using signals is appropriate.

# Why is it a problem?

One of the main problems with signals is that signals do *not* always run.
Indeed the `pre_save` and `post_save` signals will *not* run when we save
or update objects in bulk. For example if we create multiple `Post`s with:

<pre class="python"><code>Post.objects.<b>bulk_create(</b>[
    Post(title='foo'),
    Post(title='bar'),
    Post(title='qux')
]<b>)</b></code></pre>

then the signals do not run. The same happens when you update posts, for example
with:

<pre class="python"><code>Post.objects.all().<b>update(</b>views=0<b>)</b></code></pre>

Often people assume that signals *will* run in that case, and for example
perform calculations with the signals: they recalculate a certain field, based
on the updated values. Since one can update a field *without* triggering the
the corresponding signals, then this results in an inconsistent value.

If the signals run, for example when we call `.save()` on a model object, then
the triggers *will* run. Contrary to popular belief, signals do *not* run asynchronous,
but in a synchronous manner: there is a list of functions and these will all run.
A second problem is that these signals might raise an error, and this will thus
result in the function that triggered the views, raising that error.
Developers often do not take this into account.

If such error is raised, then eventually the `.save()` call will raise an error. Even if
the developer takes this into account, it is hard to anticipate on the consequences: if there
are multiple handlers for the same signal, then some of the handlers can have made changes
whereas others might not have been invoked. It thus makes it more complicated to repair
the object, since the handlers might already have changed the object partially.

It is also rather easy to get stuck in an infinite loop with signals. If we for example have a model of a
`Profile` with a signal that will remove the `User` if we remove the `Profile`:

<pre><code>from django.db import models
from django.db.models.signals import pre_delete
from django.dispatch import receiver

class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE
    )

# &hellip;

@receiver(pre_delete, sender=Profile)
def delete_profile(sender, instance, **kwargs):
    instance.user.delete()</code></pre>

If we now remove a `Profile`, this will get stuck in an infinite loop. Indeed, first we start
removing a `Profile`. This will trigger the signal to run, which will remove the related user object.
But Django will look what to do when removing the user, and it thus will *first* remove the `Profile` again
triggering the signal. It is easy to end up with infinite recursion when defining signals. Especially if we
use signals on two models that are related to each other.

The `pre_save` and `post_save` signals of an object run immediately before and after an object
is saved to the database. If we have a model with a `ManyToManyField`, then when we create that
object and the signals run, the `ManyToManyField` is *not* yet populated. This is because a `ModelForm`
first needs to create the object, before that object has a primary key and thus can start populating
the many-to-many relation. Every now and then people realize this when they want to run aggregates
on the many-to-many relation(s).

Even if only one handler is attached to the the signal, and that handler can never raise an error,
the handler still is often not an elegant solution. Another developer might not be aware of its existence,
since it has only a "weak" binding to th model, and thus it makes the effect of saving an object less
predictable.

Finally other programs can also make changes to the database, and thus will not trigger the signals,
and this eventually could lead to the database being in an inconsistent state.

# What can be done to resolve the problem?

Often it is better to avoid using signals. One can implement a lot of logic *without* signals.


# Extra tips