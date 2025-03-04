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
