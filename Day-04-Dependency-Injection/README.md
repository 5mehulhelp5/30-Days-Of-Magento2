# Day 04: Dependency Injection

## Overview

Dependency Injection (DI) is a design pattern and one of the most fundamental concepts in Magento 2. Understanding DI is crucial for writing clean, testable, and maintainable code.

## Table of Contents

1. [What is Dependency Injection?](#what-is-dependency-injection)
2. [Why Use Dependency Injection?](#why-use-dependency-injection)
3. [How DI Works in Magento 2](#how-di-works-in-magento-2)
4. [Object Manager](#object-manager)
5. [Constructor Injection](#constructor-injection)
6. [di.xml Configuration](#dixml-configuration)
7. [Preferences](#preferences)
8. [Virtual Types](#virtual-types)
9. [Plugins vs Preferences](#plugins-vs-preferences)
10. [Real-World Examples](#real-world-examples)
11. [Best Practices](#best-practices)

## What is Dependency Injection?

**Dependency Injection** is a design pattern where objects receive their dependencies from external sources rather than creating them internally.

### Without DI (Bad Practice)

```php
class OrderProcessor
{
    protected $emailSender;

    public function __construct()
    {
        // Creating dependency inside the class - TIGHT COUPLING
        $this->emailSender = new EmailSender();
    }

    public function processOrder($order)
    {
        // Process order logic
        $this->emailSender->send($order->getCustomerEmail(), 'Order Confirmation');
    }
}
```

**Problems:**
- Hard to test (can't mock EmailSender)
- Hard to replace EmailSender with different implementation
- Tight coupling between classes

### With DI (Good Practice)

```php
class OrderProcessor
{
    protected $emailSender;

    // Dependencies are INJECTED via constructor
    public function __construct(EmailSenderInterface $emailSender)
    {
        $this->emailSender = $emailSender;
    }

    public function processOrder($order)
    {
        // Process order logic
        $this->emailSender->send($order->getCustomerEmail(), 'Order Confirmation');
    }
}
```

**Benefits:**
- Easy to test (can inject mock EmailSender)
- Can swap implementations via di.xml
- Loose coupling - follows SOLID principles

## Why Use Dependency Injection?

### 1. Testability

```php
// Easy to write unit tests with DI
class OrderProcessorTest extends \PHPUnit\Framework\TestCase
{
    public function testProcessOrder()
    {
        // Create a mock email sender
        $mockEmailSender = $this->createMock(EmailSenderInterface::class);
        $mockEmailSender->expects($this->once())
            ->method('send');

        // Inject the mock
        $orderProcessor = new OrderProcessor($mockEmailSender);
        $orderProcessor->processOrder($order);
    }
}
```

### 2. Flexibility

Change implementations without modifying code:

```xml
<!-- di.xml -->
<preference for="EmailSenderInterface" type="SmtpEmailSender"/>
<!-- Switch to -->
<preference for="EmailSenderInterface" type="SendGridEmailSender"/>
```

### 3. Maintainability

- Changes to dependencies don't require changes to dependent classes
- Clear dependencies in constructor signature
- Single Responsibility Principle

### 4. Reusability

- Same class can be used with different dependencies
- Promotes composition over inheritance

## How DI Works in Magento 2

### DI Container Flow

```
┌─────────────────────────────────────────────┐
│  1. Class needs to be instantiated          │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│  2. Object Manager reads di.xml configs     │
│     - Preferences                           │
│     - Virtual Types                         │
│     - Arguments                             │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│  3. Analyzes constructor dependencies       │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│  4. Recursively creates dependencies        │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│  5. Applies plugins (interceptors)          │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│  6. Returns fully configured instance       │
└─────────────────────────────────────────────┘
```

## Object Manager

The **Object Manager** is Magento's implementation of a DI container. It's responsible for:
- Creating object instances
- Managing object lifecycle
- Resolving dependencies
- Applying preferences and virtual types

### Important Rules

❌ **NEVER use Object Manager directly in your code:**

```php
// BAD - Don't do this!
$objectManager = \Magento\Framework\App\ObjectManager::getInstance();
$product = $objectManager->create(\Magento\Catalog\Model\Product::class);
```

✅ **ALWAYS use constructor injection:**

```php
// GOOD - Do this!
class MyClass
{
    protected $productFactory;

    public function __construct(
        \Magento\Catalog\Model\ProductFactory $productFactory
    ) {
        $this->productFactory = $productFactory;
    }

    public function createProduct()
    {
        $product = $this->productFactory->create();
    }
}
```

### When Object Manager is Acceptable

Only in these specific cases:
1. **Backward compatibility** constraints
2. **Global scope** (integration tests, static blocks)
3. **Framework code** (not application code)

## Constructor Injection

Constructor injection is the primary method of dependency injection in Magento 2.

### Basic Example

```php
namespace Vendor\Module\Model;

use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Framework\App\Config\ScopeConfigInterface;
use Psr\Log\LoggerInterface;

class ProductManager
{
    /**
     * @var ProductRepositoryInterface
     */
    protected $productRepository;

    /**
     * @var ScopeConfigInterface
     */
    protected $scopeConfig;

    /**
     * @var LoggerInterface
     */
    protected $logger;

    /**
     * Constructor
     *
     * @param ProductRepositoryInterface $productRepository
     * @param ScopeConfigInterface $scopeConfig
     * @param LoggerInterface $logger
     */
    public function __construct(
        ProductRepositoryInterface $productRepository,
        ScopeConfigInterface $scopeConfig,
        LoggerInterface $logger
    ) {
        $this->productRepository = $productRepository;
        $this->scopeConfig = $scopeConfig;
        $this->logger = $logger;
    }

    /**
     * Get product by SKU
     *
     * @param string $sku
     * @return \Magento\Catalog\Api\Data\ProductInterface|null
     */
    public function getProductBySku($sku)
    {
        try {
            return $this->productRepository->get($sku);
        } catch (\Exception $e) {
            $this->logger->error('Failed to load product: ' . $e->getMessage());
            return null;
        }
    }
}
```

### Dependency Types

#### 1. Interface Dependencies (Preferred)

```php
public function __construct(
    ProductRepositoryInterface $productRepository  // Interface - GOOD
) {
    $this->productRepository = $productRepository;
}
```

#### 2. Concrete Class Dependencies

```php
public function __construct(
    \Magento\Catalog\Model\Product $product  // Concrete class - OK for specific cases
) {
    $this->product = $product;
}
```

#### 3. Factories

Used when you need to create NEW instances:

```php
public function __construct(
    \Magento\Catalog\Model\ProductFactory $productFactory
) {
    $this->productFactory = $productFactory;
}

public function createNewProduct()
{
    // Creates a NEW product instance
    $product = $this->productFactory->create();
    return $product;
}
```

#### 4. Arrays and Scalars

```php
public function __construct(
    array $data = [],
    $customValue = 'default'
) {
    $this->data = $data;
    $this->customValue = $customValue;
}
```

## di.xml Configuration

The `di.xml` file is where you configure the DI container.

### File Locations

```
app/code/Vendor/Module/etc/di.xml              # Global (all areas)
app/code/Vendor/Module/etc/frontend/di.xml     # Frontend only
app/code/Vendor/Module/etc/adminhtml/di.xml    # Admin only
app/code/Vendor/Module/etc/webapi_rest/di.xml  # REST API only
app/code/Vendor/Module/etc/webapi_soap/di.xml  # SOAP API only
```

### Basic di.xml Structure

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Preferences: Interface to Class mapping -->
    <preference for="InterfaceName" type="ClassName"/>

    <!-- Type configuration: Constructor arguments, plugins -->
    <type name="ClassName">
        <arguments>
            <argument name="paramName" xsi:type="type">value</argument>
        </arguments>
        <plugin name="pluginName" type="PluginClass"/>
    </type>

    <!-- Virtual types: Custom type configuration -->
    <virtualType name="VirtualTypeName" type="BaseClassName">
        <arguments>
            <argument name="paramName" xsi:type="type">value</argument>
        </arguments>
    </virtualType>

</config>
```

### Argument Types

```xml
<type name="Vendor\Module\Model\Example">
    <arguments>
        <!-- String -->
        <argument name="stringParam" xsi:type="string">Hello World</argument>

        <!-- Number -->
        <argument name="numberParam" xsi:type="number">123</argument>

        <!-- Boolean -->
        <argument name="boolParam" xsi:type="boolean">true</argument>

        <!-- Array -->
        <argument name="arrayParam" xsi:type="array">
            <item name="key1" xsi:type="string">value1</item>
            <item name="key2" xsi:type="string">value2</item>
        </argument>

        <!-- Object -->
        <argument name="objectParam" xsi:type="object">Vendor\Module\Model\CustomClass</argument>

        <!-- Null -->
        <argument name="nullParam" xsi:type="null"/>

        <!-- Constant -->
        <argument name="constParam" xsi:type="const">Vendor\Module\Model\Source\Options::VALUE</argument>

        <!-- Init parameter (from env.php or config.php) -->
        <argument name="initParam" xsi:type="init_parameter">Magento\Store\Model\StoreManager::PARAM_RUN_CODE</argument>
    </arguments>
</type>
```

## Preferences

Preferences map interfaces to concrete implementations.

### Example 1: Basic Preference

```xml
<!-- etc/di.xml -->
<preference for="Magento\Catalog\Api\ProductRepositoryInterface"
            type="Magento\Catalog\Model\ProductRepository"/>
```

This tells Magento: "When someone asks for `ProductRepositoryInterface`, give them `ProductRepository`"

### Example 2: Override Core Class

**Scenario**: You want to customize product repository behavior.

**Step 1**: Create your custom class

```php
// app/code/Vendor/Module/Model/CustomProductRepository.php
namespace Vendor\Module\Model;

use Magento\Catalog\Model\ProductRepository as CoreProductRepository;

class CustomProductRepository extends CoreProductRepository
{
    /**
     * Override get method to add custom logic
     */
    public function get($sku, $editMode = false, $storeId = null, $forceReload = false)
    {
        // Custom logic before
        $this->logger->info('Loading product: ' . $sku);

        // Call parent method
        $product = parent::get($sku, $editMode, $storeId, $forceReload);

        // Custom logic after
        $this->logger->info('Product loaded successfully');

        return $product;
    }
}
```

**Step 2**: Configure preference

```xml
<!-- etc/di.xml -->
<preference for="Magento\Catalog\Api\ProductRepositoryInterface"
            type="Vendor\Module\Model\CustomProductRepository"/>
```

### Example 3: Create Custom Interface Implementation

**Step 1**: Define interface

```php
// app/code/Vendor/Module/Api/EmailSenderInterface.php
namespace Vendor\Module\Api;

interface EmailSenderInterface
{
    /**
     * Send email
     *
     * @param string $to
     * @param string $subject
     * @param string $body
     * @return bool
     */
    public function send($to, $subject, $body);
}
```

**Step 2**: Create implementation

```php
// app/code/Vendor/Module/Model/SmtpEmailSender.php
namespace Vendor\Module\Model;

use Vendor\Module\Api\EmailSenderInterface;

class SmtpEmailSender implements EmailSenderInterface
{
    public function send($to, $subject, $body)
    {
        // SMTP implementation
        return true;
    }
}
```

**Step 3**: Configure preference

```xml
<preference for="Vendor\Module\Api\EmailSenderInterface"
            type="Vendor\Module\Model\SmtpEmailSender"/>
```

**Step 4**: Use in your code

```php
namespace Vendor\Module\Controller\Index;

use Vendor\Module\Api\EmailSenderInterface;

class Index extends \Magento\Framework\App\Action\Action
{
    protected $emailSender;

    public function __construct(
        \Magento\Framework\App\Action\Context $context,
        EmailSenderInterface $emailSender
    ) {
        $this->emailSender = $emailSender;
        parent::__construct($context);
    }

    public function execute()
    {
        $this->emailSender->send('user@example.com', 'Test', 'Hello!');
    }
}
```

## Virtual Types

Virtual types allow you to create "virtual" variations of a class without creating a new PHP class file.

### When to Use Virtual Types

- When you need multiple instances of the same class with different configurations
- To avoid creating unnecessary PHP classes
- For configuring complex dependencies

### Example 1: Different Logger Instances

```xml
<!-- Create virtual types for different log files -->
<virtualType name="OrderLogger" type="Magento\Framework\Logger\Monolog">
    <arguments>
        <argument name="name" xsi:type="string">order</argument>
        <argument name="handlers" xsi:type="array">
            <item name="system" xsi:type="object">OrderLoggerHandler</item>
        </argument>
    </arguments>
</virtualType>

<virtualType name="OrderLoggerHandler" type="Magento\Framework\Logger\Handler\Base">
    <arguments>
        <argument name="fileName" xsi:type="string">/var/log/order.log</argument>
    </arguments>
</virtualType>

<!-- Use the virtual logger in your class -->
<type name="Vendor\Module\Model\OrderProcessor">
    <arguments>
        <argument name="logger" xsi:type="object">OrderLogger</argument>
    </arguments>
</type>
```

### Example 2: Custom Resource Collections

```xml
<!-- Create virtual type for featured products collection -->
<virtualType name="FeaturedProductCollection" type="Magento\Catalog\Model\ResourceModel\Product\Collection">
    <arguments>
        <argument name="additionalData" xsi:type="array">
            <item name="is_featured" xsi:type="boolean">true</item>
        </argument>
    </arguments>
</virtualType>

<!-- Inject into your block -->
<type name="Vendor\Module\Block\FeaturedProducts">
    <arguments>
        <argument name="productCollection" xsi:type="object">FeaturedProductCollection</argument>
    </arguments>
</type>
```

### Example 3: Different API Clients

```xml
<!-- Production API Client -->
<virtualType name="ProductionApiClient" type="Vendor\Module\Model\ApiClient">
    <arguments>
        <argument name="config" xsi:type="array">
            <item name="base_url" xsi:type="string">https://api.production.com</item>
            <item name="timeout" xsi:type="number">30</item>
        </argument>
    </arguments>
</virtualType>

<!-- Sandbox API Client -->
<virtualType name="SandboxApiClient" type="Vendor\Module\Model\ApiClient">
    <arguments>
        <argument name="config" xsi:type="array">
            <item name="base_url" xsi:type="string">https://api.sandbox.com</item>
            <item name="timeout" xsi:type="number">60</item>
        </argument>
    </arguments>
</virtualType>
```

## Plugins vs Preferences

### Use Preferences When:

✅ You need to completely replace a class
✅ You're implementing an interface
✅ You want to extend and override methods

### Use Plugins When:

✅ You want to modify behavior without changing the class
✅ You need to run code before/after/around methods
✅ Multiple modules need to modify the same class

### Example Comparison

**Scenario**: Add logging to product save operation

**With Preference (replaces entire class):**
```xml
<preference for="Magento\Catalog\Api\ProductRepositoryInterface"
            type="Vendor\Module\Model\CustomProductRepository"/>
```

**With Plugin (modifies only save method):**
```xml
<type name="Magento\Catalog\Api\ProductRepositoryInterface">
    <plugin name="vendor_module_product_save_logger"
            type="Vendor\Module\Plugin\ProductRepositoryPlugin"/>
</type>
```

```php
class ProductRepositoryPlugin
{
    public function afterSave($subject, $result)
    {
        // Log after save
        return $result;
    }
}
```

**Plugin is better here** because it's less invasive and allows other modules to also modify the save behavior.

## Real-World Examples

### Example 1: Custom Shipping Method

**File**: `app/code/Vendor/CustomShipping/Model/Carrier/CustomCarrier.php`

```php
namespace Vendor\CustomShipping\Model\Carrier;

use Magento\Quote\Model\Quote\Address\RateRequest;
use Magento\Shipping\Model\Carrier\AbstractCarrier;
use Magento\Shipping\Model\Carrier\CarrierInterface;
use Magento\Framework\App\Config\ScopeConfigInterface;
use Magento\Quote\Model\Quote\Address\RateResult\ErrorFactory;
use Psr\Log\LoggerInterface;
use Magento\Shipping\Model\Rate\ResultFactory;
use Magento\Quote\Model\Quote\Address\RateResult\MethodFactory;

class CustomCarrier extends AbstractCarrier implements CarrierInterface
{
    protected $_code = 'customshipping';

    protected $rateResultFactory;
    protected $rateMethodFactory;

    /**
     * Constructor with DI
     */
    public function __construct(
        ScopeConfigInterface $scopeConfig,
        ErrorFactory $rateErrorFactory,
        LoggerInterface $logger,
        ResultFactory $rateResultFactory,
        MethodFactory $rateMethodFactory,
        array $data = []
    ) {
        $this->rateResultFactory = $rateResultFactory;
        $this->rateMethodFactory = $rateMethodFactory;
        parent::__construct($scopeConfig, $rateErrorFactory, $logger, $data);
    }

    /**
     * Get shipping rates
     */
    public function collectRates(RateRequest $request)
    {
        if (!$this->getConfigFlag('active')) {
            return false;
        }

        $result = $this->rateResultFactory->create();
        $method = $this->rateMethodFactory->create();

        $method->setCarrier($this->_code);
        $method->setCarrierTitle($this->getConfigData('title'));
        $method->setMethod($this->_code);
        $method->setMethodTitle($this->getConfigData('name'));
        $method->setPrice($this->getConfigData('price'));
        $method->setCost($this->getConfigData('price'));

        $result->append($method);
        return $result;
    }

    public function getAllowedMethods()
    {
        return [$this->_code => $this->getConfigData('name')];
    }
}
```

### Example 2: Custom Product Price Calculator

**Interface**: `app/code/Vendor/PriceCalculator/Api/PriceCalculatorInterface.php`

```php
namespace Vendor\PriceCalculator\Api;

interface PriceCalculatorInterface
{
    /**
     * Calculate custom price
     *
     * @param float $basePrice
     * @param int $quantity
     * @return float
     */
    public function calculate($basePrice, $quantity);
}
```

**Implementation**: `app/code/Vendor/PriceCalculator/Model/TieredPriceCalculator.php`

```php
namespace Vendor\PriceCalculator\Model;

use Vendor\PriceCalculator\Api\PriceCalculatorInterface;
use Magento\Framework\App\Config\ScopeConfigInterface;
use Psr\Log\LoggerInterface;

class TieredPriceCalculator implements PriceCalculatorInterface
{
    const XML_PATH_TIER_CONFIG = 'pricing/tier/config';

    protected $scopeConfig;
    protected $logger;
    protected $tierConfig;

    /**
     * Constructor
     */
    public function __construct(
        ScopeConfigInterface $scopeConfig,
        LoggerInterface $logger,
        array $tierConfig = []
    ) {
        $this->scopeConfig = $scopeConfig;
        $this->logger = $logger;
        $this->tierConfig = $tierConfig;
    }

    /**
     * Calculate tiered pricing
     */
    public function calculate($basePrice, $quantity)
    {
        $discount = 0;

        // 10-49 items: 5% discount
        if ($quantity >= 10 && $quantity < 50) {
            $discount = 0.05;
        }
        // 50-99 items: 10% discount
        elseif ($quantity >= 50 && $quantity < 100) {
            $discount = 0.10;
        }
        // 100+ items: 15% discount
        elseif ($quantity >= 100) {
            $discount = 0.15;
        }

        $finalPrice = $basePrice * (1 - $discount);

        $this->logger->info(sprintf(
            'Calculated price: Base=%s, Qty=%s, Discount=%s%%, Final=%s',
            $basePrice,
            $quantity,
            $discount * 100,
            $finalPrice
        ));

        return $finalPrice;
    }
}
```

**DI Configuration**: `app/code/Vendor/PriceCalculator/etc/di.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Map interface to implementation -->
    <preference for="Vendor\PriceCalculator\Api\PriceCalculatorInterface"
                type="Vendor\PriceCalculator\Model\TieredPriceCalculator"/>

    <!-- Configure tier settings -->
    <type name="Vendor\PriceCalculator\Model\TieredPriceCalculator">
        <arguments>
            <argument name="tierConfig" xsi:type="array">
                <item name="tier1" xsi:type="array">
                    <item name="min_qty" xsi:type="number">10</item>
                    <item name="discount" xsi:type="number">0.05</item>
                </item>
                <item name="tier2" xsi:type="array">
                    <item name="min_qty" xsi:type="number">50</item>
                    <item name="discount" xsi:type="number">0.10</item>
                </item>
                <item name="tier3" xsi:type="array">
                    <item name="min_qty" xsi:type="number">100</item>
                    <item name="discount" xsi:type="number">0.15</item>
                </item>
            </argument>
        </arguments>
    </type>

</config>
```

**Usage in Controller**:

```php
namespace Vendor\PriceCalculator\Controller\Index;

use Magento\Framework\App\Action\Action;
use Magento\Framework\App\Action\Context;
use Magento\Framework\Controller\Result\JsonFactory;
use Vendor\PriceCalculator\Api\PriceCalculatorInterface;

class Calculate extends Action
{
    protected $jsonFactory;
    protected $priceCalculator;

    public function __construct(
        Context $context,
        JsonFactory $jsonFactory,
        PriceCalculatorInterface $priceCalculator
    ) {
        $this->jsonFactory = $jsonFactory;
        $this->priceCalculator = $priceCalculator;
        parent::__construct($context);
    }

    public function execute()
    {
        $basePrice = (float) $this->getRequest()->getParam('price', 100);
        $quantity = (int) $this->getRequest()->getParam('qty', 1);

        $finalPrice = $this->priceCalculator->calculate($basePrice, $quantity);

        $result = $this->jsonFactory->create();
        return $result->setData([
            'base_price' => $basePrice,
            'quantity' => $quantity,
            'final_price' => $finalPrice,
            'savings' => $basePrice - $finalPrice
        ]);
    }
}
```

### Example 3: Multi-Currency Price Converter

**Interface**: `app/code/Vendor/Currency/Api/CurrencyConverterInterface.php`

```php
namespace Vendor\Currency\Api;

interface CurrencyConverterInterface
{
    /**
     * Convert amount from one currency to another
     *
     * @param float $amount
     * @param string $fromCurrency
     * @param string $toCurrency
     * @return float
     */
    public function convert($amount, $fromCurrency, $toCurrency);
}
```

**Implementation**: `app/code/Vendor/Currency/Model/CurrencyConverter.php`

```php
namespace Vendor\Currency\Model;

use Vendor\Currency\Api\CurrencyConverterInterface;
use Magento\Directory\Model\CurrencyFactory;
use Magento\Store\Model\StoreManagerInterface;
use Psr\Log\LoggerInterface;

class CurrencyConverter implements CurrencyConverterInterface
{
    protected $currencyFactory;
    protected $storeManager;
    protected $logger;
    protected $ratesCache = [];

    public function __construct(
        CurrencyFactory $currencyFactory,
        StoreManagerInterface $storeManager,
        LoggerInterface $logger
    ) {
        $this->currencyFactory = $currencyFactory;
        $this->storeManager = $storeManager;
        $this->logger = $logger;
    }

    public function convert($amount, $fromCurrency, $toCurrency)
    {
        if ($fromCurrency === $toCurrency) {
            return $amount;
        }

        try {
            $cacheKey = $fromCurrency . '_' . $toCurrency;

            if (!isset($this->ratesCache[$cacheKey])) {
                $currency = $this->currencyFactory->create();
                $currency->load($fromCurrency);
                $rate = $currency->getAnyRate($toCurrency);
                $this->ratesCache[$cacheKey] = $rate;
            }

            $convertedAmount = $amount * $this->ratesCache[$cacheKey];

            $this->logger->info(sprintf(
                'Currency conversion: %s %s = %s %s (rate: %s)',
                $amount,
                $fromCurrency,
                $convertedAmount,
                $toCurrency,
                $this->ratesCache[$cacheKey]
            ));

            return $convertedAmount;

        } catch (\Exception $e) {
            $this->logger->error('Currency conversion failed: ' . $e->getMessage());
            return $amount;
        }
    }
}
```

**DI Configuration**: `app/code/Vendor/Currency/etc/di.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <preference for="Vendor\Currency\Api\CurrencyConverterInterface"
                type="Vendor\Currency\Model\CurrencyConverter"/>

</config>
```

### Example 4: Custom Cache Manager with Virtual Types

**File**: `app/code/Vendor/Cache/etc/di.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Create virtual cache type for products -->
    <virtualType name="ProductCacheType" type="Magento\Framework\Cache\Frontend\Decorator\TagScope">
        <arguments>
            <argument name="cacheFrontend" xsi:type="object">Magento\Framework\App\Cache\Type\FrontendPool</argument>
            <argument name="tag" xsi:type="string">PRODUCT_CACHE</argument>
        </arguments>
    </virtualType>

    <!-- Create virtual cache type for categories -->
    <virtualType name="CategoryCacheType" type="Magento\Framework\Cache\Frontend\Decorator\TagScope">
        <arguments>
            <argument name="cacheFrontend" xsi:type="object">Magento\Framework\App\Cache\Type\FrontendPool</argument>
            <argument name="tag" xsi:type="string">CATEGORY_CACHE</argument>
        </arguments>
    </virtualType>

    <!-- Inject different cache types into managers -->
    <type name="Vendor\Cache\Model\ProductCacheManager">
        <arguments>
            <argument name="cache" xsi:type="object">ProductCacheType</argument>
        </arguments>
    </type>

    <type name="Vendor\Cache\Model\CategoryCacheManager">
        <arguments>
            <argument name="cache" xsi:type="object">CategoryCacheType</argument>
        </arguments>
    </type>

</config>
```

## Best Practices

### 1. Always Use Interfaces

✅ **Good:**
```php
public function __construct(ProductRepositoryInterface $productRepository)
```

❌ **Bad:**
```php
public function __construct(ProductRepository $productRepository)
```

### 2. Avoid Object Manager

✅ **Good:**
```php
public function __construct(ProductFactory $productFactory)
{
    $this->productFactory = $productFactory;
}

public function createProduct()
{
    return $this->productFactory->create();
}
```

❌ **Bad:**
```php
public function createProduct()
{
    $objectManager = \Magento\Framework\App\ObjectManager::getInstance();
    return $objectManager->create(\Magento\Catalog\Model\Product::class);
}
```

### 3. Use Factories for New Instances

When you need to create **new instances** (not singletons):

```php
public function __construct(
    \Magento\Catalog\Model\ProductFactory $productFactory
) {
    $this->productFactory = $productFactory;
}

public function createMultipleProducts()
{
    $product1 = $this->productFactory->create(); // New instance
    $product2 = $this->productFactory->create(); // Another new instance
}
```

### 4. Type Hint Appropriately

```php
// Type hint against interfaces
public function __construct(
    ProductRepositoryInterface $productRepository,  // Interface
    ScopeConfigInterface $scopeConfig,              // Interface
    LoggerInterface $logger                         // Interface
)
```

### 5. Use Constructor for Dependencies, Not Business Logic

✅ **Good:**
```php
public function __construct(ProductRepositoryInterface $productRepository)
{
    $this->productRepository = $productRepository;
}

public function getProduct($sku)
{
    return $this->productRepository->get($sku);
}
```

❌ **Bad:**
```php
public function __construct(ProductRepositoryInterface $productRepository)
{
    $this->productRepository = $productRepository;
    $this->product = $this->productRepository->get('SKU123'); // NO!
}
```

### 6. Organize di.xml Files by Area

```
etc/di.xml              # Global configuration
etc/frontend/di.xml     # Frontend-specific
etc/adminhtml/di.xml    # Admin-specific
```

### 7. Use Virtual Types to Avoid Class Proliferation

Instead of creating multiple similar classes, use virtual types:

```xml
<!-- Instead of creating ProductLogger.php, OrderLogger.php, etc. -->
<virtualType name="ProductLogger" type="Magento\Framework\Logger\Monolog">
    <arguments>
        <argument name="name" xsi:type="string">product</argument>
    </arguments>
</virtualType>

<virtualType name="OrderLogger" type="Magento\Framework\Logger\Monolog">
    <arguments>
        <argument name="name" xsi:type="string">order</argument>
    </arguments>
</virtualType>
```

### 8. Document Dependencies

```php
/**
 * Product Manager
 *
 * @package Vendor\Module
 */
class ProductManager
{
    /**
     * @var ProductRepositoryInterface
     */
    protected $productRepository;

    /**
     * @var LoggerInterface
     */
    protected $logger;

    /**
     * Constructor
     *
     * @param ProductRepositoryInterface $productRepository Product repository
     * @param LoggerInterface $logger Logger
     */
    public function __construct(
        ProductRepositoryInterface $productRepository,
        LoggerInterface $logger
    ) {
        $this->productRepository = $productRepository;
        $this->logger = $logger;
    }
}
```

## Common Patterns

### Pattern 1: Repository Pattern with DI

```php
class CustomerManager
{
    protected $customerRepository;
    protected $searchCriteriaBuilder;

    public function __construct(
        CustomerRepositoryInterface $customerRepository,
        SearchCriteriaBuilder $searchCriteriaBuilder
    ) {
        $this->customerRepository = $customerRepository;
        $this->searchCriteriaBuilder = $searchCriteriaBuilder;
    }

    public function getActiveCustomers()
    {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('is_active', 1)
            ->create();

        return $this->customerRepository->getList($searchCriteria);
    }
}
```

### Pattern 2: Factory Pattern with DI

```php
class OrderProcessor
{
    protected $orderFactory;
    protected $orderRepository;

    public function __construct(
        OrderInterfaceFactory $orderFactory,
        OrderRepositoryInterface $orderRepository
    ) {
        $this->orderFactory = $orderFactory;
        $this->orderRepository = $orderRepository;
    }

    public function createOrder($data)
    {
        $order = $this->orderFactory->create();
        $order->setData($data);
        return $this->orderRepository->save($order);
    }
}
```

### Pattern 3: Strategy Pattern with DI

```php
class PaymentProcessor
{
    protected $paymentStrategies;

    public function __construct(
        array $paymentStrategies = []
    ) {
        $this->paymentStrategies = $paymentStrategies;
    }

    public function process($method, $amount)
    {
        if (isset($this->paymentStrategies[$method])) {
            return $this->paymentStrategies[$method]->process($amount);
        }
        throw new \Exception('Payment method not supported');
    }
}
```

```xml
<type name="Vendor\Module\Model\PaymentProcessor">
    <arguments>
        <argument name="paymentStrategies" xsi:type="array">
            <item name="credit_card" xsi:type="object">Vendor\Module\Model\Payment\CreditCard</item>
            <item name="paypal" xsi:type="object">Vendor\Module\Model\Payment\PayPal</item>
            <item name="bank_transfer" xsi:type="object">Vendor\Module\Model\Payment\BankTransfer</item>
        </argument>
    </arguments>
</type>
```

## Troubleshooting

### Issue 1: Class Not Found

**Error**: `Class X does not exist`

**Solution**: Run `bin/magento setup:di:compile`

### Issue 2: Preference Not Working

**Check**:
1. Is di.xml in correct location?
2. Did you clear cache? `bin/magento cache:flush`
3. Is module enabled? `bin/magento module:status`

### Issue 3: Circular Dependency

**Error**: `Circular dependency: Class A depends on B, which depends on A`

**Solution**: Refactor to remove circular dependency or use Proxy

```php
// Use Proxy to break circular dependency
public function __construct(
    \Vendor\Module\Model\ClassA\Proxy $classA
)
```

## Next Steps

- [Day 05: Routing](../Day-05-Routing/README.md)
- Practice creating classes with proper DI
- Experiment with preferences and virtual types

## Additional Resources

- [Magento DevDocs: Dependency Injection](https://devdocs.magento.com/guides/v2.4/extension-dev-guide/depend-inj.html)
- [ObjectManager](https://devdocs.magento.com/guides/v2.4/extension-dev-guide/object-manager.html)
- [Factories](https://devdocs.magento.com/guides/v2.4/extension-dev-guide/factories.html)
- [Proxies](https://devdocs.magento.com/guides/v2.4/extension-dev-guide/proxies.html)
