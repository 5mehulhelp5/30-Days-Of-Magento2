# Data & Schema Patches in Magento 2

Patches are the modern way to handle data manipulation and schema modifications in Magento 2. They replace the old `InstallData`, `UpgradeData`, `InstallSchema`, and `UpgradeSchema` scripts.

## Table of Contents

1. [Overview](#overview)
2. [Types of Patches](#types-of-patches)
3. [Data Patches](#data-patches)
4. [Schema Patches](#schema-patches)
5. [Patch Dependencies](#patch-dependencies)
6. [Revertable Patches](#revertable-patches)
7. [Best Practices](#best-practices)

## Overview

Patches are classes that contain data or schema modification instructions. They are executed once and tracked in the `patch_list` database table.

### Key Benefits

- **Version-independent**: No need to track setup versions
- **Dependency management**: Control execution order
- **Revertability**: Optionally make patches revertable
- **Clear purpose**: Each patch has a single, clear responsibility

## Types of Patches

### 1. Data Patches

Located in: `Setup/Patch/Data/`
Used for: Inserting, updating, or deleting data

### 2. Schema Patches

Located in: `Setup/Patch/Schema/`
Used for: Complex schema operations that can't be done via declarative schema

> **Note**: For most schema operations, use `db_schema.xml`. Schema patches are for complex scenarios only.

## Data Patches

### Basic Data Patch Structure

**`Setup/Patch/Data/AddDefaultCategories.php`**

```php
<?php
namespace Vendor\Module\Setup\Patch\Data;

use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;

class AddDefaultCategories implements DataPatchInterface
{
    /**
     * @var ModuleDataSetupInterface
     */
    private $moduleDataSetup;

    /**
     * @param ModuleDataSetupInterface $moduleDataSetup
     */
    public function __construct(
        ModuleDataSetupInterface $moduleDataSetup
    ) {
        $this->moduleDataSetup = $moduleDataSetup;
    }

    /**
     * Apply patch
     *
     * @return void
     */
    public function apply()
    {
        $this->moduleDataSetup->getConnection()->startSetup();

        // Your data manipulation logic here
        $tableName = $this->moduleDataSetup->getTable('vendor_categories');

        $data = [
            ['name' => 'Electronics', 'status' => 1],
            ['name' => 'Clothing', 'status' => 1],
            ['name' => 'Books', 'status' => 1]
        ];

        $this->moduleDataSetup->getConnection()->insertMultiple($tableName, $data);

        $this->moduleDataSetup->getConnection()->endSetup();
    }

    /**
     * Get array of patches that have to be executed prior to this
     *
     * @return string[]
     */
    public static function getDependencies()
    {
        return [];
    }

    /**
     * Get aliases (previous names) for the patch
     *
     * @return string[]
     */
    public function getAliases()
    {
        return [];
    }
}
```

### Data Patch with Model Operations

**`Setup/Patch/Data/CreateAdminUser.php`**

```php
<?php
namespace Vendor\Module\Setup\Patch\Data;

use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\User\Model\UserFactory;
use Magento\Authorization\Model\RoleFactory;
use Magento\Authorization\Model\RulesFactory;

class CreateAdminUser implements DataPatchInterface
{
    /**
     * @var UserFactory
     */
    private $userFactory;

    /**
     * @var RoleFactory
     */
    private $roleFactory;

    /**
     * @var RulesFactory
     */
    private $rulesFactory;

    /**
     * @param UserFactory $userFactory
     * @param RoleFactory $roleFactory
     * @param RulesFactory $rulesFactory
     */
    public function __construct(
        UserFactory $userFactory,
        RoleFactory $roleFactory,
        RulesFactory $rulesFactory
    ) {
        $this->userFactory = $userFactory;
        $this->roleFactory = $roleFactory;
        $this->rulesFactory = $rulesFactory;
    }

    /**
     * Apply patch
     *
     * @return void
     */
    public function apply()
    {
        // Create admin role
        $role = $this->roleFactory->create();
        $role->setName('Custom Admin Role')
            ->setPid(0)
            ->setRoleType('G')
            ->setUserId(0);
        $role->save();

        // Set role permissions
        $resource = ['Magento_Backend::all'];
        $this->rulesFactory->create()
            ->setRoleId($role->getId())
            ->setResources($resource)
            ->saveRel();

        // Create admin user
        $user = $this->userFactory->create();
        $user->setData([
            'username'  => 'customadmin',
            'firstname' => 'Custom',
            'lastname'  => 'Admin',
            'email'     => 'admin@example.com',
            'password'  => 'Admin@123',
            'is_active' => 1
        ]);
        $user->setRoleId($role->getId());
        $user->save();
    }

    /**
     * Get dependencies
     *
     * @return string[]
     */
    public static function getDependencies()
    {
        return [];
    }

    /**
     * Get aliases
     *
     * @return string[]
     */
    public function getAliases()
    {
        return [];
    }
}
```

### Data Patch with CMS Block/Page

**`Setup/Patch/Data/CreateCmsBlock.php`**

```php
<?php
namespace Vendor\Module\Setup\Patch\Data;

use Magento\Cms\Model\BlockFactory;
use Magento\Framework\Setup\Patch\DataPatchInterface;

class CreateCmsBlock implements DataPatchInterface
{
    /**
     * @var BlockFactory
     */
    private $blockFactory;

    /**
     * @param BlockFactory $blockFactory
     */
    public function __construct(BlockFactory $blockFactory)
    {
        $this->blockFactory = $blockFactory;
    }

    /**
     * Apply patch
     *
     * @return void
     */
    public function apply()
    {
        $cmsBlockData = [
            'title' => 'Custom Promotional Block',
            'identifier' => 'custom_promo_block',
            'content' => '<div class="promo">Special Offer!</div>',
            'is_active' => 1,
            'stores' => [0], // All store views
        ];

        $this->blockFactory->create()->setData($cmsBlockData)->save();
    }

    /**
     * Get dependencies
     *
     * @return string[]
     */
    public static function getDependencies()
    {
        return [];
    }

    /**
     * Get aliases
     *
     * @return string[]
     */
    public function getAliases()
    {
        return [];
    }
}
```

### Data Patch with EAV Attributes

**`Setup/Patch/Data/AddProductAttribute.php`**

```php
<?php
namespace Vendor\Module\Setup\Patch\Data;

use Magento\Eav\Setup\EavSetupFactory;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;

class AddProductAttribute implements DataPatchInterface
{
    /**
     * @var ModuleDataSetupInterface
     */
    private $moduleDataSetup;

    /**
     * @var EavSetupFactory
     */
    private $eavSetupFactory;

    /**
     * @param ModuleDataSetupInterface $moduleDataSetup
     * @param EavSetupFactory $eavSetupFactory
     */
    public function __construct(
        ModuleDataSetupInterface $moduleDataSetup,
        EavSetupFactory $eavSetupFactory
    ) {
        $this->moduleDataSetup = $moduleDataSetup;
        $this->eavSetupFactory = $eavSetupFactory;
    }

    /**
     * Apply patch
     *
     * @return void
     */
    public function apply()
    {
        $eavSetup = $this->eavSetupFactory->create(['setup' => $this->moduleDataSetup]);

        $eavSetup->addAttribute(
            \Magento\Catalog\Model\Product::ENTITY,
            'custom_warranty',
            [
                'type' => 'varchar',
                'label' => 'Warranty Period',
                'input' => 'select',
                'source' => \Vendor\Module\Model\Product\Attribute\Source\Warranty::class,
                'required' => false,
                'sort_order' => 100,
                'global' => \Magento\Eav\Model\Entity\Attribute\ScopedAttributeInterface::SCOPE_GLOBAL,
                'group' => 'General',
                'is_used_in_grid' => true,
                'is_visible_in_grid' => false,
                'is_filterable_in_grid' => true,
                'visible' => true,
                'is_html_allowed_on_front' => false,
                'visible_on_front' => true
            ]
        );
    }

    /**
     * Get dependencies
     *
     * @return string[]
     */
    public static function getDependencies()
    {
        return [];
    }

    /**
     * Get aliases
     *
     * @return string[]
     */
    public function getAliases()
    {
        return [];
    }
}
```

## Schema Patches

Schema patches are used for complex schema operations that can't be done via `db_schema.xml`.

### Basic Schema Patch

**`Setup/Patch/Schema/AddCustomIndex.php`**

```php
<?php
namespace Vendor\Module\Setup\Patch\Schema;

use Magento\Framework\Setup\Patch\SchemaPatchInterface;
use Magento\Framework\Setup\SchemaSetupInterface;

class AddCustomIndex implements SchemaPatchInterface
{
    /**
     * @var SchemaSetupInterface
     */
    private $schemaSetup;

    /**
     * @param SchemaSetupInterface $schemaSetup
     */
    public function __construct(SchemaSetupInterface $schemaSetup)
    {
        $this->schemaSetup = $schemaSetup;
    }

    /**
     * Apply patch
     *
     * @return void
     */
    public function apply()
    {
        $this->schemaSetup->startSetup();

        $tableName = $this->schemaSetup->getTable('vendor_custom_table');
        $connection = $this->schemaSetup->getConnection();

        // Add composite index
        $connection->addIndex(
            $tableName,
            $this->schemaSetup->getIdxName($tableName, ['column1', 'column2']),
            ['column1', 'column2']
        );

        $this->schemaSetup->endSetup();
    }

    /**
     * Get dependencies
     *
     * @return string[]
     */
    public static function getDependencies()
    {
        return [];
    }

    /**
     * Get aliases
     *
     * @return string[]
     */
    public function getAliases()
    {
        return [];
    }
}
```

## Patch Dependencies

Control execution order by specifying dependencies:

**`Setup/Patch/Data/UpdateCategoryData.php`**

```php
<?php
namespace Vendor\Module\Setup\Patch\Data;

use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;

class UpdateCategoryData implements DataPatchInterface
{
    private $moduleDataSetup;

    public function __construct(ModuleDataSetupInterface $moduleDataSetup)
    {
        $this->moduleDataSetup = $moduleDataSetup;
    }

    public function apply()
    {
        // Update logic that depends on categories being created first
        $tableName = $this->moduleDataSetup->getTable('vendor_categories');

        $this->moduleDataSetup->getConnection()->update(
            $tableName,
            ['featured' => 1],
            ['name = ?' => 'Electronics']
        );
    }

    /**
     * This patch depends on AddDefaultCategories being executed first
     *
     * @return string[]
     */
    public static function getDependencies()
    {
        return [
            \Vendor\Module\Setup\Patch\Data\AddDefaultCategories::class
        ];
    }

    public function getAliases()
    {
        return [];
    }
}
```

## Revertable Patches

Make patches revertable by implementing `PatchRevertableInterface`:

**`Setup/Patch/Data/AddTestData.php`**

```php
<?php
namespace Vendor\Module\Setup\Patch\Data;

use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Framework\Setup\Patch\PatchRevertableInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;

class AddTestData implements DataPatchInterface, PatchRevertableInterface
{
    private $moduleDataSetup;

    public function __construct(ModuleDataSetupInterface $moduleDataSetup)
    {
        $this->moduleDataSetup = $moduleDataSetup;
    }

    /**
     * Apply patch
     */
    public function apply()
    {
        $tableName = $this->moduleDataSetup->getTable('vendor_test_data');

        $data = [
            ['name' => 'Test 1', 'value' => 100],
            ['name' => 'Test 2', 'value' => 200]
        ];

        $this->moduleDataSetup->getConnection()->insertMultiple($tableName, $data);
    }

    /**
     * Revert patch
     */
    public function revert()
    {
        $tableName = $this->moduleDataSetup->getTable('vendor_test_data');

        $this->moduleDataSetup->getConnection()->delete(
            $tableName,
            ['name IN (?)' => ['Test 1', 'Test 2']]
        );
    }

    public static function getDependencies()
    {
        return [];
    }

    public function getAliases()
    {
        return [];
    }
}
```

To revert a patch:

```bash
bin/magento module:uninstall Vendor_Module
```

## Migration from Legacy Setup Scripts

### Old Way (InstallData/UpgradeData)

```php
// Setup/InstallData.php
class InstallData implements InstallDataInterface
{
    public function install(ModuleDataSetupInterface $setup, ModuleContextInterface $context)
    {
        // Installation logic
    }
}

// Setup/UpgradeData.php
class UpgradeData implements UpgradeDataInterface
{
    public function upgrade(ModuleDataSetupInterface $setup, ModuleContextInterface $context)
    {
        if (version_compare($context->getVersion(), '1.0.1', '<')) {
            // Upgrade logic for 1.0.1
        }
        if (version_compare($context->getVersion(), '1.0.2', '<')) {
            // Upgrade logic for 1.0.2
        }
    }
}
```

### New Way (Data Patches)

```php
// Setup/Patch/Data/InitialData.php
class InitialData implements DataPatchInterface
{
    public function apply()
    {
        // Installation logic
    }
}

// Setup/Patch/Data/AddNewFeatureData.php
class AddNewFeatureData implements DataPatchInterface
{
    public function apply()
    {
        // Upgrade logic (no version check needed)
    }

    public static function getDependencies()
    {
        return [InitialData::class]; // Specify order
    }
}
```

## Best Practices

### 1. Single Responsibility

Each patch should do one thing:

```php
// Good: Separate patches
- AddDefaultCategories.php
- AddDefaultProducts.php
- ConfigureShipping.php

// Bad: One large patch
- InitializeStore.php (does everything)
```

### 2. Naming Conventions

Use descriptive names that indicate what the patch does:

```php
// Good names
- AddCustomerLoyaltyAttribute.php
- MigrateOldOrderStatuses.php
- UpdateProductPrices.php

// Bad names
- Patch1.php
- DataFix.php
- Update.php
```

### 3. Error Handling

```php
public function apply()
{
    try {
        $this->moduleDataSetup->getConnection()->startSetup();

        // Your logic here

        $this->moduleDataSetup->getConnection()->endSetup();
    } catch (\Exception $e) {
        // Log error
        throw new \Exception('Patch failed: ' . $e->getMessage());
    }
}
```

### 4. Idempotency

Make patches safe to run multiple times:

```php
public function apply()
{
    $connection = $this->moduleDataSetup->getConnection();
    $tableName = $this->moduleDataSetup->getTable('vendor_items');

    // Check if data already exists
    $select = $connection->select()
        ->from($tableName)
        ->where('identifier = ?', 'default_item');

    if (!$connection->fetchOne($select)) {
        // Only insert if not exists
        $connection->insert($tableName, [
            'identifier' => 'default_item',
            'name' => 'Default Item'
        ]);
    }
}
```

### 5. Transactions

Use transactions for data integrity:

```php
public function apply()
{
    $connection = $this->moduleDataSetup->getConnection();
    $connection->startSetup();

    try {
        $connection->beginTransaction();

        // Multiple operations
        $connection->insert($table1, $data1);
        $connection->insert($table2, $data2);

        $connection->commit();
    } catch (\Exception $e) {
        $connection->rollBack();
        throw $e;
    }

    $connection->endSetup();
}
```

## Useful Commands

```bash
# Apply all pending patches
bin/magento setup:upgrade

# Check patch status
bin/magento setup:db:status

# Dry run to see what would be executed
bin/magento setup:upgrade --dry-run

# Uninstall module and revert revertable patches
bin/magento module:uninstall Vendor_Module
```

## Checking Patch Execution

Patches are tracked in the `patch_list` table:

```sql
SELECT * FROM patch_list WHERE patch_name LIKE '%Vendor_Module%';
```

## Summary

| Feature | Description |
|---------|-------------|
| **Data Patches** | Handle data manipulation (insert, update, delete) |
| **Schema Patches** | Handle complex schema operations |
| **Dependencies** | Control execution order with `getDependencies()` |
| **Revertability** | Make patches reversible with `PatchRevertableInterface` |
| **Tracking** | Automatic tracking in `patch_list` table |
| **Version-free** | No need to manage setup versions |

## Resources

- [Magento DevDocs: Patches](https://developer.adobe.com/commerce/php/development/components/patches/)
- [Magento DevDocs: Data Patches](https://developer.adobe.com/commerce/php/development/components/patches/data/)
- [Magento DevDocs: Schema Patches](https://developer.adobe.com/commerce/php/development/components/patches/schema/)
