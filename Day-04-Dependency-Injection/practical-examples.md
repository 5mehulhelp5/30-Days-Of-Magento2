# Dependency Injection - Practical Examples

This document contains complete, real-world examples demonstrating DI concepts in Magento 2.

## Example 1: Product Import Service

A complete service for importing products with proper dependency injection.

### Directory Structure

```
app/code/Acme/ProductImport/
├── Api/
│   └── ProductImportInterface.php
├── Model/
│   ├── ProductImport.php
│   └── Validator/
│       └── ProductValidator.php
├── etc/
│   ├── di.xml
│   └── module.xml
└── registration.php
```

### Interface

**File**: `Api/ProductImportInterface.php`

```php
<?php
namespace Acme\ProductImport\Api;

interface ProductImportInterface
{
    /**
     * Import products from array
     *
     * @param array $productsData
     * @return array
     */
    public function import(array $productsData);
}
```

### Validator

**File**: `Model/Validator/ProductValidator.php`

```php
<?php
namespace Acme\ProductImport\Model\Validator;

use Psr\Log\LoggerInterface;

class ProductValidator
{
    protected $logger;
    protected $requiredFields = ['sku', 'name', 'price', 'type_id'];

    public function __construct(
        LoggerInterface $logger,
        array $requiredFields = []
    ) {
        $this->logger = $logger;
        if (!empty($requiredFields)) {
            $this->requiredFields = $requiredFields;
        }
    }

    /**
     * Validate product data
     *
     * @param array $productData
     * @return array
     */
    public function validate(array $productData)
    {
        $errors = [];

        foreach ($this->requiredFields as $field) {
            if (!isset($productData[$field]) || empty($productData[$field])) {
                $errors[] = sprintf('Required field "%s" is missing or empty', $field);
            }
        }

        if (isset($productData['price']) && !is_numeric($productData['price'])) {
            $errors[] = 'Price must be numeric';
        }

        if (isset($productData['price']) && $productData['price'] < 0) {
            $errors[] = 'Price cannot be negative';
        }

        if (!empty($errors)) {
            $this->logger->warning('Product validation failed', [
                'sku' => $productData['sku'] ?? 'unknown',
                'errors' => $errors
            ]);
        }

        return $errors;
    }
}
```

### Implementation

**File**: `Model/ProductImport.php`

```php
<?php
namespace Acme\ProductImport\Model;

use Acme\ProductImport\Api\ProductImportInterface;
use Acme\ProductImport\Model\Validator\ProductValidator;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Catalog\Api\Data\ProductInterfaceFactory;
use Magento\Framework\Exception\CouldNotSaveException;
use Psr\Log\LoggerInterface;

class ProductImport implements ProductImportInterface
{
    protected $productRepository;
    protected $productFactory;
    protected $validator;
    protected $logger;

    /**
     * Constructor with all dependencies injected
     *
     * @param ProductRepositoryInterface $productRepository
     * @param ProductInterfaceFactory $productFactory
     * @param ProductValidator $validator
     * @param LoggerInterface $logger
     */
    public function __construct(
        ProductRepositoryInterface $productRepository,
        ProductInterfaceFactory $productFactory,
        ProductValidator $validator,
        LoggerInterface $logger
    ) {
        $this->productRepository = $productRepository;
        $this->productFactory = $productFactory;
        $this->validator = $validator;
        $this->logger = $logger;
    }

    /**
     * Import products
     *
     * @param array $productsData
     * @return array
     */
    public function import(array $productsData)
    {
        $result = [
            'success' => 0,
            'failed' => 0,
            'errors' => []
        ];

        foreach ($productsData as $index => $productData) {
            try {
                // Validate product data
                $validationErrors = $this->validator->validate($productData);
                if (!empty($validationErrors)) {
                    $result['failed']++;
                    $result['errors'][$index] = $validationErrors;
                    continue;
                }

                // Try to load existing product
                try {
                    $product = $this->productRepository->get($productData['sku']);
                } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
                    // Product doesn't exist, create new
                    $product = $this->productFactory->create();
                }

                // Set product data
                $product->setSku($productData['sku']);
                $product->setName($productData['name']);
                $product->setPrice($productData['price']);
                $product->setTypeId($productData['type_id']);
                $product->setAttributeSetId($productData['attribute_set_id'] ?? 4);
                $product->setStatus($productData['status'] ?? 1);
                $product->setVisibility($productData['visibility'] ?? 4);

                // Save product
                $this->productRepository->save($product);

                $result['success']++;
                $this->logger->info('Product imported successfully', [
                    'sku' => $productData['sku']
                ]);

            } catch (CouldNotSaveException $e) {
                $result['failed']++;
                $result['errors'][$index] = ['Save error: ' . $e->getMessage()];
                $this->logger->error('Failed to import product', [
                    'sku' => $productData['sku'] ?? 'unknown',
                    'error' => $e->getMessage()
                ]);
            } catch (\Exception $e) {
                $result['failed']++;
                $result['errors'][$index] = ['Unexpected error: ' . $e->getMessage()];
                $this->logger->error('Unexpected error during import', [
                    'sku' => $productData['sku'] ?? 'unknown',
                    'error' => $e->getMessage()
                ]);
            }
        }

        return $result;
    }
}
```

