# Permissions

A simple and performant Roles-Permissions system for Laravel.

```php
if ($user->role('editor', 'admin')->isGranted()) {
	return 'Only editors or admins can modify posts!';
} 

if ($user->permission('delete post')->isDenied()) {
    return 'Only admins can delete posts!';
}
```

## [Download it](https://github.com/sponsors/DarkGhostHunter/sponsorships?sponsor=DarkGhostHunter&tier_id=303801)

[![](sponsors.png)](https://github.com/sponsors/DarkGhostHunter/sponsorships?sponsor=DarkGhostHunter&tier_id=303801)

[Become a Sponsor and get instant access to this package](https://github.com/sponsors/DarkGhostHunter/sponsorships?sponsor=DarkGhostHunter&tier_id=303801).

## Requirements

* PHP 8.0 or later
* Laravel 9, 10 or later
* Cache compatible with [Atomic Locks](https://laravel.com/docs/10.x/cache#atomic-locks) (`file`, `redis`, `database`, `array`...)

## Installation

You can install the package via Composer. Open your `composer.json` and point the location of the private repository under the `repositories` key.

```json
{
    // ...
    "repositories": [
        {
            "type": "vcs",
            "name": "laragear/permissions",
            "url": "https://github.com/laragear/permissions.git"
        }
    ],
}
```

Then call Composer to retrieve the package.

```shell
composer require laragear/permissions
```

You will be prompted for a personal access token. If you don't have one, follow the instructions or [create one here](https://github.com/settings/tokens/new?scopes=repo). It takes just seconds.

> **Info**
> 
> You can find more information about in [this article](https://darkghosthunter.medium.com/php-use-your-private-repository-in-composer-without-ssh-keys-da9541439f59).

## How it works?

Roles, and their permissions, are loaded into the application at boot time, making them immutable but also faster to retrieve.

Users are attached to roles and permissions through a simple trait. While the authorization data is persisted in the database, it's pre-emptively cached and refreshed each time a change is done. This avoids multiple calls to the database, at the same time it keeps it fresh.

## Setup

To enable roles and permissions management in your app, we need to do three things: add the `permissions` table, add the `HasPermissions` to your User models, and finally add the roles and permissions.

### 1. Publishing assets

Fire the Artisan CLI to publish all the assets of Laragear Permissions. You will receive a migration file, a config file, and the stubs to create roles and permissions.

```shell
php artisan vendor:publish --provider="Laragear\Permissions\PermissionsServiceProvider"
```

### 2. Migrating

The `..._create_permissions_table.php` file in your `/database/migrations` folder creates a new table to hold the permission data of each user model. 

If you're using polymorphic relations or additional columns, this is a great opportunity to [change the migration configuration](#migration-configuration). Otherwise, just migrate your app like always.

```shell
php artisan migrate
```

### 3. Trait

Finally, ensure the User model uses the `HasPermissions` trait, or the `HasMorphPermissions` trait if you're using polymorphic relations on multiple models.

```php
namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Laragear\Permissions\HasPermissions;

class User extends Authenticatable
{
    use HasPermissions;

    // ...
}
```

## Usage

After you install Laragear Permissions, you will have one file called `roles/roles.php` in your application. Here you can define **roles**, each one with a set of named **permissions**.

Let's set an example and commence with an online shop, operated by 3 people: Melissa who processes orders, Pedro who manages the inventory, and myself who manages the finances.

To create a role we can use the convenient `Role` facade. After we set a name, we will use the `can()` method with an array of permissions related to the Role.

Let's make Roles for the Melissa, the _cashier_, and Pedro, the _inventory clerk_.

```php
use Laragear\Permissions\Facades\Role;

Role::name('cashier')->can([
	'see orders',
	'modify orders',
	'complete orders',
]);

Role::name('inventory clerk')->can([
	'manage inventory',
]);
```

For my own role, I need access to everything. Instead of writing all the permissions in one single array, we can create a new based on a mix of other roles' permissions, and add a new permission on top.

```php
use Laragear\Permissions\Facades\Role;

Role::name('cashier')->can([
    // ...
]);

Role::name('inventory clerk')->can([
    // ...
]);

// Create a new role based on the other two roles.
Role::name('manager')->basedOn('cashier', 'inventory clerk')->can('see finances');
```

If there is a permission that I don't need, I can use the `except()` method filter out the permissions specified.

```php
Role::name('manager')->from('cashier', 'inventory clerk')->can('see finances')->except('complete orders');
```

And that is the crash course on roles and permissions.

## Managing Roles and Permissions

### Attaching Roles

Once we have our Roles defined, the next step is attaching these Roles to a User. Use the `role()` method with the names of the roles, and then use `attach()` method.

```php
use App\Models\User;

$user = User::query()->where('name', 'Melissa')->first();

$user->role('cashier')->attach();
```

### Detaching Roles

Detaching a role is easy as using `role()` with the name of the Roles to detach, and the `detach()` method.

```php
use App\Models\User;

$user = User::query()->where('name', 'Melissa')->first();

$user->role('cashier')->detach();
```

### Granting permissions

> **Warning**
> 
> Attaching and detaching Roles is much better than assigning Permissions directly to a User, and is the recommended way to handle authorization. 

You may add dynamically single permissions to a user, regardless of the role they have. Simply use `grant()` on the permission you want to attach.

Let's allow Melissa to help Pedro by granting her permission to manage the inventory.

```php
use App\Models\User;

$user = User::query()->where('name', 'Melissa')->first();

$user->permission('manage inventory')->grant();
```

Explicit permissions take precedence over roles, and will remain attached even if a role with the same permissions is detached or attached.

For example, imagine we grant `manage inventory` to Pedro, when is `inventory clerk` already grants him. The permission will be attached anyway.

```php
use App\Models\User;

$user = User::query()->where('name', 'Pedro')->first();

$user->permission('manage inventory')->grant();
```

If we remove the `inventory clerk`, all the role permissions will be removed, but the `manage inventory` permission will remain, so we will need to detach it after.

```php
use App\Models\User;

$user = User::query()->where('name', 'Pedro')->first();

$user->permission('manage inventory')->detach();
```

### Denying permissions

> **Warning**
>
> Attaching and detaching Roles is much better than assigning Permissions directly to a User, and is the recommended way to handle authorization.

Sometimes you will need to _deny_ a permission to a user, even if the role they have already grants him that particular permission. Instead of creating a new role specifically for that user, you can just _deny_ that permission using `deny()`.

Let's deny Melissa's permission to manage the inventory, as Pedro already has done everything.

```php
use App\Models\User;

$user = User::query()->where('name', 'Melissa')->first();

$user->permission('manage inventory')->deny();
```

### Detaching permissions

When permissions are granted or denied explicitly for a given user, removing/adding a role won't modify these permissions, as explicit permissions always have precedence.

To remove an explicit permission, you can use `detach()` on the permission names.

```php
use App\Models\User;

$user = User::query()->where('name', 'Melissa')->first();

$user->permission('manage inventory')->detach();
```

Alternatively, you can remove all explicit permissions with `clearPermission()`.

```php
use App\Models\User;

$user = User::query()->where('name', 'Melissa')->first();

$user->permission('manage inventory')->clearPermissions();
```

## Checking permissions

Now that our users have roles and permissions, the next step is to check if a user has a permission. We can easily check that with `isGranted()` over the permission name.

```php
use App\Models\User;

$user = User::query()->where('name', 'Melissa')->first();

if ($user->permission('manage inventory')->isGranted()) {
    return view('inventory.all');
}

return 'You cannot manage the inventory';
```

You can do the inverse with `isDenied()`:

```php
if ($user->permission('manage inventory')->isDenied()) {
    return 'You cannot manage the inventory';
}
```

Finally, you may check for **all** issued permissions by just adding them as the arguments.  

```php
// If the user has permissions to create a product, and update a product...
if ($user->permission('update product', 'delete product')->isGranted()) {
    return view('product.show', $product);
}

// If the user doesn't have permission to create a product, and update a product...
if ($user->permission('create product', 'update product')->isDenied()) {
    return 'You cannot manage products';
}
```

### Checking Roles

Most of the time you will be checking Permissions directly instead of Roles, but some rare occasions you will need to do the latter, for example, to grant all-access to a given role. In these scenarios, use the `role()` method with `isGranted()` and `isDenied()` methods.

For example, if the User has the role of `manager`, it will be able to view the administration dashboard.

```php
use App\Models\User;

$user = User::query()->where('name', 'Melissa')->first();

if ($user->role('manager')->isGranted()) {
    return view('admin.dashboard');
}

return 'You are not the manager!';
``` 

### Listing all permissions

You can get all the computed (effective) permissions as a [Collection](https://laravel.com/docs/10.x/collections) for a given user using `listPermissions()`. 

```php
use App\Models\User;

$user = User::query()->where('name', 'Melissa')->first();

$permissions = $user->listPermissions();

// Collection([
//     'see orders',
//     'complete orders',
//     'modify orders',
//     'manage inventory',
//     'modify orders',
//     'manage inventory',
// ])
```

## Roles for Teams

You may have an application that serves multiple users that can be part of one or multiple teams. This package will allow you to manage roles and permissions _per team_ with only issuing the Eloquent Model of the team, or a string that identifies it like a UUID, name or even integer.

```php
use App\Models\User;

$user = User::query()->where('name', 'Melissa')->first();

$team = $user->teams()->find(64);

// Attach a role for a given team
$user->inTeam($team)->role('editor')->attach();

// Detach a role for a given team
$user->inTeam($team)->role('editor')->detach();

// Attach a permission for a given team. 
$user->inTeam($team)->permission('manage inventory')->grant();

// Detach a permission for a given team. 
$user->inTeam($team)->permission('manage inventory')->deny();

// Get the permissions for a given team
$user->inTeam($team)->listPermissions();
````

For checking permissions, use the `inTeam()` method of the target user model, using the same rules

```php
use App\Models\User;

$user = User::query()->where('name', 'Melissa')->first();

$team = $user->teams()->find(64);

if ($user->inTeam($team)->role('editor')->isDenied()) {
    return "You are not the editor of the team {$team->name}";
}
```

## Integrating with Laravel Authorization

You may want to integrate permissions and roles with [Laravel Gates and Policies](https://laravel.com/docs/10.x/authorization). You can easily do it as long you have access to the model that uses permissions.

For example, let's check if a User has permissions to see an order. 

```php
use Illuminate\Support\Facades\Gate;
use App\Models\User;
use App\Models\Order;

Gate::define('see-order', function (User $user, Order $order) {
    // If the user is the author of the order, approve.
    if ($order->author()->is($user)) {
        return true;
    }
    
    // If the user has the permissions to see all orders, approve.
    return $user->permission('see all orders')->isGranted();
});
```

Then later, in your code, you can use Laravel Authorization mechanisms like any day of the week.

```php
use Illuminate\Support\Facades\Gate;
use App\Models\Order;

public function order(Order $order)
{
    // Call the authorization gate.
    Gate::authorize('see-order', $order);
    
    return response()->view('order.show', ['order' => $order]);
}
```

## Deleting permissions

If you're deleting a User that uses Permissions, you may need to first delete their respective permissions rows to avoid leaving this data orphaned. You can wrap both operations in a [database transaction](https://laravel.com/docs/10.x/database#database-transactions) to be double sure.

```php
use App\Models\User;
use Illuminate\Support\Facades\DB;

DB::transaction(function () {
    $user = User::query()->where('name', 'Melissa')->firstOrFail();

    $user->permissions()->delete();
    $user->delete();
})
```

You can avoid this by linking the foreign column in the migration with `cascadeOnDelete()`, although this will only work on non-morph relations. Just simply add set the table name it will be linked, and the column name if it's not `id` already.

For example, we can link the permissions to the `users` table:

```php
use Illuminate\Database\Migrations\Migration;

return new class extends Migration
{
    protected array $cascadeOnDelete = [
        'table' => 'users', 
        'column' => 'id'
    ];
    
    // ...
}
```

## Querying the Permissions directly

You can still query the permissions relation directly in the database using the `Permissions` Eloquent Model.

For example, we may query how many users with the role `editor` are present in our application:

```php
use Laragear\Permissions\Models\Permission;

$count = Permission::query()->whereJsonContains('roles->editor')->count();

return "We have $count editors in our site";
```

You may also use the `permissions()` Eloquent relation to retrieve all the permissions for each team for the user, or the first if you're not using teams.

For example, we can use the query to retrieve all granted and denied permissions.

```php
use App\Models\User;

$user = User::find(1);

/** @var \Laragear\Permissions\Models\Permission $permissions */
$permissions = $user->permissions()->first();

$granted = $permissions->permissions_granted;
$denied = $permissions->permissions_denied;
```

> **Warning**
> 
> It's not recommended to make changes directly on database won't propagate to the cache, so you may want to [flush the cache](https://laravel.com/docs/10.x/cache#removing-items-from-the-cache) if you make any change as last resort.

### Retrieving users from the `Permission` model

The `Permission` model doesn't have any set relation to point to the user, because all access is done through your User model. You can use [Dynamic Relationships](https://laravel.com/docs/10.x/eloquent-relationships#dynamic-relationships) to set the inverse relation and get the users.

The column name that links the target user with the Permission row is called `authorizable_id`, or plus `authorizable_type` if you're using morph relations. These are constants defined in the `Permission` model.

```php
use Laragear\Permissions\Models\Permission;
use App\Models\Customer;
 
Permission::resolveRelationUsing(
    'customer', fn ($model) => $model->belongsTo(Customer::class, Permission::QUALIFIED_KEY_NAME);
});
```

## Configuration

First publish the configuration file:

```shell
php artisan vendor:publish --provider="Laragear\Permissions\PermissionsServiceProvider" --tag="config"
```

You will receive the `config/permissions.php` file with the following contents:

```php
return [
    'cache' => [
        'store' => env('PERMISSIONS_CACHE_STORE'),
        'prefix' => 'permissions',
        'ttl' => 60 * 60,
        'lock' => 5
    ],
];
```

### Cache

```php
return [
    'cache' => [
        'store' => env('PERMISSIONS_CACHE_STORE'),
        'prefix' => 'permissions',
        'ttl' => 60 * 60,
        'lock' => 3
    ],
];
```

The permissions are saved into the cache to avoid querying the database every time this data every time its requested. This control which cache store is used and how.

By default, if the store is `null` or empty, it will use the application default, which is `file` out of the box. The `prefix` controls the string prefix to store the authorization data. The `ttl` key is how much it should be kept on the cache, being `null` forever.

The `lock` manages how much time the lock should be acquired, and waited for. All authorization operations are atomic, which avoids data races when they happen.

## Migration Configuration

The migration file you should have in your migrations folder contains three properties to easily change the relation.

```php
use Illuminate\Database\Migrations\Migration;

return new class extends Migration
{
    protected bool $polymorphic = false;
    protected string $primaryKeys = 'int';
    protected array $cascadeOnDelete = ['table' => null, 'column' => 'id'];
    
    // ...
}
```

The `$polymorphic` property determines if the `permissions` table should be linked to multiple models using a polymorphic column. The `$primaryKeys` sets the common primary key types for all the relations.

When not using a polymorphic relation, the `$cascadeOnDelete` allows to create a foreign constraint to the user's table, so when a user is deleted the permission data is also deleted. You only need to add the table name, but if it's empty, it won't create it.

Additionally, you have the `createAdditionalColumns()` method to create additional columns in the table if you need to.

```php
use Illuminate\Database\Schema\Blueprint;

protected function createAdditionalColumns(Blueprint $table): void
{
    $table->boolean('has_all_access')->default(false);
}
``` 

## Laravel Octane compatibility

- The only singletons registered are the Registrar and Repository. These are meant to persist as-is across multiple requests. 
- There are no singletons using a stale config instance.
- There are no singletons using a stale request instance.
- There are no static properties written during a request.

There should be no problems using this package with Laravel Octane.

## Security

If you discover any security related issues, please email darkghosthunter@gmail.com instead of using the issue tracker.

# License

This specific package version is licensed under the terms of the [MIT License](LICENSE.md), at time of publishing.

[Laravel](https://laravel.com) is a Trademark of [Taylor Otwell](https://github.com/TaylorOtwell/). Copyright © 2011-2023 Laravel LLC.
