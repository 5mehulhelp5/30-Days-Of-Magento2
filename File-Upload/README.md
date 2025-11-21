# Day 21: File & Image Upload in Magento 2

Learn how to implement file and image upload functionality in Magento 2, including form configuration, validation, and media storage management.

## Table of Contents
1. [Overview](#overview)
2. [UI Component File Upload](#ui-component-file-upload)
3. [Image Upload in Admin Forms](#image-upload-in-admin-forms)
4. [File Upload in Frontend Forms](#file-upload-in-frontend-forms)
5. [Image Processing & Validation](#image-processing--validation)
6. [Media Storage & File Management](#media-storage--file-management)
7. [Complete Example Module](#complete-example-module)
8. [Best Practices](#best-practices)

## Overview

Magento 2 provides several ways to handle file and image uploads:
- UI Component imageUploader for admin forms
- Traditional form file input fields
- MediaStorage library for file management
- Image adapters for image processing

### Key Components
- `Magento\Framework\File\Uploader` - Core file upload handler
- `Magento\MediaStorage\Model\File\UploaderFactory` - Factory for uploader
- `Magento\Framework\Filesystem` - Filesystem abstraction
- `Magento\Framework\Image\AdapterFactory` - Image processing

## UI Component File Upload

### 1. Form XML Configuration

Create a form with image uploader UI component:

**`view/adminhtml/ui_component/example_entity_form.xml`**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<form xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Ui:etc/ui_configuration.xsd">
    <fieldset name="general">
        <field name="image" formElement="imageUploader">
            <settings>
                <label translate="true">Image</label>
                <componentType>imageUploader</componentType>
            </settings>
            <formElements>
                <imageUploader>
                    <settings>
                        <allowedExtensions>jpg jpeg gif png</allowedExtensions>
                        <maxFileSize>2097152</maxFileSize>
                        <uploaderConfig>
                            <param xsi:type="string" name="url">example/entity/upload</param>
                        </uploaderConfig>
                        <previewTmpl>Magento_Catalog/image-preview</previewTmpl>
                        <openDialogTitle>Media Gallery</openDialogTitle>
                        <initialMediaGalleryOpenSubpath>wysiwyg</initialMediaGalleryOpenSubpath>
                        <validation>
                            <rule name="required-entry" xsi:type="boolean">false</rule>
                        </validation>
                    </settings>
                </imageUploader>
            </formElements>
        </field>
    </fieldset>
</form>
```

### 2. Upload Controller

**`Controller/Adminhtml/Entity/Upload.php`**
```php
<?php
namespace Vendor\Module\Controller\Adminhtml\Entity;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Magento\Catalog\Model\ImageUploader;
use Magento\Framework\Controller\ResultFactory;

class Upload extends Action
{
    /**
     * Authorization level of a basic admin session
     */
    const ADMIN_RESOURCE = 'Vendor_Module::entity';

    /**
     * @var ImageUploader
     */
    protected $imageUploader;

    /**
     * @param Context $context
     * @param ImageUploader $imageUploader
     */
    public function __construct(
        Context $context,
        ImageUploader $imageUploader
    ) {
        parent::__construct($context);
        $this->imageUploader = $imageUploader;
    }

    /**
     * Upload file controller action
     *
     * @return \Magento\Framework\Controller\ResultInterface
     */
    public function execute()
    {
        $imageId = $this->_request->getParam('param_name', 'image');

        try {
            $result = $this->imageUploader->saveFileToTmpDir($imageId);
        } catch (\Exception $e) {
            $result = ['error' => $e->getMessage(), 'errorcode' => $e->getCode()];
        }

        return $this->resultFactory->create(ResultFactory::TYPE_JSON)->setData($result);
    }
}
```

### 3. Configure ImageUploader via DI

**`etc/di.xml`**
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Virtual type for custom image uploader -->
    <virtualType name="Vendor\Module\EntityImageUploader" type="Magento\Catalog\Model\ImageUploader">
        <arguments>
            <argument name="baseTmpPath" xsi:type="string">vendor_module/tmp</argument>
            <argument name="basePath" xsi:type="string">vendor_module</argument>
            <argument name="allowedExtensions" xsi:type="array">
                <item name="jpg" xsi:type="string">jpg</item>
                <item name="jpeg" xsi:type="string">jpeg</item>
                <item name="gif" xsi:type="string">gif</item>
                <item name="png" xsi:type="string">png</item>
            </argument>
            <argument name="allowedMimeTypes" xsi:type="array">
                <item name="jpg" xsi:type="string">image/jpg</item>
                <item name="jpeg" xsi:type="string">image/jpeg</item>
                <item name="gif" xsi:type="string">image/gif</item>
                <item name="png" xsi:type="string">image/png</item>
            </argument>
        </arguments>
    </virtualType>

    <!-- Inject virtual type into controller -->
    <type name="Vendor\Module\Controller\Adminhtml\Entity\Upload">
        <arguments>
            <argument name="imageUploader" xsi:type="object">Vendor\Module\EntityImageUploader</argument>
        </arguments>
    </type>
</config>
```

## Image Upload in Admin Forms

### Custom Image Upload Handler

**`Model/ImageUploader.php`**
```php
<?php
namespace Vendor\Module\Model;

use Magento\Framework\App\Filesystem\DirectoryList;
use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\Filesystem;
use Magento\Framework\Filesystem\Directory\WriteInterface;
use Magento\Framework\UrlInterface;
use Magento\MediaStorage\Helper\File\Storage\Database;
use Magento\MediaStorage\Model\File\UploaderFactory;
use Magento\Store\Model\StoreManagerInterface;
use Psr\Log\LoggerInterface;

class ImageUploader
{
    /**
     * @var Database
     */
    protected $coreFileStorageDatabase;

    /**
     * @var WriteInterface
     */
    protected $mediaDirectory;

    /**
     * @var UploaderFactory
     */
    protected $uploaderFactory;

    /**
     * @var StoreManagerInterface
     */
    protected $storeManager;

    /**
     * @var LoggerInterface
     */
    protected $logger;

    /**
     * @var string
     */
    protected $baseTmpPath;

    /**
     * @var string
     */
    protected $basePath;

    /**
     * @var string[]
     */
    protected $allowedExtensions;

    /**
     * @var string[]
     */
    protected $allowedMimeTypes;

    /**
     * @param Database $coreFileStorageDatabase
     * @param Filesystem $filesystem
     * @param UploaderFactory $uploaderFactory
     * @param StoreManagerInterface $storeManager
     * @param LoggerInterface $logger
     * @param string $baseTmpPath
     * @param string $basePath
     * @param string[] $allowedExtensions
     * @param string[] $allowedMimeTypes
     */
    public function __construct(
        Database $coreFileStorageDatabase,
        Filesystem $filesystem,
        UploaderFactory $uploaderFactory,
        StoreManagerInterface $storeManager,
        LoggerInterface $logger,
        $baseTmpPath,
        $basePath,
        $allowedExtensions,
        $allowedMimeTypes = []
    ) {
        $this->coreFileStorageDatabase = $coreFileStorageDatabase;
        $this->mediaDirectory = $filesystem->getDirectoryWrite(DirectoryList::MEDIA);
        $this->uploaderFactory = $uploaderFactory;
        $this->storeManager = $storeManager;
        $this->logger = $logger;
        $this->baseTmpPath = $baseTmpPath;
        $this->basePath = $basePath;
        $this->allowedExtensions = $allowedExtensions;
        $this->allowedMimeTypes = $allowedMimeTypes;
    }

    /**
     * Set base tmp path
     *
     * @param string $baseTmpPath
     * @return void
     */
    public function setBaseTmpPath($baseTmpPath)
    {
        $this->baseTmpPath = $baseTmpPath;
    }

    /**
     * Set base path
     *
     * @param string $basePath
     * @return void
     */
    public function setBasePath($basePath)
    {
        $this->basePath = $basePath;
    }

    /**
     * Set allowed extensions
     *
     * @param string[] $allowedExtensions
     * @return void
     */
    public function setAllowedExtensions($allowedExtensions)
    {
        $this->allowedExtensions = $allowedExtensions;
    }

    /**
     * Retrieve base tmp path
     *
     * @return string
     */
    public function getBaseTmpPath()
    {
        return $this->baseTmpPath;
    }

    /**
     * Retrieve base path
     *
     * @return string
     */
    public function getBasePath()
    {
        return $this->basePath;
    }

    /**
     * Retrieve allowed extensions
     *
     * @return string[]
     */
    public function getAllowedExtensions()
    {
        return $this->allowedExtensions;
    }

    /**
     * Get file path
     *
     * @param string $path
     * @param string $fileName
     * @return string
     */
    public function getFilePath($path, $fileName)
    {
        return rtrim($path, '/') . '/' . ltrim($fileName, '/');
    }

    /**
     * Move file from tmp to target path
     *
     * @param string $imageName
     * @return string
     * @throws LocalizedException
     */
    public function moveFileFromTmp($imageName)
    {
        $baseTmpPath = $this->getBaseTmpPath();
        $basePath = $this->getBasePath();

        $baseImagePath = $this->getFilePath($basePath, $imageName);
        $baseTmpImagePath = $this->getFilePath($baseTmpPath, $imageName);

        try {
            $this->coreFileStorageDatabase->copyFile(
                $baseTmpImagePath,
                $baseImagePath
            );
            $this->mediaDirectory->renameFile(
                $baseTmpImagePath,
                $baseImagePath
            );
        } catch (\Exception $e) {
            throw new LocalizedException(
                __('Something went wrong while saving the file(s).')
            );
        }

        return $imageName;
    }

    /**
     * Save file to tmp dir
     *
     * @param string $fileId
     * @return array
     * @throws LocalizedException
     */
    public function saveFileToTmpDir($fileId)
    {
        $baseTmpPath = $this->getBaseTmpPath();

        $uploader = $this->uploaderFactory->create(['fileId' => $fileId]);
        $uploader->setAllowedExtensions($this->getAllowedExtensions());
        $uploader->setAllowRenameFiles(true);

        if (!$uploader->checkMimeType($this->allowedMimeTypes)) {
            throw new LocalizedException(__('File validation failed.'));
        }

        $result = $uploader->save($this->mediaDirectory->getAbsolutePath($baseTmpPath));
        unset($result['path']);

        if (!$result) {
            throw new LocalizedException(
                __('File can not be saved to the destination folder.')
            );
        }

        $result['tmp_name'] = str_replace('\\', '/', $result['tmp_name']);
        $result['url'] = $this->storeManager
                ->getStore()
                ->getBaseUrl(UrlInterface::URL_TYPE_MEDIA)
            . $this->getFilePath($baseTmpPath, $result['file']);
        $result['name'] = $result['file'];

        if (isset($result['file'])) {
            try {
                $relativePath = rtrim($baseTmpPath, '/') . '/' . ltrim($result['file'], '/');
                $this->coreFileStorageDatabase->saveFile($relativePath);
            } catch (\Exception $e) {
                $this->logger->critical($e);
                throw new LocalizedException(
                    __('Something went wrong while saving the file(s).')
                );
            }
        }

        return $result;
    }
}
```

## File Upload in Frontend Forms

### 1. Frontend Form Template

**`view/frontend/templates/upload_form.phtml`**
```php
<?php
/**
 * @var \Magento\Framework\View\Element\Template $block
 */
?>
<form action="<?= $block->escapeUrl($block->getFormAction()) ?>"
      method="post"
      id="upload-form"
      enctype="multipart/form-data"
      data-mage-init='{"validation":{}}'>

    <?= $block->getBlockHtml('formkey') ?>

    <fieldset class="fieldset">
        <legend class="legend"><span><?= __('Upload Image') ?></span></legend>

        <div class="field image required">
            <label class="label" for="image">
                <span><?= __('Image') ?></span>
            </label>
            <div class="control">
                <input type="file"
                       name="image"
                       id="image"
                       title="<?= __('Image') ?>"
                       class="input-text"
                       data-validate='{"required":true, "validate-image":true}'
                       accept="image/*" />
                <small class="note"><?= __('Allowed file types: jpg, jpeg, gif, png. Max file size: 2MB') ?></small>
            </div>
        </div>

        <div class="field file">
            <label class="label" for="attachment">
                <span><?= __('Attachment') ?></span>
            </label>
            <div class="control">
                <input type="file"
                       name="attachment"
                       id="attachment"
                       title="<?= __('Attachment') ?>"
                       class="input-text"
                       data-validate='{"validate-file-type":"pdf,doc,docx"}' />
            </div>
        </div>
    </fieldset>

    <div class="actions-toolbar">
        <div class="primary">
            <button type="submit" class="action submit primary">
                <span><?= __('Submit') ?></span>
            </button>
        </div>
    </div>
</form>
```

### 2. Frontend Upload Controller

**`Controller/Index/Upload.php`**
```php
<?php
namespace Vendor\Module\Controller\Index;

use Magento\Framework\App\Action\Action;
use Magento\Framework\App\Action\Context;
use Magento\Framework\App\Filesystem\DirectoryList;
use Magento\Framework\Controller\ResultFactory;
use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\Filesystem;
use Magento\MediaStorage\Model\File\UploaderFactory;
use Psr\Log\LoggerInterface;

class Upload extends Action
{
    /**
     * @var UploaderFactory
     */
    protected $uploaderFactory;

    /**
     * @var Filesystem
     */
    protected $filesystem;

    /**
     * @var LoggerInterface
     */
    protected $logger;

    /**
     * @param Context $context
     * @param UploaderFactory $uploaderFactory
     * @param Filesystem $filesystem
     * @param LoggerInterface $logger
     */
    public function __construct(
        Context $context,
        UploaderFactory $uploaderFactory,
        Filesystem $filesystem,
        LoggerInterface $logger
    ) {
        parent::__construct($context);
        $this->uploaderFactory = $uploaderFactory;
        $this->filesystem = $filesystem;
        $this->logger = $logger;
    }

    /**
     * Execute action
     *
     * @return \Magento\Framework\Controller\ResultInterface
     */
    public function execute()
    {
        $resultRedirect = $this->resultFactory->create(ResultFactory::TYPE_REDIRECT);

        if (!$this->getRequest()->isPost()) {
            $this->messageManager->addErrorMessage(__('Invalid request.'));
            return $resultRedirect->setPath('*/*/');
        }

        try {
            // Upload image
            $imageResult = $this->uploadFile('image', 'vendor_module/images');

            // Upload attachment (optional)
            $attachmentResult = null;
            if (isset($_FILES['attachment']) && $_FILES['attachment']['name']) {
                $attachmentResult = $this->uploadFile('attachment', 'vendor_module/attachments');
            }

            // Process the uploaded files
            // Save to database, send email, etc.

            $this->messageManager->addSuccessMessage(
                __('Files uploaded successfully.')
            );
        } catch (LocalizedException $e) {
            $this->messageManager->addErrorMessage($e->getMessage());
        } catch (\Exception $e) {
            $this->logger->critical($e);
            $this->messageManager->addErrorMessage(
                __('An error occurred while uploading the file.')
            );
        }

        return $resultRedirect->setPath('*/*/');
    }

    /**
     * Upload file
     *
     * @param string $fileId
     * @param string $destination
     * @return array
     * @throws LocalizedException
     */
    protected function uploadFile($fileId, $destination)
    {
        try {
            $uploader = $this->uploaderFactory->create(['fileId' => $fileId]);

            // Set allowed file types
            if ($fileId === 'image') {
                $uploader->setAllowedExtensions(['jpg', 'jpeg', 'gif', 'png']);
                $uploader->setAllowRenameFiles(true);
                $uploader->setFilesDispersion(true);
            } else {
                $uploader->setAllowedExtensions(['pdf', 'doc', 'docx']);
                $uploader->setAllowRenameFiles(true);
            }

            $mediaDirectory = $this->filesystem->getDirectoryRead(DirectoryList::MEDIA);
            $destinationPath = $mediaDirectory->getAbsolutePath($destination);

            $result = $uploader->save($destinationPath);

            if (!$result) {
                throw new LocalizedException(
                    __('File cannot be saved to the destination folder.')
                );
            }

            return $result;
        } catch (\Exception $e) {
            if ($e->getCode() != \Magento\MediaStorage\Model\File\Uploader::TMP_NAME_EMPTY) {
                throw new LocalizedException(__($e->getMessage()));
            }
        }

        return [];
    }
}
```

## Image Processing & Validation

### Custom Image Validator

**`Model/File/Validator/Image.php`**
```php
<?php
namespace Vendor\Module\Model\File\Validator;

use Magento\Framework\Exception\ValidatorException;

class Image
{
    /**
     * Maximum file size in bytes (2MB)
     */
    const MAX_FILE_SIZE = 2097152;

    /**
     * Allowed image MIME types
     */
    const ALLOWED_MIME_TYPES = [
        'image/jpg',
        'image/jpeg',
        'image/gif',
        'image/png'
    ];

    /**
     * Minimum image dimensions
     */
    const MIN_WIDTH = 100;
    const MIN_HEIGHT = 100;

    /**
     * Maximum image dimensions
     */
    const MAX_WIDTH = 4000;
    const MAX_HEIGHT = 4000;

    /**
     * Validate uploaded image
     *
     * @param array $file
     * @return bool
     * @throws ValidatorException
     */
    public function validate($file)
    {
        if (!isset($file['tmp_name']) || !file_exists($file['tmp_name'])) {
            throw new ValidatorException(__('The file does not exist.'));
        }

        // Validate file size
        if ($file['size'] > self::MAX_FILE_SIZE) {
            throw new ValidatorException(
                __('The file size exceeds the maximum allowed size of %1 MB.',
                   self::MAX_FILE_SIZE / 1024 / 1024)
            );
        }

        // Validate MIME type
        $finfo = finfo_open(FILEINFO_MIME_TYPE);
        $mimeType = finfo_file($finfo, $file['tmp_name']);
        finfo_close($finfo);

        if (!in_array($mimeType, self::ALLOWED_MIME_TYPES)) {
            throw new ValidatorException(
                __('Invalid file type. Allowed types: jpg, jpeg, gif, png.')
            );
        }

        // Validate image dimensions
        $imageSize = getimagesize($file['tmp_name']);
        if (!$imageSize) {
            throw new ValidatorException(__('The file is not a valid image.'));
        }

        list($width, $height) = $imageSize;

        if ($width < self::MIN_WIDTH || $height < self::MIN_HEIGHT) {
            throw new ValidatorException(
                __('Image dimensions must be at least %1x%2 pixels.',
                   self::MIN_WIDTH, self::MIN_HEIGHT)
            );
        }

        if ($width > self::MAX_WIDTH || $height > self::MAX_HEIGHT) {
            throw new ValidatorException(
                __('Image dimensions must not exceed %1x%2 pixels.',
                   self::MAX_WIDTH, self::MAX_HEIGHT)
            );
        }

        return true;
    }
}
```

### Image Resizing Helper

**`Helper/Image.php`**
```php
<?php
namespace Vendor\Module\Helper;

use Magento\Framework\App\Filesystem\DirectoryList;
use Magento\Framework\App\Helper\AbstractHelper;
use Magento\Framework\App\Helper\Context;
use Magento\Framework\Filesystem;
use Magento\Framework\Image\AdapterFactory;

class Image extends AbstractHelper
{
    /**
     * @var Filesystem
     */
    protected $filesystem;

    /**
     * @var AdapterFactory
     */
    protected $imageFactory;

    /**
     * @param Context $context
     * @param Filesystem $filesystem
     * @param AdapterFactory $imageFactory
     */
    public function __construct(
        Context $context,
        Filesystem $filesystem,
        AdapterFactory $imageFactory
    ) {
        parent::__construct($context);
        $this->filesystem = $filesystem;
        $this->imageFactory = $imageFactory;
    }

    /**
     * Resize image
     *
     * @param string $imagePath
     * @param int|null $width
     * @param int|null $height
     * @return string
     */
    public function resize($imagePath, $width = null, $height = null)
    {
        $mediaDirectory = $this->filesystem->getDirectoryRead(DirectoryList::MEDIA);
        $absolutePath = $mediaDirectory->getAbsolutePath($imagePath);

        if (!file_exists($absolutePath)) {
            return '';
        }

        $imageResized = $this->filesystem
            ->getDirectoryRead(DirectoryList::MEDIA)
            ->getAbsolutePath('resized/' . $width . 'x' . $height . '/') . basename($imagePath);

        if (file_exists($imageResized)) {
            return $imageResized;
        }

        $imageAdapter = $this->imageFactory->create();
        $imageAdapter->open($absolutePath);
        $imageAdapter->constrainOnly(true);
        $imageAdapter->keepTransparency(true);
        $imageAdapter->keepFrame(false);
        $imageAdapter->keepAspectRatio(true);
        $imageAdapter->resize($width, $height);
        $imageAdapter->save($imageResized);

        return $imageResized;
    }

    /**
     * Create thumbnail
     *
     * @param string $imagePath
     * @param int $size
     * @return string
     */
    public function createThumbnail($imagePath, $size = 150)
    {
        return $this->resize($imagePath, $size, $size);
    }
}
```

## Media Storage & File Management

### File Manager Service

**`Model/FileManager.php`**
```php
<?php
namespace Vendor\Module\Model;

use Magento\Framework\App\Filesystem\DirectoryList;
use Magento\Framework\Exception\FileSystemException;
use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\Filesystem;
use Magento\Framework\Filesystem\Driver\File;

class FileManager
{
    /**
     * @var Filesystem
     */
    protected $filesystem;

    /**
     * @var File
     */
    protected $file;

    /**
     * @param Filesystem $filesystem
     * @param File $file
     */
    public function __construct(
        Filesystem $filesystem,
        File $file
    ) {
        $this->filesystem = $filesystem;
        $this->file = $file;
    }

    /**
     * Delete file
     *
     * @param string $filePath
     * @return bool
     * @throws LocalizedException
     */
    public function deleteFile($filePath)
    {
        try {
            $mediaDirectory = $this->filesystem->getDirectoryWrite(DirectoryList::MEDIA);
            $absolutePath = $mediaDirectory->getAbsolutePath($filePath);

            if ($this->file->isExists($absolutePath)) {
                $this->file->deleteFile($absolutePath);
                return true;
            }
        } catch (FileSystemException $e) {
            throw new LocalizedException(__('Unable to delete file: %1', $e->getMessage()));
        }

        return false;
    }

    /**
     * Copy file
     *
     * @param string $source
     * @param string $destination
     * @return bool
     * @throws LocalizedException
     */
    public function copyFile($source, $destination)
    {
        try {
            $mediaDirectory = $this->filesystem->getDirectoryWrite(DirectoryList::MEDIA);
            $sourcePath = $mediaDirectory->getAbsolutePath($source);
            $destinationPath = $mediaDirectory->getAbsolutePath($destination);

            if ($this->file->isExists($sourcePath)) {
                $this->file->copy($sourcePath, $destinationPath);
                return true;
            }
        } catch (FileSystemException $e) {
            throw new LocalizedException(__('Unable to copy file: %1', $e->getMessage()));
        }

        return false;
    }

    /**
     * Get file info
     *
     * @param string $filePath
     * @return array
     */
    public function getFileInfo($filePath)
    {
        $mediaDirectory = $this->filesystem->getDirectoryRead(DirectoryList::MEDIA);
        $absolutePath = $mediaDirectory->getAbsolutePath($filePath);

        if (!$this->file->isExists($absolutePath)) {
            return [];
        }

        $stat = $this->file->stat($absolutePath);

        return [
            'size' => $stat['size'],
            'modified_time' => $stat['mtime'],
            'mime_type' => mime_content_type($absolutePath),
            'extension' => pathinfo($absolutePath, PATHINFO_EXTENSION)
        ];
    }
}
```

## Complete Example Module

### Module Structure
```
app/code/Vendor/FileUpload/
├── Controller/
│   ├── Adminhtml/
│   │   └── Upload/
│   │       ├── Index.php
│   │       └── Save.php
│   └── Index/
│       └── Upload.php
├── Model/
│   ├── Entity.php
│   ├── ResourceModel/
│   │   ├── Entity.php
│   │   └── Entity/
│   │       └── Collection.php
│   ├── ImageUploader.php
│   └── FileManager.php
├── etc/
│   ├── module.xml
│   ├── di.xml
│   └── adminhtml/
│       └── routes.xml
├── view/
│   ├── adminhtml/
│   │   └── ui_component/
│   │       └── vendor_fileupload_entity_form.xml
│   └── frontend/
│       └── templates/
│           └── upload_form.phtml
└── registration.php
```

### Data Model with Image

**`Model/Entity.php`**
```php
<?php
namespace Vendor\FileUpload\Model;

use Magento\Framework\Model\AbstractModel;
use Vendor\FileUpload\Model\ResourceModel\Entity as EntityResource;

class Entity extends AbstractModel
{
    /**
     * Initialize resource model
     *
     * @return void
     */
    protected function _construct()
    {
        $this->_init(EntityResource::class);
    }

    /**
     * Get image URL
     *
     * @return string|null
     */
    public function getImageUrl()
    {
        if ($this->getImage()) {
            return $this->_storeManager->getStore()
                ->getBaseUrl(\Magento\Framework\UrlInterface::URL_TYPE_MEDIA)
                . 'vendor_fileupload/' . $this->getImage();
        }
        return null;
    }
}
```

## Best Practices

### 1. Security
- Always validate file types and MIME types
- Check file size limits
- Sanitize filenames
- Store uploaded files outside web root when possible
- Use random filenames to prevent overwriting
- Implement virus scanning for production environments

### 2. Performance
- Compress images after upload
- Generate thumbnails asynchronously
- Use CDN for serving uploaded media
- Implement lazy loading for images
- Cache resized images

### 3. Validation
```php
// Validate file extension
$uploader->setAllowedExtensions(['jpg', 'jpeg', 'gif', 'png']);

// Validate MIME type
$uploader->checkMimeType(['image/jpeg', 'image/png', 'image/gif']);

// Validate file size (in controller)
if ($_FILES['image']['size'] > 2097152) { // 2MB
    throw new LocalizedException(__('File size too large'));
}
```

### 4. Error Handling
```php
try {
    $result = $this->imageUploader->saveFileToTmpDir('image');
} catch (\Exception $e) {
    $this->logger->critical($e);
    $this->messageManager->addErrorMessage(
        __('An error occurred while uploading the file.')
    );
}
```

### 5. Database Storage
- Store only the filename/path, not the file content
- Use relative paths
- Keep track of file metadata (size, type, upload date)
- Implement soft delete to allow file recovery

### 6. File Organization
```
pub/media/
├── vendor_module/
│   ├── tmp/              # Temporary uploads
│   ├── images/           # Final image storage
│   ├── attachments/      # Document storage
│   └── cache/            # Resized images cache
```

## Summary

This guide covers:
- UI Component imageUploader for admin forms
- Custom file upload handlers
- Frontend file upload forms
- Image validation and processing
- Media storage management
- File system operations
- Security best practices

## Next Steps
- Implement image cropping functionality
- Add watermarking capabilities
- Integrate with cloud storage (AWS S3, Google Cloud Storage)
- Implement advanced image optimization
- Add support for multiple file uploads
- Create image gallery components