### DI Configuration

**File**: `etc/di.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Map interface to implementation -->
    <preference for="Acme\ProductImport\Api\ProductImportInterface"
                type="Acme\ProductImport\Model\ProductImport"/>

    <!-- Configure validator with custom required fields -->
    <type name="Acme\ProductImport\Model\Validator\ProductValidator">
        <arguments>
            <argument name="requiredFields" xsi:type="array">
                <item name="0" xsi:type="string">sku</item>
                <item name="1" xsi:type="string">name</item>
                <item name="2" xsi:type="string">price</item>
                <item name="3" xsi:type="string">type_id</item>
                <item name="4" xsi:type="string">attribute_set_id</item>
            </argument>
        </arguments>
    </type>

</config>
```

### Usage Example

**File**: `Controller/Adminhtml/Import/Save.php`

```php
<?php
namespace Acme\ProductImport\Controller\Adminhtml\Import;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Magento\Framework\Controller\Result\JsonFactory;
use Acme\ProductImport\Api\ProductImportInterface;

class Save extends Action
{
    protected $jsonFactory;
    protected $productImport;

    public function __construct(
        Context $context,
        JsonFactory $jsonFactory,
        ProductImportInterface $productImport
    ) {
        $this->jsonFactory = $jsonFactory;
        $this->productImport = $productImport;
        parent::__construct($context);
    }

    public function execute()
    {
        $productsData = [
            [
                'sku' => 'PROD-001',
                'name' => 'Product 1',
                'price' => 99.99,
                'type_id' => 'simple',
                'attribute_set_id' => 4
            ],
            [
                'sku' => 'PROD-002',
                'name' => 'Product 2',
                'price' => 149.99,
                'type_id' => 'simple',
                'attribute_set_id' => 4
            ]
        ];

        $result = $this->productImport->import($productsData);

        $resultJson = $this->jsonFactory->create();
        return $resultJson->setData($result);
    }
}
```

## Example 2: Email Notification System

Complete email notification system with multiple email types using virtual types.

### Directory Structure

```
app/code/Acme/Notification/
├── Api/
│   └── EmailSenderInterface.php
├── Model/
│   └── EmailSender.php
├── etc/
│   ├── di.xml
│   └── email_templates.xml
```

### Interface

**File**: `Api/EmailSenderInterface.php`

```php
<?php
namespace Acme\Notification\Api;

interface EmailSenderInterface
{
    /**
     * Send email
     *
     * @param string $to
     * @param array $templateVars
     * @return bool
     */
    public function send($to, array $templateVars);
}
```

### Implementation

**File**: `Model/EmailSender.php`

```php
<?php
namespace Acme\Notification\Model;

use Acme\Notification\Api\EmailSenderInterface;
use Magento\Framework\Mail\Template\TransportBuilder;
use Magento\Framework\Translate\Inline\StateInterface;
use Magento\Store\Model\StoreManagerInterface;
use Psr\Log\LoggerInterface;

class EmailSender implements EmailSenderInterface
{
    protected $transportBuilder;
    protected $inlineTranslation;
    protected $storeManager;
    protected $logger;
    protected $templateId;
    protected $senderEmail;
    protected $senderName;

    public function __construct(
        TransportBuilder $transportBuilder,
        StateInterface $inlineTranslation,
        StoreManagerInterface $storeManager,
        LoggerInterface $logger,
        $templateId = 'default_template',
        $senderEmail = 'sales',
        $senderName = 'Sales Team'
    ) {
        $this->transportBuilder = $transportBuilder;
        $this->inlineTranslation = $inlineTranslation;
        $this->storeManager = $storeManager;
        $this->logger = $logger;
        $this->templateId = $templateId;
        $this->senderEmail = $senderEmail;
        $this->senderName = $senderName;
    }

    public function send($to, array $templateVars)
    {
        try {
            $this->inlineTranslation->suspend();

            $store = $this->storeManager->getStore();

            $transport = $this->transportBuilder
                ->setTemplateIdentifier($this->templateId)
                ->setTemplateOptions([
                    'area' => \Magento\Framework\App\Area::AREA_FRONTEND,
                    'store' => $store->getId()
                ])
                ->setTemplateVars($templateVars)
                ->setFrom([
                    'email' => $this->senderEmail,
                    'name' => $this->senderName
                ])
                ->addTo($to)
                ->getTransport();

            $transport->sendMessage();

            $this->inlineTranslation->resume();

            $this->logger->info('Email sent successfully', [
                'to' => $to,
                'template' => $this->templateId
            ]);

            return true;

        } catch (\Exception $e) {
            $this->logger->error('Failed to send email', [
                'to' => $to,
                'error' => $e->getMessage()
            ]);
            $this->inlineTranslation->resume();
            return false;
        }
    }
}
```

