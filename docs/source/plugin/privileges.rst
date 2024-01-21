===============
User privileges
===============

IRC users can have privileges in a **channel**, given by MODE messages such as:

.. code-block:: irc

    MODE #example +ov Nickname Nickname

This will give both OP and Voice privileges to the user named "Nickname" in the
"#example" channel (and only in this channel). When Sopel receives a MODE
message it registers and updates its knowledge of a user's privileges in a
channel, which can be used by plugins in various ways.

Historically, these two privilege levels ("op" and "voiced") were the only
channel privileges available:

* :attr:`~sopel.privileges.AccessLevel.OP`: channel operator, set and unset by
  modes ``+o`` and ``-o``
* :attr:`~sopel.privileges.AccessLevel.VOICE`: the privilege to send messages to
  a channel with the ``+m`` mode, set and unset by modes ``+v`` and ``-v``

.. _nonstandard privilege levels:

Over time, IRC servers and clients have adopted various combinations of
nonstandard, less formally defined privilege levels:

* :attr:`~sopel.privileges.AccessLevel.HALFOP`: intermediate level between VOICE
  and OP, set and unset by modes ``+h`` and ``-h``
* :attr:`~sopel.privileges.AccessLevel.ADMIN`: channel admin, above OP and below
  OWNER, set and unset by modes ``+a`` and ``-a``
* :attr:`~sopel.privileges.AccessLevel.OWNER`: channel owner, above ADMIN and OP,
  set and unset by modes ``+q`` and ``-q``

**It's important to note that not all IRC networks support these nonstandard
privileges, and the ones that do may not support all of them.** If you are
writing a plugin for public distribution, ensure your code behaves sensibly when
only the standardized ``+v`` (voice) and ``+o`` (op) modes exist.

Access rights
=============

Privileged users
----------------

A plugin can limit who can trigger its callables using the
:func:`~sopel.plugin.require_privilege` decorator::

    from sopel import plugin
    from sopel.privileges import AccessLevel

    @plugin.require_privilege(AccessLevel.OP)
    @plugin.require_chanmsg
    @plugin.command('chanopcommand')
    def chanop_command(bot, trigger):
        # only a channel operator can use this command

This way, only users with OP privileges or above in a channel can use the
command ``chanopcommand`` in that channel: other users will be ignored by the
bot. It is possible to tell these users why with the ``message`` parameter::

    @plugin.require_privilege(AccessLevel.OP, 'You need +o privileges.')

.. important::

    A command that requires channel privileges will always execute if
    called from a private message to the bot. You can use the
    :func:`sopel.plugin.require_chanmsg` decorator to ignore the command
    if it's called in PMs.

The bot is a user too
---------------------

Sometimes, you may want the bot to be a privileged user in a channel to allow a
command. For that, there is the :func:`~sopel.plugin.require_bot_privilege`
decorator::

    @plugin.require_bot_privilege(AccessLevel.OP)
    @plugin.require_chanmsg
    @plugin.command('opbotcommand')
    def change_topic(bot, trigger):
        # only if the bot has OP privileges

This way, this command cannot be used if the bot doesn't have the right
privileges in the channel where it is used, independent from the privileges of
the user who invokes the command.

As with ``require_privilege``, you can provide an error message::

    @plugin.require_bot_privilege(
        AccessLevel.OP, 'The bot needs +o privileges.')

And you **can** use both ``require_privilege`` and ``require_bot_privilege`` on
the same plugin callable::

    @plugin.require_privilege(AccessLevel.VOICE)
    @plugin.require_bot_privilege(AccessLevel.OP)
    @plugin.require_chanmsg
    @plugin.command('special')
    def special_command(bot, trigger):
        # only if the user has +v and the bot has +o (or above)

This way, you can allow a less privileged user to access a command for a more
privileged bot (this works for any combination of privileges).

.. important::

    A command that requires channel privileges will always execute if
    called from a private message to the bot. You can use the
    :func:`sopel.plugin.require_chanmsg` decorator to ignore the command
    if it's called in PMs.

Restrict to user account
------------------------

Sometimes, a command should be used only by users who are authenticated via
IRC services. On IRC networks that provide such information to IRC clients,
this is possible with the :func:`~sopel.plugin.require_account` decorator::

    @plugin.require_privilege(AccessLevel.VOICE)
    @plugin.require_account
    @plugin.require_chanmsg
    @plugin.command('danger')
    def dangerous_command(bot, trigger):
        # only if the user has +v and has a registered account

This has two consequences:

1. this command cannot be used by users who are not authenticated
2. this command cannot be used on an IRC network that doesn't allow
   authentication or doesn't expose that information

It makes your plugin safer to use and prevents the possibility to use it on
insecure IRC networks.

.. seealso::

   `IRCv3 account-tracking specifications`__.

.. __: https://ircv3.net/irc/#account-tracking


Getting user privileges in a channel
====================================

Within a :term:`plugin callable`, you can get access to a user's privileges in
a channel to check privileges manually. For example, you could adapt the level
of information your callable provides based on said privileges.

First you need a user's nick and a channel (e.g. from the trigger parameter),
then you can get that user's privileges through the **channel**'s
:attr:`~sopel.tools.target.Channel.privileges` attribute::

    user_privileges = channel.privileges['Nickname']
    user_privileges = channel.privileges[trigger.nick]

You can check the user's privileges manually using bitwise operators. Here
for example, we check if the user is voiced (``+v``) or above::

    from sopel.privileges import AccessLevel

    if user_privileges & AccessLevel.VOICE:
        # user is voiced
    elif user_privileges > AccessLevel.VOICE:
        # not voiced, but higher privileges
        # like AccessLevel.HALFOP or AccessLevel.OP
    else:
        # no privilege

Another option is to use dedicated methods from the ``channel`` object::

    if channel.is_voiced('Nickname'):
        # user is voiced
    elif channel.has_privilege('Nickname', AccessLevel.VOICE):
        # not voiced, but higher privileges
        # like AccessLevel.HALFOP or AccessLevel.OP
    else:
        # no privilege

You can also iterate over the list of users and filter them by privileges::

    # get users with the OP privilege
    op_users = [
        user
        for nick, user in channel.users
        if channel.is_op(nick, AccessLevel.OP)
    ]

    # get users with OP privilege or above
    op_or_higher_users = [
        user
        for nick, user in channel.users
        if channel.has_privileges(nick, AccessLevel.OP)
    ]

.. seealso::

    Read about the :class:`~sopel.tools.target.Channel` and
    :class:`~sopel.tools.target.User` classes for more details.


sopel.privileges
================

.. automodule:: sopel.privileges
   :members:
   :member-order: bysource
