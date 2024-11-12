# magento2-swatch-image

Here’s a full working example for a Magento 2 module that handles image uploads, including the optional swatch_image attribute handling for products. This example assumes that your module's form allows setting or clearing both a custom image and swatch_image if needed.

Step 1: Define Image Fields in the UI Component Form
To create both a custom image field and a swatch_image field in your form, add entries in the form.xml file. This file is used to define your custom form in the admin panel.

`app/code/YourVendor/YourModule/view/adminhtml/ui_component/<grid_name>_form.xml`


```xml
<field name="image" formElement="fileUploader">
    <settings>
        <dataType>text</dataType>
        <label translate="true">Image</label>
        <notice translate="true">Allowed file types: jpg, jpeg, png, gif</notice>
        <validation>
            <rule name="required-entry" xsi:type="boolean">true</rule>
        </validation>
        <dataScope>image</dataScope>
        <uploaderConfig>
            <param xsi:type="string" name="url">*/*/upload</param>
        </uploaderConfig>
    </settings>
</field>
<field name="swatch_image" formElement="fileUploader">
    <settings>
        <dataType>text</dataType>
        <label translate="true">Swatch Image</label>
        <notice translate="true">Allowed file types: jpg, jpeg, png, gif</notice>
        <validation>
            <rule name="required-entry" xsi:type="boolean">false</rule>
        </validation>
        <dataScope>swatch_image</dataScope>
        <uploaderConfig>
            <param xsi:type="string" name="url">*/*/upload</param>
        </uploaderConfig>
    </settings>
</field>
```
Step 2: Create the Image Upload Controller
The upload controller handles saving the uploaded images to pub/media. This controller will respond to the AJAX upload request for both the image and swatch_image fields.

`app/code/YourVendor/YourModule/Controller/Adminhtml/Entity/Upload.php`

```php

namespace YourVendor\YourModule\Controller\Adminhtml\Entity;

use Magento\Backend\App\Action;
use Magento\Framework\Controller\Result\JsonFactory;
use Magento\MediaStorage\Model\File\UploaderFactory;
use Magento\Framework\Filesystem;
use Magento\Framework\App\Filesystem\DirectoryList;

class Upload extends Action
{
    protected $resultJsonFactory;
    protected $uploaderFactory;
    protected $filesystem;

    public function __construct(
        Action\Context $context,
        JsonFactory $resultJsonFactory,
        UploaderFactory $uploaderFactory,
        Filesystem $filesystem
    ) {
        $this->resultJsonFactory = $resultJsonFactory;
        $this->uploaderFactory = $uploaderFactory;
        $this->filesystem = $filesystem;
        parent::__construct($context);
    }

    public function execute()
    {
        $result = $this->resultJsonFactory->create();
        try {
            $uploader = $this->uploaderFactory->create(['fileId' => $this->getRequest()->getParam('field')]);
            $uploader->setAllowedExtensions(['jpg', 'jpeg', 'png', 'gif']);
            $uploader->setAllowRenameFiles(true);
            $uploader->setFilesDispersion(false);

            $mediaDirectory = $this->filesystem->getDirectoryWrite(DirectoryList::MEDIA);
            $resultSave = $uploader->save($mediaDirectory->getAbsolutePath('custom/images'));

            if ($resultSave) {
                $filePath = 'custom/images/' . $resultSave['file'];
                $result->setData([
                    'file' => $filePath,
                    'url' => $this->getUrl('pub/media/' . $filePath),
                ]);
            }
        } catch (\Exception $e) {
            $result->setData(['error' => $e->getMessage()]);
        }

        return $result;
    }
}
```

Step 3: Modify the Save Controller to Handle Image and Swatch Image
Here, we modify the Save controller to manage both image and swatch_image attributes. If the fields are empty, it clears them from the database.

`app/code/YourVendor/YourModule/Controller/Adminhtml/Entity/Save.php`

```php

namespace YourVendor\YourModule\Controller\Adminhtml\Entity;

use Magento\Backend\App\Action;
use Magento\Framework\App\Filesystem\DirectoryList;
use Magento\MediaStorage\Model\File\UploaderFactory;
use Magento\Framework\Filesystem;

class Save extends Action
{
    protected $uploaderFactory;
    protected $filesystem;

    public function __construct(
        Action\Context $context,
        UploaderFactory $uploaderFactory,
        Filesystem $filesystem
    ) {
        $this->uploaderFactory = $uploaderFactory;
        $this->filesystem = $filesystem;
        parent::__construct($context);
    }

    public function execute()
    {
        $data = $this->getRequest()->getPostValue();

        // Handle main image
        if (isset($data['image']) && is_array($data['image'])) {
            $data['image'] = $data['image']['value'];
        } else {
            $data['image'] = null; // Clear image if not provided
        }

        // Handle swatch image
        if (isset($data['swatch_image']) && is_array($data['swatch_image'])) {
            $data['swatch_image'] = $data['swatch_image']['value'];
        } else {
            $data['swatch_image'] = null; // Clear swatch image if not provided
        }

        // Save the entity (this example assumes you have a model/repository to handle it)
        try {
            $model = $this->_objectManager->create('YourVendor\YourModule\Model\YourEntity');
            $model->setData($data);
            $model->save();

            $this->messageManager->addSuccessMessage(__('The entity has been saved.'));
        } catch (\Exception $e) {
            $this->messageManager->addErrorMessage($e->getMessage());
        }

        return $this->_redirect('*/*/');
    }
}
```
4. Update the Database to Store Image and Swatch Image Paths
Ensure your database table has columns to store both image and swatch_image paths. If they don’t exist, create a db_schema.xml file for setting up these fields.

`app/code/YourVendor/YourModule/etc/db_schema.xml`

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/db_schema.xsd">
    <table name="your_table_name" resource="default" engine="innodb" comment="Custom Table">
        <column xsi:type="int" name="entity_id" nullable="false" identity="true" unsigned="true" comment="Entity ID"/>
        <column xsi:type="varchar" name="image" nullable="true" length="255" comment="Image"/>
        <column xsi:type="varchar" name="swatch_image" nullable="true" length="255" comment="Swatch Image"/>
        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="entity_id"/>
        </constraint>
    </table>
</schema>
```

Summary
Upload controller: Handles uploading images and swatch images to pub/media/custom/images.
Save controller: Updates both image and swatch_image paths in the database, setting fields to null if an image is removed.
Database: Includes fields for image and swatch_image in the database table.
This should provide you with a flexible setup for managing both regular and swatch images in your custom Magento module! Let me know if you need additional adjustments.