### DI Configuration with Virtual Types

**File**: `etc/di.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Order Confirmation Email -->
    <virtualType name="OrderConfirmationEmail" type="Acme\Notification\Model\EmailSender">
        <arguments>
            <argument name="templateId" xsi:type="string">order_confirmation_template</argument>
            <argument name="senderEmail" xsi:type="string">sales@example.com</argument>
            <argument name="senderName" xsi:type="string">Sales Department</argument>
        </arguments>
    </virtualType>

    <!-- Shipment Notification Email -->
    <virtualType name="ShipmentNotificationEmail" type="Acme\Notification\Model\EmailSender">
        <arguments>
            <argument name="templateId" xsi:type="string">shipment_notification_template</argument>
            <argument name="senderEmail" xsi:type="string">shipping@example.com</argument>
            <argument name="senderName" xsi:type="string">Shipping Department</argument>
        </arguments>
    </virtualType>

    <!-- Invoice Email -->
    <virtualType name="InvoiceEmail" type="Acme\Notification\Model\EmailSender">
        <arguments>
            <argument name="templateId" xsi:type="string">invoice_template</argument>
            <argument name="senderEmail" xsi:type="string">billing@example.com</argument>
            <argument name="senderName" xsi:type="string">Billing Department</argument>
        </arguments>
    </virtualType>

    <!-- Customer Welcome Email -->
    <virtualType name="CustomerWelcomeEmail" type="Acme\Notification\Model\EmailSender">
        <arguments>
            <argument name="templateId" xsi:type="string">customer_welcome_template</argument>
            <argument name="senderEmail" xsi:type="string">support@example.com</argument>
            <argument name="senderName" xsi:type="string">Customer Support</argument>
        </arguments>
    </virtualType>

    <!-- Inject virtual types into specific services -->
    <type name="Acme\Notification\Model\OrderNotificationService">
        <arguments>
            <argument name="orderEmailSender" xsi:type="object">OrderConfirmationEmail</argument>
            <argument name="invoiceEmailSender" xsi:type="object">InvoiceEmail</argument>
        </arguments>
    </type>

    <type name="Acme\Notification\Model\ShippingNotificationService">
        <arguments>
            <argument name="emailSender" xsi:type="object">ShipmentNotificationEmail</argument>
        </arguments>
    </type>

    <type name="Acme\Notification\Model\CustomerNotificationService">
        <arguments>
            <argument name="emailSender" xsi:type="object">CustomerWelcomeEmail</argument>
        </arguments>
    </type>

</config>
```

### Service Classes Using Virtual Types

**File**: `Model/OrderNotificationService.php`

```php
<?php
namespace Acme\Notification\Model;

use Acme\Notification\Api\EmailSenderInterface;
use Magento\Sales\Api\Data\OrderInterface;

class OrderNotificationService
{
    protected $orderEmailSender;
    protected $invoiceEmailSender;

    /**
     * Note: These are virtual types injected via di.xml
     */
    public function __construct(
        EmailSenderInterface $orderEmailSender,
        EmailSenderInterface $invoiceEmailSender
    ) {
        $this->orderEmailSender = $orderEmailSender;
        $this->invoiceEmailSender = $invoiceEmailSender;
    }

    public function sendOrderConfirmation(OrderInterface $order)
    {
        $templateVars = [
            'order' => $order,
            'order_id' => $order->getIncrementId(),
            'customer_name' => $order->getCustomerName()
        ];

        return $this->orderEmailSender->send(
            $order->getCustomerEmail(),
            $templateVars
        );
    }

    public function sendInvoiceNotification(OrderInterface $order)
    {
        $templateVars = [
            'order' => $order,
            'order_id' => $order->getIncrementId()
        ];

        return $this->invoiceEmailSender->send(
            $order->getCustomerEmail(),
            $templateVars
        );
    }
}
```

