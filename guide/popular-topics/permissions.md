## Introduction

Permissions are Discords primary feature enabling users to customize the workings of their server to their liking.
To break it down to essentials: Permissions and permission overwrites tell Discord who is allowed to do what and where.
When first confronted with them they can be quite confusing, but no worries we are here to take care of that, so let's dive in!

### Roles as bot permissions

If you want to keep your bots permission checks simple, you might find it sufficient to just check if the member executing the command has a certain role.

If you have the role ID, you can simply check if the `.roles` Collection on a GuildMember object includes it, using `.has()`. Should you not know the ID and want to check for something like a role named "Mod", you can use `.some()`.

```js
member.roles.has('role-id-here');
// returns true if the member has the role

member.roles.some(role => role.name === 'Mod');
// returns true if any of the member's roles is exactly named "Mod"
```

If you want to enhance this system slightly, you can include the guild owner by comparing the executing members ID with `message.guild.ownerID`. 

To include permission checks like `ADMINISTRATOR` or `MANAGE_GUILD`, keep reading as we will cover Discord Permissions and all their intricacies in the following sections.

### Terminology

* Permission: The ability to execute a certain action in Discord
* Overwrite: Rule on a channel to modify the permissions for a member or role
* Bit field: Binary representation of Discord permissions 
* Flag: Human readable string in MACRO_CASE, for example `'KICK_MEMBERS'`, refers to a position in the permission bit field. You can find a list of all valid flags in the [discord.js documentation](https://discord.js.org/#/docs/main/stable/class/Permissions?scrollTo=s-FLAGS)
* Base Permissions: Permissions for roles the member has, set on the guild level
* Final Permissions: Permissions for a member or role, after all overwrites are applied

<tip>You can provide permission decimals wherever we use flag literals in this guide. If you are interested in a handy permission calculator you can look at the "Bot" section in the [Discord developer portal](https://discordapp.com/developers/applications).</tip>

## Base permissions

### Setting role permissions

Base permissions are set on roles, not the guild member itself. To change them, you access a Role object (for example via `member.roles.first()` or `guild.roles.random()`) and use the `.setPermissions()` method. This is how you'd change the base permissions for the @everyone role, for example:

```js
guild.defaultRole.setPermissions(['SEND_MESSAGES', 'VIEW_CHANNEL']);
```

Any permission not referenced in the flag array or bit field are not granted to the role. 

<tip>Note that flag names are literal. Although `VIEW_CHANNEL` grants access to view multiple channels the permission flag is still called `VIEW_CHANNEL` in singular.</tip>

### Creating a role with permissions

Alternatively you can provide permissions as a property of [RoleData](https://discord.js.org/#/docs/main/stable/typedef/RoleData) objects during role creation as an array of flag strings or a permission number:

```js
guild.createRole({ name: 'Mod', permissions: ['MANAGE_MESSAGES', 'KICK_MEMBERS'] });
```

### Checking member permissions

To know if a one of a member's roles has a permission enabled, you can use the `.hasPermission()` method of the [GuildMember](https://discord.js.org/#/docs/main/stable/class/GuildMember) class and provide a permission flag, array, or number to check for. You can also specify if you want to allow the `ADMINISTRATOR` permission or the guild owner status to override this check with the following parameters.

```js
if (member.hasPermission('KICK_MEMBERS')) {
	console.log('This member can kick');
}

if (member.hasPermission(['KICK_MEMBERS', 'BAN_MEMBERS'])) {
	console.log('This member can kick and ban');
}

if (member.hasPermission('KICK_MEMBERS', false, false)) {
	console.log('This member can kick without allowing admin to override');
}

if (member.hasPermission('KICK_MEMBERS', false, false, false)) {
	console.log('This member can kick without allowing admin or owner to override');
}
```

If you provide multiple permissions to the method, it will only return `true` if all permissions you specified are granted.

## Channel overwrites

Permission overwrites control the abilities of members for this specific channel or a set of channels if applied to a category with synchronized child channels.

As you have likely already seen in your desktop client, channel overwrites have three states: 

- Explicit allow (`true`, green ✓)
- Explicit deny (`false`, red X) 
- Default (`null`, gray /)

### Adding overwrites

To add a permission overwrite for a role or guild member, you access the channel object and use the `.overwritePermissions()` method. The first parameter is the target of the overwrite, either a Role or GuildMember object (or its respective resolvable), and the second is a [PermissionOverwriteOptions](https://discord.js.org/#/docs/main/stable/typedef/PermissionOverwriteOptions) object.

Let's add an overwrite to lock everyone out of the channel. The guild ID doubles as the role id for the default role @everyone as demonstrated below:

```js
channel.overwritePermissions(channel.guild.defaultRole, { VIEW_CHANNEL: false });
```

Any permission flags not specified get neither an explicit allow nor deny overwrite and will use the base permission, unless another role has an explicit overwrite set.

You can also provide an array of overwrites during channel creation as shown below:

```js
guild.createChannel('new-channel', 'text', [
	{
		id: guild.defaultRole.id,
		deny: ['VIEW_CHANNEL'],
	},
	{
		id: user.id,
		allow: ['VIEW_CHANNEL'],
	},
]);
```

These objects are [ChannelCreationOverwrites](https://discord.js.org/#/docs/main/stable/typedef/ChannelCreationOverwrites) and differ from [PermissionOverwriteOptions](https://discord.js.org/#/docs/main/stable/typedef/PermissionOverwriteOptions); be careful to not mix them up!

<warning>On the master branch, the functionality of `GuildChannel#overwritePermissions` is changed to replacing all overwrites. `GuildChannel#updateOverwrite` is introduced to take its place in updating a single overwrite while keeping all others intact.</warning>

### Replacing overwrites

To replace all permission overwrites on the channel with a provided set of new overwrites, you can use the `.replacePermissionOverwrites()` function. This is extremely handy if you want to copy a channels full set of overwrites to another one, as this method allows passing an array or Collection of [PermissionOverwrites](https://discord.js.org/#/docs/main/stable/class/PermissionOverwrites) as well as [ChannelCreationOverwrites](https://discord.js.org/#/docs/main/stable/typedef/ChannelCreationOverwrites).

```js
// copying overwrites from another channel
channel.replacePermissionOverwrites({ overwrites: otherChannel.permissionOverwrites });

// replacing overwrites with PermissionOverwriteOptions
channel.replacePermissionOverwrites({
	overwrites: [
		{
			id: guild.defaultRole.id,
			deny: ['VIEW_CHANNEL'],
		},
		{
			id: user.id,
			allow: ['VIEW_CHANNEL'],
		},
	],
});
```

<warning>On the master branch, the functionality of `GuildChannel#overwritePermissions` is changed to replace overwrites</warning>

### Removing overwrites

To remove the overwrite for a specific member or role, you can get it from the channels permissionOverwrites Collection and call the `.delete()` method on it. Since the Collection is keyed by the target's ID (either role ID or user ID), the respective overwrite is very easy to access.

```js
// deleting the channels overwrite for the message author
channel.permissionOverwrites.get(message.author.id).delete();
```

### Syncing with a category

If the permission overwrites on a channel under a category match with the parent (category) the channel is considered to be synchronized. This means that any changes in the categories overwrites will now also change the channels overwrites. Changing the child channels overwrites will not effect the parent. 

To easily synchronize permissions with the parent channel you can call the `.lockPermissions()` method on the respective child channel.  

```js
if (!channel.parent) {
	return console.log('This channel is not listed under a category');
}

channel.lockPermissions()
	.then(() => console.log('Successfully synchronized permissions with parent channel'))
	.catch(console.error);
```

### Permissions after overwrites

discord.js features two utility methods to easily determine the final permissions for a guild member or role in a specific channel: `.permissionsFor()` on the [GuildChannel](https://discord.js.org/#/docs/main/stable/class/GuildChannel?scrollTo=permissionsFor) class and `.permissionsIn()` on the [GuildMember](https://discord.js.org/#/docs/main/stable/class/GuildMember?scrollTo=permissionsIn) class (note that the Role class does not yet feature a `.permissionsIn()` method on stable). Both return a [Permissions](https://discord.js.org/#/docs/main/stable/class/Permissions) object.

To check your bots permissions in the channel the command was used in, you could use something like this:

```js
// final permissions for a guild member using permissionsFor
const botPermissionsFor = channel.permissionsFor(guild.me);

// final permissions for a guild member using permissionsIn
const botPermissionsIn = guild.me.permissionsIn(channel);

// final permissions for a role
const rolePermissions = channel.permissionsFor(role);
```

<warning>The `.permissionsFor()` method returns a bit field with all permissions set if the member or role has the global `ADMINISTRATOR` permission and does not take overwrites into consideration in this case. Using the second parameter of the `.has()` method as described further down in the guide will not allow you to check without taking `ADMINISTRATOR` into account here!</warning>

If you want to know how to work with the returned Permissions objects keep reading as this will be our next topic.

## The Permissions object

The [Permissions](https://discord.js.org/#/docs/main/stable/class/Permissions) object is a discord.js class containing a permissions bit field and a bunch of utility methods to manipulate it easily.
Remember that using these methods will not manipulate permissions, but create a new instance representing the changed bit field.

### Converting permission numbers

Some methods and properties in Discord.js return permission decimals rather than a Permissions object, making it hard to manipulate or read them if you don't want to use bitwise operations.
You can, however, pass these decimals to the Permissions constructor to convert them.

```js
const { Permissions } = require('discord.js');

// constructing a Permissions Object from an array of PermissionResolvables
const flags = [
	'MANAGE_CHANNELS',
	'EMBED_LINKS',
	'ATTACH_FILES',
	'READ_MESSAGE_HISTORY',
	'MANAGE_ROLES',
];

const permissions = new Permissions(flags);
```

### Checking for permissions

The Permissions object features the `.has()` method, allowing an easy way to check flags in a Permissions bit field.
The `.has()` method takes two parameters: the first being either a permission number, single flag, or an array of permission numbers and flags, the second being a boolean, indicating if you want to allow the `ADMINISTRATOR` permission to override (defaults to true).

Let's say you want to know if the decimal bit field representation `268550160` has `MANAGE_CHANNELS` referenced:

```js
const { Permissions } = require('discord.js');

const permissions = new Permissions([
	'MANAGE_CHANNELS',
	'EMBED_LINKS',
	'ATTACH_FILES',
	'READ_MESSAGE_HISTORY',
	'MANAGE_ROLES',
]);

console.log(permissions.has('MANAGE_CHANNELS'));
// output: true

console.log(permissions.has(['MANAGE_CHANNELS', 'EMBED_LINKS']));
// output: true

console.log(permissions.has(['MANAGE_CHANNELS', 'KICK_MEMBERS']));
// output: false

const adminPermissions = new Permissions('ADMINISTRATOR');

console.log(adminPermissions.has('MANAGE_CHANNELS'));
// output: true

console.log(adminPermissions.has('MANAGE_CHANNELS'), true);
// output: true

console.log(adminPermissions.has('MANAGE_CHANNELS'), false);
// output: false
```

### Manipulating permissions

The Permissions object enables you to easily add or remove certain permissions from an existing bit field without having to worry about bitwise operations. Both `.add()` and `.remove()` can take a single permission flag or number, an array of permission flags or numbers, or multiple permission flags or numbers as multiple parameters.

```js
const { Permissions } = require('discord.js');

const permissions = new Permissions([
	'MANAGE_CHANNELS',
	'EMBED_LINKS',
	'ATTACH_FILES',
	'READ_MESSAGE_HISTORY',
	'MANAGE_ROLES',
]);

console.log(permissions.has('KICK_MEMBERS'));
// output: false

permissions.add('KICK_MEMBERS');
console.log(permissions.has('KICK_MEMBERS'));
// output: true

permissions.remove(2);
console.log(permissions.has('KICK_MEMBERS'));
// output : false
```

You can utilize these methods to adapt permissions or overwrites without touching the other flags. To achieve this you can get the existing permissions for a role, manipulating the bit field as described above and passing the changed bit field to `role.setPermissions()`.

<tip>In the stable branch, `role.permissions` returns a number which needs to be converted to a Permissions object for this to work as described here. We covered how to achieve this in the section "[Converting permission numbers to Objects](popular-topics/permissions?id=converting-permission-numbers)"</tip>

## Resulting code

If you want to compare your code to the code we've constructed so far, you can review it over on the GitHub repository [here](https://github.com/discordjs/guide/tree/master/code-samples/popular-topics/permissions/).
