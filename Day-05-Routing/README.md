# Day 5: Routing in Magento 2

A comprehensive guide to understanding and implementing routing in Magento 2.

## Table of Contents

1. [Introduction](#introduction)
2. [How Routing Works](#how-routing-works)
3. [Route Configuration](#route-configuration)
4. [Controllers](#controllers)
5. [URL Structure](#url-structure)
6. [Route Parameters](#route-parameters)
7. [URL Rewriting](#url-rewriting)
8. [Custom Routers](#custom-routers)
9. [Practical Examples](#practical-examples)
10. [Best Practices](#best-practices)

---

## Introduction

### What is Routing?

Routing in Magento 2 is the mechanism that maps **URLs to controller actions**. When a user visits a URL, Magento's routing system:

1. Parses the URL
2. Identifies the appropriate router
3. Finds the matching route
4. Executes the corresponding controller action

### Why is Routing Important?

- **URL Structure** - Creates clean, SEO-friendly URLs
- **Module Organization** - Maps URLs to specific modules
- **Request Handling** - Directs requests to appropriate controllers
- **Customization** - Allows overriding and extending default behavior

### Route Types

Magento 2 has different routers for different areas:

| Router | Area | ID | Purpose |
|--------|------|-----|---------|
| **Standard Router** | Frontend | `standard` | Public-facing pages |
| **Admin Router** | Backend | `admin` | Admin panel pages |
| **URL Rewrite Router** | Both | N/A | SEO-friendly URLs |
| **Default Router** | Both | N/A | CMS pages, 404 handling |

---

## How Routing Works

### Routing Flow

```
User Request → FrontController → Router Collection → Match Router → Execute Controller
```

**Step-by-Step Process**:

1. **Request received** - User visits URL like `/blog/post/view/id/5`
2. **Front Controller** - `Magento\Framework\App\FrontController` receives request
3. **Router Collection** - Loops through registered routers
4. **Router Matching** - Each router tries to match the URL
5. **Route Found** - Matching router processes the request
6. **Controller Execution** - Controller action executes
7. **Response** - Result is returned to user

### URL Format

Standard Magento 2 URL structure:

```
https://example.com/frontName/controllerName/actionName/param1/value1/param2/value2
```

**Example**:
```
https://example.com/catalog/product/view/id/123
                      ↓        ↓       ↓    ↓   ↓
                  frontName controller action key value
```

### Router Priority

Routers are executed in a specific order:

1. **Custom Routers** (highest priority)
2. **Admin Router** - `admin` routes
3. **Standard Router** - `standard` routes
4. **URL Rewrite Router** - SEO URLs
5. **CMS Router** - CMS pages
6. **Default Router** - 404 handling (lowest priority)

---

## Route Configuration

### Creating routes.xml

Routes are defined in `routes.xml` files located in your module's `etc` directory.

#### Frontend Routes

**File**: `app/code/Vendor/Module/etc/frontend/routes.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:App/etc/routes.xsd">
    <router id="standard">
        <route id="vendor_module" frontName="blog">
            <module name="Vendor_Module" />
        </route>
    </router>
</config>
```

**URL Result**: `https://example.com/blog/...`

#### Admin Routes

**File**: `app/code/Vendor/Module/etc/adminhtml/routes.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:App/etc/routes.xsd">
    <router id="admin">
        <route id="vendor_module" frontName="blog">
            <module name="Vendor_Module" />
        </route>
    </router>
</config>
```

**URL Result**: `https://example.com/admin/blog/...`

### Configuration Attributes

| Attribute | Description | Example |
|-----------|-------------|---------|
| `id` | Unique route identifier | `vendor_module` |
| `frontName` | URL segment | `blog` |
| `module` | Module name | `Vendor_Module` |
| `before` | Load before specified module | `before="Magento_Customer"` |
| `after` | Load after specified module | `after="Magento_Cms"` |

### Multiple Modules for Same Route

```xml
<router id="standard">
    <route id="catalog" frontName="catalog">
        <module name="Magento_Catalog" />
        <module name="Vendor_CatalogExtension" before="Magento_Catalog" />
    </route>
</router>
```

This allows `Vendor_CatalogExtension` to override `Magento_Catalog` controllers.

---

## Controllers

### Controller Structure

Controllers in Magento 2 follow a specific directory structure:

```
Vendor/Module/
└── Controller/
    ├── Index/
    │   └── Index.php          → /frontName/index/index
    ├── Post/
    │   ├── View.php           → /frontName/post/view
    │   ├── Edit.php           → /frontName/post/edit
    │   └── Save.php           → /frontName/post/save
    └── Category/
        └── List.php           → /frontName/category/list
```

### Creating a Controller

#### Basic Controller

**File**: `app/code/Vendor/Module/Controller/Index/Index.php`

```php
<?php

namespace Vendor\Module\Controller\Index;

use Magento\Framework\App\Action\Action;
use Magento\Framework\App\Action\Context;
use Magento\Framework\View\Result\PageFactory;

class Index extends Action
{
    protected $resultPageFactory;

    public function __construct(
        Context $context,
        PageFactory $resultPageFactory
    ) {
        parent::__construct($context);
        $this->resultPageFactory = $resultPageFactory;
    }

    public function execute()
    {
        // Return a page result
        return $this->resultPageFactory->create();
    }
}
```

**URL**: `https://example.com/blog/index/index` or `https://example.com/blog`

### Controller Types

#### 1. Page Result (Full Page)

```php
public function execute()
{
    $resultPage = $this->resultPageFactory->create();
    $resultPage->getConfig()->getTitle()->set(__('Blog Posts'));
    return $resultPage;
}
```

#### 2. JSON Result

```php
protected $resultJsonFactory;

public function __construct(
    Context $context,
    \Magento\Framework\Controller\Result\JsonFactory $resultJsonFactory
) {
    parent::__construct($context);
    $this->resultJsonFactory = $resultJsonFactory;
}

public function execute()
{
    $result = $this->resultJsonFactory->create();
    return $result->setData(['success' => true, 'message' => 'Data saved']);
}
```

#### 3. Redirect Result

```php
protected $resultRedirectFactory;

public function execute()
{
    $resultRedirect = $this->resultRedirectFactory->create();
    $resultRedirect->setPath('*/*/index'); // Redirect to index action
    return $resultRedirect;
}
```

#### 4. Forward Result

```php
protected $resultForwardFactory;

public function __construct(
    Context $context,
    \Magento\Framework\Controller\Result\ForwardFactory $resultForwardFactory
) {
    parent::__construct($context);
    $this->resultForwardFactory = $resultForwardFactory;
}

public function execute()
{
    $resultForward = $this->resultForwardFactory->create();
    return $resultForward->forward('noroute'); // Forward to 404
}
```

#### 5. Raw Result (Plain Text)

```php
protected $resultRawFactory;

public function __construct(
    Context $context,
    \Magento\Framework\Controller\Result\RawFactory $resultRawFactory
) {
    parent::__construct($context);
    $this->resultRawFactory = $resultRawFactory;
}

public function execute()
{
    $result = $this->resultRawFactory->create();
    $result->setContents('Plain text response');
    return $result;
}
```

### Admin Controllers

Admin controllers extend `\Magento\Backend\App\Action` and include ACL checking.

**File**: `app/code/Vendor/Module/Controller/Adminhtml/Post/Index.php`

```php
<?php

namespace Vendor\Module\Controller\Adminhtml\Post;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Magento\Framework\View\Result\PageFactory;

class Index extends Action
{
    const ADMIN_RESOURCE = 'Vendor_Module::posts';

    protected $resultPageFactory;

    public function __construct(
        Context $context,
        PageFactory $resultPageFactory
    ) {
        parent::__construct($context);
        $this->resultPageFactory = $resultPageFactory;
    }

    public function execute()
    {
        $resultPage = $this->resultPageFactory->create();
        $resultPage->setActiveMenu('Vendor_Module::posts');
        $resultPage->getConfig()->getTitle()->prepend(__('Manage Posts'));
        return $resultPage;
    }
}
```

---

## URL Structure

### Standard URL Format

```
https://example.com/[frontName]/[controllerName]/[actionName]/[param1]/[value1]
```

### URL Components

**Example**: `https://example.com/blog/post/view/id/5/page/2`

| Component | Value | Source |
|-----------|-------|--------|
| Base URL | `https://example.com` | Store configuration |
| Front Name | `blog` | `routes.xml` |
| Controller | `post` | Directory name |
| Action | `view` | Class name |
| Parameters | `id=5, page=2` | URL params |

### Default Actions

If controller or action is omitted, Magento uses `index`:

```
/blog              → /blog/index/index
/blog/post         → /blog/post/index
/blog/post/view    → /blog/post/view
```

---

## Route Parameters

### Getting Parameters

#### In Controllers

```php
public function execute()
{
    // Get single parameter
    $id = $this->getRequest()->getParam('id');

    // Get with default value
    $page = $this->getRequest()->getParam('page', 1);

    // Get all parameters
    $params = $this->getRequest()->getParams();

    // Get POST data
    $postData = $this->getRequest()->getPostValue();

    // Check request method
    if ($this->getRequest()->isPost()) {
        // Handle POST request
    }
}
```

### Parameter Validation

```php
public function execute()
{
    $id = $this->getRequest()->getParam('id');

    // Validate numeric
    if (!is_numeric($id)) {
        $this->messageManager->addErrorMessage(__('Invalid ID'));
        return $this->resultRedirectFactory->create()->setPath('*/*/');
    }

    // Cast to integer
    $id = (int) $id;

    // Validate positive number
    if ($id <= 0) {
        throw new \Magento\Framework\Exception\LocalizedException(
            __('Invalid product ID')
        );
    }
}
```

### Building URLs with Parameters

#### In PHP

```php
// Using URL Builder
$url = $this->_url->getUrl('blog/post/view', ['id' => 5, 'category' => 'tech']);
// Result: /blog/post/view/id/5/category/tech

// Using path only
$this->resultRedirectFactory->create()->setPath(
    'blog/post/view',
    ['id' => 5]
);
```

#### In Templates

```php
// In .phtml files
$url = $block->getUrl('blog/post/view', ['id' => 5]);

<!-- Or using escapeUrl -->
<a href="<?= $block->escapeUrl($block->getUrl('blog/post/view', ['id' => 5])) ?>">
    View Post
</a>
```

#### In Layout XML

```xml
<block class="Magento\Framework\View\Element\Html\Link">
    <arguments>
        <argument name="path" xsi:type="string">blog/post/view</argument>
        <argument name="label" xsi:type="string" translate="true">View Posts</argument>
    </arguments>
</block>
```

---

## URL Rewriting

### What are URL Rewrites?

URL rewrites create SEO-friendly URLs that map to standard Magento routes.

**Before**: `/catalog/product/view/id/123`
**After**: `/my-awesome-product.html`

### Creating URL Rewrites Programmatically

```php
<?php

namespace Vendor\Module\Model;

use Magento\UrlRewrite\Model\UrlRewriteFactory;
use Magento\UrlRewrite\Model\ResourceModel\UrlRewrite;

class UrlRewriteCreator
{
    protected $urlRewriteFactory;
    protected $urlRewriteResource;

    public function __construct(
        UrlRewriteFactory $urlRewriteFactory,
        UrlRewrite $urlRewriteResource
    ) {
        $this->urlRewriteFactory = $urlRewriteFactory;
        $this->urlRewriteResource = $urlRewriteResource;
    }

    public function createRewrite($postId)
    {
        $urlRewrite = $this->urlRewriteFactory->create();

        $urlRewrite->setEntityType('custom')
            ->setEntityId($postId)
            ->setRequestPath('blog/my-post-slug')
            ->setTargetPath('blog/post/view/id/' . $postId)
            ->setRedirectType(0)
            ->setStoreId(1)
            ->setIsAutogenerated(0);

        $this->urlRewriteResource->save($urlRewrite);
    }
}
```

### URL Rewrite Types

| Redirect Type | Code | Description |
|---------------|------|-------------|
| No Redirect | 0 | Direct rewrite (no 301/302) |
| Permanent | 301 | Permanent redirect |
| Temporary | 302 | Temporary redirect |

---

## Custom Routers

### When to Use Custom Routers

Create custom routers when you need:
- Complex URL patterns
- Dynamic routing logic
- Integration with external systems
- Non-standard URL structures

### Creating a Custom Router

#### Step 1: Create Router Class

**File**: `app/code/Vendor/Module/Controller/Router.php`

```php
<?php

namespace Vendor\Module\Controller;

use Magento\Framework\App\ActionFactory;
use Magento\Framework\App\ActionInterface;
use Magento\Framework\App\RequestInterface;
use Magento\Framework\App\RouterInterface;

class Router implements RouterInterface
{
    protected $actionFactory;

    public function __construct(ActionFactory $actionFactory)
    {
        $this->actionFactory = $actionFactory;
    }

    public function match(RequestInterface $request): ?ActionInterface
    {
        $identifier = trim($request->getPathInfo(), '/');

        // Check if this router should handle the request
        if (strpos($identifier, 'custom-route') !== 0) {
            return null;
        }

        // Parse custom URL pattern
        // Example: /custom-route/post-123
        if (preg_match('#^custom-route/post-(\d+)$#', $identifier, $matches)) {
            $postId = $matches[1];

            // Set module, controller, action
            $request->setModuleName('vendor_module')
                ->setControllerName('post')
                ->setActionName('view')
                ->setParam('id', $postId);

            return $this->actionFactory->create(
                \Magento\Framework\App\Action\Forward::class
            );
        }

        return null;
    }
}
```

#### Step 2: Register Router in di.xml

**File**: `app/code/Vendor/Module/etc/di.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Framework\App\RouterList">
        <arguments>
            <argument name="routerList" xsi:type="array">
                <item name="custom_router" xsi:type="array">
                    <item name="class" xsi:type="string">Vendor\Module\Controller\Router</item>
                    <item name="disable" xsi:type="boolean">false</item>
                    <item name="sortOrder" xsi:type="string">40</item>
                </item>
            </argument>
        </arguments>
    </type>
</config>
```

**Sort Order**:
- Lower numbers = higher priority
- Standard router: 30
- Custom routers: 20-40
- CMS router: 60

---

## Practical Examples

### Example 1: Blog Module with Routes

#### Module Structure

```
Vendor/BlogModule/
├── etc/
│   ├── module.xml
│   └── frontend/
│       └── routes.xml
├── Controller/
│   ├── Index/
│   │   └── Index.php
│   ├── Post/
│   │   ├── View.php
│   │   └── List.php
│   └── Category/
│       └── View.php
└── registration.php
```

#### routes.xml

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:App/etc/routes.xsd">
    <router id="standard">
        <route id="blog" frontName="blog">
            <module name="Vendor_BlogModule" />
        </route>
    </router>
</config>
```

#### Post View Controller

```php
<?php

namespace Vendor\BlogModule\Controller\Post;

use Magento\Framework\App\Action\Action;
use Magento\Framework\App\Action\Context;
use Magento\Framework\View\Result\PageFactory;
use Vendor\BlogModule\Model\PostFactory;

class View extends Action
{
    protected $resultPageFactory;
    protected $postFactory;

    public function __construct(
        Context $context,
        PageFactory $resultPageFactory,
        PostFactory $postFactory
    ) {
        parent::__construct($context);
        $this->resultPageFactory = $resultPageFactory;
        $this->postFactory = $postFactory;
    }

    public function execute()
    {
        $postId = (int) $this->getRequest()->getParam('id');

        if (!$postId) {
            $this->messageManager->addErrorMessage(__('Post ID is required'));
            return $this->resultRedirectFactory->create()->setPath('blog');
        }

        $post = $this->postFactory->create()->load($postId);

        if (!$post->getId()) {
            $this->messageManager->addErrorMessage(__('Post not found'));
            return $this->resultRedirectFactory->create()->setPath('blog');
        }

        $resultPage = $this->resultPageFactory->create();
        $resultPage->getConfig()->getTitle()->set($post->getTitle());

        return $resultPage;
    }
}
```

**URL**: `/blog/post/view/id/5`

### Example 2: AJAX Controller with JSON Response

```php
<?php

namespace Vendor\Module\Controller\Post;

use Magento\Framework\App\Action\Action;
use Magento\Framework\App\Action\Context;
use Magento\Framework\Controller\Result\JsonFactory;

class Save extends Action
{
    protected $jsonFactory;
    protected $postRepository;

    public function __construct(
        Context $context,
        JsonFactory $jsonFactory,
        \Vendor\Module\Api\PostRepositoryInterface $postRepository
    ) {
        parent::__construct($context);
        $this->jsonFactory = $jsonFactory;
        $this->postRepository = $postRepository;
    }

    public function execute()
    {
        $result = $this->jsonFactory->create();

        if (!$this->getRequest()->isAjax()) {
            return $result->setData([
                'success' => false,
                'message' => 'Invalid request'
            ]);
        }

        try {
            $data = $this->getRequest()->getPostValue();

            // Validate data
            if (empty($data['title'])) {
                throw new \Exception(__('Title is required'));
            }

            // Save post
            $post = $this->postRepository->save($data);

            return $result->setData([
                'success' => true,
                'message' => __('Post saved successfully'),
                'post_id' => $post->getId()
            ]);

        } catch (\Exception $e) {
            return $result->setData([
                'success' => false,
                'message' => $e->getMessage()
            ]);
        }
    }
}
```

### Example 3: Admin Controller with ACL

```php
<?php

namespace Vendor\Module\Controller\Adminhtml\Post;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Magento\Framework\View\Result\PageFactory;

class Edit extends Action
{
    const ADMIN_RESOURCE = 'Vendor_Module::post_save';

    protected $resultPageFactory;

    public function __construct(
        Context $context,
        PageFactory $resultPageFactory
    ) {
        parent::__construct($context);
        $this->resultPageFactory = $resultPageFactory;
    }

    protected function _isAllowed()
    {
        return $this->_authorization->isAllowed(self::ADMIN_RESOURCE);
    }

    public function execute()
    {
        $id = $this->getRequest()->getParam('id');

        $resultPage = $this->resultPageFactory->create();
        $resultPage->setActiveMenu('Vendor_Module::posts');

        if ($id) {
            $resultPage->getConfig()->getTitle()->prepend(__('Edit Post'));
        } else {
            $resultPage->getConfig()->getTitle()->prepend(__('New Post'));
        }

        return $resultPage;
    }
}
```

---

## Best Practices

### 1. Route Naming Conventions

✅ **Good**:
```xml
<route id="vendor_modulename" frontName="modulename">
```

❌ **Bad**:
```xml
<route id="mod" frontName="x">
```

### 2. Controller Organization

✅ **Good Structure**:
```
Controller/
├── Index/Index.php
├── Post/
│   ├── View.php
│   ├── Edit.php
│   └── Save.php
```

❌ **Bad Structure**:
```
Controller/
├── PostView.php
├── PostEdit.php
└── PostSave.php
```

### 3. Parameter Validation

✅ **Always Validate**:
```php
$id = (int) $this->getRequest()->getParam('id');
if ($id <= 0) {
    throw new \Magento\Framework\Exception\NoSuchEntityException();
}
```

❌ **Don't Trust Input**:
```php
$id = $this->getRequest()->getParam('id');
$model->load($id); // Dangerous!
```

### 4. Use Dependency Injection

✅ **Good**:
```php
public function __construct(
    Context $context,
    PostRepositoryInterface $postRepository
) {
    parent::__construct($context);
    $this->postRepository = $postRepository;
}
```

❌ **Bad**:
```php
public function execute()
{
    $post = $this->_objectManager->create('Vendor\Module\Model\Post');
}
```

### 5. Proper Response Types

Choose the right result type for your use case:

| Use Case | Result Type |
|----------|-------------|
| Full page render | `PageFactory` |
| AJAX/API response | `JsonFactory` |
| Redirect | `RedirectFactory` |
| Plain text | `RawFactory` |
| Forward to another action | `ForwardFactory` |

### 6. URL Generation

✅ **Use URL Builder**:
```php
$url = $this->_url->getUrl('blog/post/view', ['id' => $postId]);
```

❌ **Don't Hardcode**:
```php
$url = '/blog/post/view/id/' . $postId; // Bad!
```

### 7. Admin Routes Security

Always implement ACL checking:

```php
const ADMIN_RESOURCE = 'Vendor_Module::resource_name';

protected function _isAllowed()
{
    return $this->_authorization->isAllowed(self::ADMIN_RESOURCE);
}
```

### 8. SEO-Friendly URLs

For public-facing pages, use URL rewrites:

```php
// Instead of: /catalog/product/view/id/123
// Use: /my-product-name.html
```

---

## Common Issues and Solutions

### Issue 1: 404 Not Found

**Problem**: Route not working after creating routes.xml

**Solution**:
```bash
# Clear cache
bin/magento cache:clean
bin/magento cache:flush

# Recompile if needed
bin/magento setup:di:compile
```

### Issue 2: Controller Not Executing

**Problem**: Controller exists but doesn't execute

**Checklist**:
1. Check class namespace matches directory structure
2. Verify `execute()` method exists
3. Check routes.xml is in correct directory
4. Clear cache
5. Check file permissions

### Issue 3: Admin Route Access Denied

**Problem**: Admin controller returns 403

**Solution**:
```php
// Ensure ACL resource is defined in acl.xml
// And user role has permission
const ADMIN_RESOURCE = 'Vendor_Module::posts';
```

### Issue 4: Parameters Not Received

**Problem**: `getParam()` returns null

**Solution**:
```php
// Check URL format
// Correct: /blog/post/view/id/5
// Wrong: /blog/post/view?id=5 (use getQuery() for query strings)

// Use getParam with default
$id = $this->getRequest()->getParam('id', 0);
```

---

## Summary

### Key Takeaways

1. **Routes map URLs to controllers** - Define in `routes.xml`
2. **Controllers handle requests** - Follow naming conventions
3. **Parameters pass data** - Validate all input
4. **Multiple result types** - Choose appropriate response
5. **Custom routers for complex needs** - Implement `RouterInterface`
6. **URL rewrites for SEO** - Create friendly URLs
7. **Security matters** - Always validate and check permissions

### Quick Reference

**Create a route**:
```xml
<router id="standard">
    <route id="blog" frontName="blog">
        <module name="Vendor_Module" />
    </route>
</router>
```

**Create a controller**:
```php
namespace Vendor\Module\Controller\Post;
class View extends \Magento\Framework\App\Action\Action
{
    public function execute()
    {
        return $this->resultPageFactory->create();
    }
}
```

**Get URL**:
```php
$url = $this->_url->getUrl('blog/post/view', ['id' => 5]);
```

---

## Additional Resources

- [Magento DevDocs - Routing](https://devdocs.magento.com/guides/v2.4/extension-dev-guide/routing.html)
- [Magento DevDocs - Controllers](https://devdocs.magento.com/guides/v2.4/extension-dev-guide/controller.html)
- [Magento DevDocs - URL Rewrites](https://devdocs.magento.com/guides/v2.4/frontend-dev-guide/url-rewrites.html)

### Next Steps

- **Day 6**: Database Schema - Learn how to create and manage database tables
- **Day 7**: Models & Collections - Work with data models

---

**Congratulations!** You now understand Magento 2 routing and can create custom routes and controllers for your modules.
