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

:::note
In the example above, `{module}` represents the lowercase version of the name of the module, where `{Module}` represents the module's name, but only with the first letter in uppercase.
For example, given a module name called `myawesomemodule` (My Awesome Module):

-   `{module}` would be `myawesomemodule`
-   `{Module}` would be`Myawesomemodule`

:::

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

## Configuration

### `admin.ini` (required)

`admin.ini` is a configuration file that allows module developers to expose configuration fields for MediaSpace administrators.

There are many types of config fields, but we'll note the most popular ones.
By default, even with an empty `admin.ini` file, the `enabled` boolean field is rendered.

The basic structure of a config field is:

```ini
fieldName.type = <type>
fieldName.comment = "This configuration does this and that. You can also add <i>HTML</i> tags if you'd like"
fieldName.validation = "Someclass::validationMethod"
```

##### Validation API

Here's an example of a validator implementation:

(Notice that if a config value does not pass the validation, you can invalidate other fields as well (if they're dependent on each other, for example))

```php
/** Validate
 * @param mixed $value
 * @param string $fieldName the validated field name
 * @param array $newConfig the new configuration object as an array
 * @return array{
 *     errorMessages: string[],
 *     invalidatedFields: string[],
 * }
 */
public static function validationMethod(mixed $value, string $fieldName, array $newConfig): array
{
  if (/** validation successful **/) {
    return ['errorMessages' => [], 'invalidatedFields' => []];
  }

  return [
    'errorMessages' => ['User-friendly error message'],
     'invalidatedFields' => ['fieldName', 'someOtherField'],
  ];
}
```

#### Text field

```ini
fieldName.type = "text"
fieldName.comment = "A simple text field"
```

#### Read-only field

A simple read-only text field.

This field type should be used when the module automatically deploys (see `deployment.ini` section below) UI Confs (i.e., player configuration objects), metadata profiles, conversion profiles, or when the module populates the field automatically when enabled.

<b>Note: administrators can still change this value using developer tools or by modifying the payload - i.e., there's no enforcement in the backend.
This is by design. This field type is mainly used to prevent changing sensitive values by mistake.</b>

Example:

```ini
fieldName.type = "readonly"
fieldName.comment = "A simple text field"
```

#### Boolean (Yes/No) field

Example:

```ini
fieldName.type = "boolean"
fieldName.comment = "Select 'Yes' to enable this feature, 'No' otherwise."
```

#### Select / multi-select

This field type renders a dropdown field (`<select>` element) for single value configs, or checkboxes for multiple values.

Example:

```ini
fieldName.type = "select"
fieldName.comment = "Select one (or more) of the choice(s)"

# option 1 - static list of choices
fieldName.labels.1 = "Choice #1"
fieldName.values.1 = "choice1"
fieldName.labels.2 = "Choice #2"
fieldName.values.2 = "choice2"

# option 2 - dynamic
fieldName.autoValues = "Mymodule_Model_Admin::getMultipleChoice" # reference a static method

# optional
fieldName.allowMulti = 1 # render checkboxes for multiple selection
```

#### Array

This field allows multiple values, each with its own set of sub-fields.

```ini
fieldName.type = "array"
fieldName.comment = "Select 'Yes' to enable this feature, 'No' otherwise."

# sub-fields
fieldName.fields.1.name = "field1"
fieldName.fields.1.type = "text"
fieldName.fields.1.comment = "Field1's field value"

fieldName.fields.1.name = "field2"
fieldName.fields.1.type = "select"
fieldName.fields.1.comment = "Select the field value from the dropdown"
```

#### Semi-hidden

This field allows setting configuration that is not visible in the UI, but is present in the DOM (and can be manipulated by using developer tools).

This field is useful for:

-   Feature flags that you wouldn't want to expose to administrators.
-   Configuration that you would like to phase out, but have to keep for backward compatibility.

Only simple field values are allowed.

Example:

```ini
fieldName.type = "semiHidden"
fieldName.comment = "Because this field is not rendered, this comment is only useful for developers or whoever can read this file"
```

### `default.ini` (required)

`default.ini` contains the default configuration of the module, before any save has taken place by the user.

It is recommended to populate as much default values as possible:

-   It gives the user an idea of what kind of value is expected.
-   It allows for easier initial set-up to get the user up and running quickly, driving usage.

### `deployment.ini`

`deployment.ini` is an (optional) configuration file that lets you deploy objects when the module is enabled.

The following objects are supported:

-   Metadata profiles
-   UI Conf objects (player configuration)
-   Conversion profiles

The basic structure looks like this:

```ini
[metadataProfiles]
profiles.profileName1.systemName = "CUSTOM_METADATA_SYSTEM_NAME"
profiles.profileName1.identifier = "someUniqueIdentifier" # this is used to map between the deployed metadata object to the config field (in `admin.ini`)
profiles.profileName1.name = "User-friendly metadata profile name"
profiles.profileName1.xsd_file = "relative/path/to/xml/schema.xml"
profiles.profileName1.objectType = "1" # See https://www.kaltura.com/api_v3/testmeDoc/enums/KalturaMetadataObjectType.html
profiles.profileName1.createMode = "3" # See https://www.kaltura.com/api_v3/testmeDoc/enums/KalturaMetadataProfileCreateMode.html

profiles.profileName2.systemName = "..."
# ... more ...

[players]
objectType = 1 # see https://www.kaltura.com/api_v3/testmeDoc/enums/KalturaUiConfObjType.html

# if you need to deploy players (UI Conf objects), the field prefix must be `widgets` (for historical reasons)
widgets.player1.usage = kalturaPlayerJs,player,ovp,tag1 # player tags
widgets.player1.identifier = "someOtherUniqueIdentifier"
widgets.player1.name = "A user-friendly name"
widgets.player1.config_file = "relative/path/to/config/file.json"
widgets.player1.conf_vars = "Player plugins string"
widgets.player1.width = 528
widgets.player1.height = 327

```

#### Mapping deployed objects to module configuration

In order to map the deployed objects to the module's configuration (in order to work with those objects as part of the module's functionality), the module must implement the `Kms_Interface_Deployable_Deployment` interface. Here's an example implementation:

```php
class Mymodule_Model_Mymodule extends Kms_Module_BaseModel implements Kms_Interface_Deployable_Deployment
{
  /**
  * @inheritDoc
  */
  public function getIdentifierToConfigMap()
  {
    return [
      'someUniqueIdentifier' => [
        'section' => self::MODULE_NAME,
        'key' => 'myModuleSpecificMetadataProfileId', // `myModuleSpecificMetadataProfileId` is a read-only field declared in `admin.ini`
      ],
    ];
  }
}
```
