---
title: Module structure
description: How MediaSpace modules are structured
---

A MediaSpace module is an implementation of MVC web application, based on Zend-Framework folder structure and naming convention.

The following is the folder structure of a MediaSpace module including typical files in each folder:

```
{module}/
  assets/            # contains static files, e.g., JavaScript/CSS
  controllers/
    IndexController.php
  models/
    {Module}.php     # module model class - required
  views/
    scripts/
      index/         # contains view scripts for `IndexController`
        index.phtml  # view script for `indexAction`
        other.phtml  # view script for `otherAction`
  admin.ini          # module configuration for KMS admins
  default.ini        # default module configuration values
  module.info        # contains user-friendly information about the module
```

Note: In the example above, `{module}` represents the lowercase version of the name of the module, where `{Module}` represents the module's name, but only with the first letter in uppercase.
For example, given a module name called `myawesomemodule` (My Awesome Module):

-   `{module}` would be `myawesomemodule`
-   `{Module}` would be`Myawesomemodule`

## Auto-loading naming conventions

MediaSpace is using Zend Framework 1 as its infrastructure. This means that every file name and class name must adhere to ZF1's conventions.

Here are the most common patterns:

### Controllers

-   All of the module's controllers are in the `controllers` directory
-   File name: `[Name]Controller.php`, where `[Name]` is the name of the controller and the first letter is uppercase
-   Each controller must extend `Kms_Module_Controller_Abstract`
-   Each action method within a controller must end with `Action`, e.g., `getEntryAction` (which translates to `get-entry` route)

### Models

The model in a MediaSpace module is responsible for "declaring" each and every feature the module extends, and it also hosts some (if not all) the business logic of the module.

Extending MediaSpace through a module is done by implementing one of the many interfaces that are available in MediaSpace.

A basic model class of a module called `mymodule` module would look like the following:

```php
class Mymodule_Model_Mymodule extends Kms_Module_BaseModel
{
}
```

The abstract class `Kms_Module_BaseModel` implements 2 interfaces:

-   `Kms_Interface_Model_ViewHook`: allows modules to provide their HTML output to be included in core views of MediaSpace.
-   `Kms_Interface_Access`: every module that has a controller must declare the access rules for its actions to integrate with MediaSpace's roles.

#### View hooks

View hooks allows modules to insert HTML into some key areas in KMS layout.
Note: as KMS is now transitioning to a more unified experience with an implementation of a design system (often referenced as DS), some of the view hooks might not be in used there, but we're working on providing alternatives.

The following view hooks are available:

-   `CORE_VIEW_HOOK_MODULES_HEADER`: rendered in the `<head>` part of the page. Useful for loading static assets (JavaScript, CSS), HTML tags that are required to be in `<head>`, e.g., `<meta>` tags.
-   `CORE_VIEW_HOOK_ENTRY_PAGE`: allows modules to render a replacement to the default player. If your module implements the "entry type" interface (see below) and your module doesn't handle regular video entries, then you would probably use this view hook.
-   `CORE_VIEW_HOOK_EDIT_ENTRY_PLAYER`: same as `CORE_VIEW_HOOK_ENTRY_PAGE`, but in the edit entry page.

#### Access Rules

Each module action requires you to declare its access rules, i.e., the minimal KMS role that is allowed to access this action. This is done by implementing the method `getAccessRules()`.
Available KMS roles:

-   `EMPTY_ROLE` (`emptyRole`) - non-authenticated users
-   `ANON_ROLE` (`anonymousRole`) - if the KMS configuration allows users to browse the application anonymously (i.e., `auth->allowAnonymous` configuration is "Yes"), this would be the role of non-authenticated users.
-   `VIEWER_ROLE` (`viewerRole`) - used mainly for users that are browsing the application. These users can't have media of their own.
-   `PRIVATE_ROLE` (`privateOnlyRole`)
-   `ADMIN_ROLE` (`adminRole`)
-   `UNMOD_ROLE` (`unmoderatedAdminRole`) - same as `adminRole`, but media uploaded by users with this role are automatically approved (when global moderation is enabled for the account).
-   `PARTNER_ROLE` (`partnerRole`) - this role is used in the KMS Administrator interface, which is available to users that have `isAdmin = 1` flag on their user objects. Normally those users can access the KMC (Kaltura Management Console), which is a separate application for managing content in the account.
    Note: using this role for an action will require you to specify (using an interface, `Kms_Interface_AdminAuthStorage`) that the action is using "admin storage", which is an admin session (you can logged in to the KMS portal and the admin applications at the same time, hence the separation).

Here's an example implementation:

```php
class Mymodule_Model_Mymodule implements Kms_Interface_AdminAuthStorage
{
  public const MODULE_NAME = 'mymodule';

  public function getAccessRules()
  {
    return [
      [
        'controller' => self::MODULE_NAME . ':index',
        'actions' => ['get-entry'],
        'role' => Kms_Plugin_Access::PRIVATE_ROLE,
      ],
      [
        'controller' => self::MODULE_NAME . ':index',
        'actions' => ['approve-entry'],
        'role' => Kms_Plugin_Access::ADMIN_ROLE,
      ],
      [
        'controller' => self::MODULE_NAME . ':admin',
        'actions' => ['manage-settings'],
        // note: for this to work, it is required to implement an interface - `Kms_Interface_AdminAuthStorage`
        'role' => Kms_Plugin_Access::PARTNER_ROLE,
      ],
      // ... other rules ...
    ];
  }

  /**
  * This will cause all actions in the `AdminController` to use admin auth storage
  */
  public function shouldUseAdminAuthStorage(Zend_Controller_Request_Abstract $request)
  {
    return $request->getControllerName() === 'admin' && $request->getModuleName === self::MODULE_NAME;
  }
}
```