## Example 3: Payment Gateway Integration

Complete payment gateway with strategy pattern and DI.

### Directory Structure

```
app/code/Acme/Payment/
├── Api/
│   └── PaymentGatewayInterface.php
├── Model/
│   ├── Gateway/
│   │   ├── StripeGateway.php
│   │   ├── PayPalGateway.php
│   │   └── AuthorizeNetGateway.php
│   └── PaymentProcessor.php
```

### Interface

**File**: `Api/PaymentGatewayInterface.php`

```php
<?php
namespace Acme\Payment\Api;

interface PaymentGatewayInterface
{
    /**
     * Process payment
     *
     * @param float $amount
     * @param array $paymentData
     * @return array
     */
    public function processPayment($amount, array $paymentData);

    /**
     * Refund payment
     *
     * @param string $transactionId
     * @param float $amount
     * @return array
     */
    public function refund($transactionId, $amount);
}
```

### Gateway Implementations

**File**: `Model/Gateway/StripeGateway.php`

```php
<?php
namespace Acme\Payment\Model\Gateway;

use Acme\Payment\Api\PaymentGatewayInterface;
use Magento\Framework\HTTP\Client\Curl;
use Psr\Log\LoggerInterface;

class StripeGateway implements PaymentGatewayInterface
{
    protected $curl;
    protected $logger;
    protected $apiKey;
    protected $apiUrl;

    public function __construct(
        Curl $curl,
        LoggerInterface $logger,
        $apiKey = '',
        $apiUrl = 'https://api.stripe.com/v1'
    ) {
        $this->curl = $curl;
        $this->logger = $logger;
        $this->apiKey = $apiKey;
        $this->apiUrl = $apiUrl;
    }

    public function processPayment($amount, array $paymentData)
    {
        try {
            $this->logger->info('Processing Stripe payment', [
                'amount' => $amount
            ]);

            // Stripe API call simulation
            $response = [
                'success' => true,
                'transaction_id' => 'stripe_' . uniqid(),
                'amount' => $amount,
                'gateway' => 'stripe'
            ];

            $this->logger->info('Stripe payment successful', $response);
            return $response;

        } catch (\Exception $e) {
            $this->logger->error('Stripe payment failed', [
                'error' => $e->getMessage()
            ]);
            return [
                'success' => false,
                'error' => $e->getMessage()
            ];
        }
    }

    public function refund($transactionId, $amount)
    {
        $this->logger->info('Processing Stripe refund', [
            'transaction_id' => $transactionId,
            'amount' => $amount
        ]);

        return [
            'success' => true,
            'refund_id' => 'refund_' . uniqid(),
            'amount' => $amount
        ];
    }
}
```

**File**: `Model/Gateway/PayPalGateway.php`

```php
<?php
namespace Acme\Payment\Model\Gateway;

use Acme\Payment\Api\PaymentGatewayInterface;
use Psr\Log\LoggerInterface;

class PayPalGateway implements PaymentGatewayInterface
{
    protected $logger;
    protected $apiUsername;
    protected $apiPassword;
    protected $apiUrl;

    public function __construct(
        LoggerInterface $logger,
        $apiUsername = '',
        $apiPassword = '',
        $apiUrl = 'https://api.paypal.com'
    ) {
        $this->logger = $logger;
        $this->apiUsername = $apiUsername;
        $this->apiPassword = $apiPassword;
        $this->apiUrl = $apiUrl;
    }

    public function processPayment($amount, array $paymentData)
    {
        $this->logger->info('Processing PayPal payment', [
            'amount' => $amount
        ]);

        return [
            'success' => true,
            'transaction_id' => 'paypal_' . uniqid(),
            'amount' => $amount,
            'gateway' => 'paypal'
        ];
    }

    public function refund($transactionId, $amount)
    {
        $this->logger->info('Processing PayPal refund', [
            'transaction_id' => $transactionId,
            'amount' => $amount
        ]);

        return [
            'success' => true,
            'refund_id' => 'refund_' . uniqid(),
            'amount' => $amount
        ];
    }
}
```

### Payment Processor with Strategy Pattern

**File**: `Model/PaymentProcessor.php`

```php
<?php
namespace Acme\Payment\Model;

use Psr\Log\LoggerInterface;

class PaymentProcessor
{
    protected $paymentGateways;
    protected $logger;

    /**
     * Constructor
     *
     * @param LoggerInterface $logger
     * @param array $paymentGateways Array of gateway implementations
     */
    public function __construct(
        LoggerInterface $logger,
        array $paymentGateways = []
    ) {
        $this->logger = $logger;
        $this->paymentGateways = $paymentGateways;
    }

    /**
     * Process payment using specified gateway
     *
     * @param string $gateway
     * @param float $amount
     * @param array $paymentData
     * @return array
     */
    public function process($gateway, $amount, array $paymentData)
    {
        if (!isset($this->paymentGateways[$gateway])) {
            $this->logger->error('Payment gateway not found', [
                'gateway' => $gateway
            ]);
            return [
                'success' => false,
                'error' => 'Payment gateway not supported: ' . $gateway
            ];
        }

        $this->logger->info('Processing payment', [
            'gateway' => $gateway,
            'amount' => $amount
        ]);

        return $this->paymentGateways[$gateway]->processPayment($amount, $paymentData);
    }

    /**
     * Refund payment
     *
     * @param string $gateway
     * @param string $transactionId
     * @param float $amount
     * @return array
     */
    public function refund($gateway, $transactionId, $amount)
    {
        if (!isset($this->paymentGateways[$gateway])) {
            return [
                'success' => false,
                'error' => 'Payment gateway not supported: ' . $gateway
            ];
        }

        return $this->paymentGateways[$gateway]->refund($transactionId, $amount);
    }

    /**
     * Get available payment gateways
     *
     * @return array
     */
    public function getAvailableGateways()
    {
        return array_keys($this->paymentGateways);
    }
}
```

### DI Configuration

**File**: `etc/di.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Configure Stripe Gateway -->
    <type name="Acme\Payment\Model\Gateway\StripeGateway">
        <arguments>
            <argument name="apiKey" xsi:type="string">sk_test_xxxxxxxxxxxxx</argument>
            <argument name="apiUrl" xsi:type="string">https://api.stripe.com/v1</argument>
        </arguments>
    </type>

    <!-- Configure PayPal Gateway -->
    <type name="Acme\Payment\Model\Gateway\PayPalGateway">
        <arguments>
            <argument name="apiUsername" xsi:type="string">paypal_username</argument>
            <argument name="apiPassword" xsi:type="string">paypal_password</argument>
            <argument name="apiUrl" xsi:type="string">https://api.paypal.com</argument>
        </arguments>
    </type>

    <!-- Inject all gateways into PaymentProcessor -->
    <type name="Acme\Payment\Model\PaymentProcessor">
        <arguments>
            <argument name="paymentGateways" xsi:type="array">
                <item name="stripe" xsi:type="object">Acme\Payment\Model\Gateway\StripeGateway</item>
                <item name="paypal" xsi:type="object">Acme\Payment\Model\Gateway\PayPalGateway</item>
                <item name="authorizenet" xsi:type="object">Acme\Payment\Model\Gateway\AuthorizeNetGateway</item>
            </argument>
        </arguments>
    </type>

</config>
```

### Usage Example

```php
<?php
namespace Acme\Payment\Controller\Checkout;

use Magento\Framework\App\Action\Action;
use Magento\Framework\App\Action\Context;
use Acme\Payment\Model\PaymentProcessor;

class ProcessPayment extends Action
{
    protected $paymentProcessor;

    public function __construct(
        Context $context,
        PaymentProcessor $paymentProcessor
    ) {
        $this->paymentProcessor = $paymentProcessor;
        parent::__construct($context);
    }

    public function execute()
    {
        $gateway = $this->getRequest()->getParam('gateway', 'stripe');
        $amount = $this->getRequest()->getParam('amount', 0);

        $paymentData = [
            'card_number' => '4242424242424242',
            'exp_month' => '12',
            'exp_year' => '2025',
            'cvv' => '123'
        ];

        $result = $this->paymentProcessor->process($gateway, $amount, $paymentData);

        if ($result['success']) {
            $this->messageManager->addSuccessMessage(
                __('Payment processed successfully. Transaction ID: %1', $result['transaction_id'])
            );
        } else {
            $this->messageManager->addErrorMessage(
                __('Payment failed: %1', $result['error'])
            );
        }

        return $this->_redirect('checkout/success');
    }
}
```

## Summary

These examples demonstrate:

1. **Constructor Injection** - All dependencies injected via constructor
2. **Interface-based Design** - Using interfaces for flexibility
3. **Virtual Types** - Creating variations without new PHP classes
4. **Strategy Pattern** - Multiple implementations selected at runtime
5. **Proper Logging** - Injected logger for debugging
6. **Configuration via di.xml** - Centralized dependency configuration
7. **Testability** - Easy to mock dependencies for unit tests

All examples follow Magento 2 best practices and can be used as templates for real projects.
