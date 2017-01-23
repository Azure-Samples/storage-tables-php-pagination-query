---
services: storage
platforms: php
author: msonecode
---

# How to do queries and pagination for Azure Table Storage in PHP

## Introduction
This sample demonstrates [how to do queries and pagination for Azure Table Storage in PHP](https://code.msdn.microsoft.com/How-to-do-queries-and-08c9ee14).

## Sample prerequisites
- PHP 5.5 or above
- See composer.json for dependencies
- [PHP Tools for Visual Studio][1](Optional)

## Building the sample
Install via Composer: open a command prompt and execute under command in your project root.  
> php composer.phar install  

![][2] 

## Running the sample
- In index.php file, replace "&lt;yourAccount&gt;" and "&lt;yourKey&gt;" with your own account and key.
- Press F5 to debug.  
![][3]

Then you will see a query result page:  
![][4]

## Using the code
In Azure Table Storage query, there's no wildcard searching command that’s equivalent to LIKE command in T-SQL. You'll see commands such as eq, gt, ge, lt, le and ne. 

``` php
$filter =  "PropertyName ne 'Example100'"; 
$options = new QueryEntitiesOptions(); 
$options->setFilter(Filter::applyQueryString($filter));
```

Pagination for Azure Table Storage

``` php
<?php 
require_once "vendor/autoload.php"; 
use JasonGrimes\Paginator; 
use MicrosoftAzure\Storage\Common\ServicesBuilder; 
use MicrosoftAzure\Storage\Common\ServiceException; 
use MicrosoftAzure\Storage\Table\Models\BatchOperations; 
use MicrosoftAzure\Storage\Table\Models\Entity; 
use MicrosoftAzure\Storage\Table\Models\EdmType; 
use MicrosoftAzure\Storage\Table\Models\QueryEntitiesOptions; 
use MicrosoftAzure\Storage\Table\Models\Filters\Filter; 
 
$accountName = '<yourAccount>'; 
$accountKey = '<yourKey>'; 
//Set azure storage table name 
$tableName = "testPaginationTable"; 
 
$connectionString = 'DefaultEndpointsProtocol=https;AccountName=' . $accountName . ';AccountKey=' .$accountKey. ''; 
$tableClient = ServicesBuilder::getInstance()->createTableService($connectionString); 
 
//Set number of records per page. 
$numPerPage = 5; 
//Set current page 
if (isset($_GET["page"])) { $page  = $_GET["page"]; } else { $page=1; }; 
 
 
//Initialization test table and data if the table does not exist. 
createTableInitDataSample($tableClient, $tableName); 
 
//In the azure table storage query, there's no direct equivalent of T-sql's LIKE command, as there is no wildcard searching. 
//You'll see eq, gt, ge, lt, le, etc. All supported operations are listed here. https://msdn.microsoft.com/library/azure/dd894031.aspx?f=255&MSPPError=-2147217396 
$filter =  "PropertyName ne 'Example100'"; 
 
//Total records numbers 
$totalCount = getTotalCount($tableClient,$tableName, $filter); 
 
//Query and Pagination 
$results = queryPaginationEntitiesSample($tableClient,$tableName,$numPerPage,$page, $filter); 
$urlPattern = 'index.php?page=(:num)'; 
$paginator = new Paginator($totalCount, $numPerPage, $page, $urlPattern); 
 
function createTableInitDataSample($tableClient, $tableName) 
{ 
    try { 
        $tableClient->createTable($tableName); 
        batchInsertEntitiesSample($tableClient, $tableName); 
    } 
    catch(ServiceException $e){ 
        $code = $e->getCode(); 
        //409 The table specified already exists. 
        if($code != 409){ 
            $error_message = $e->getMessage(); 
            echo $code.": ".$error_message.PHP_EOL; 
        } 
    } 
} 
 
function batchInsertEntitiesSample($tableClient, $tableName) 
{ 
    $batchOp = new BatchOperations(); 
    for ($i = 1; $i <= 50; ++$i) 
    { 
        $entity = new Entity(); 
        $entity->setPartitionKey("pk"); 
        $entity->setRowKey(''.$i); 
        $entity->addProperty("PropertyName", EdmType::STRING, "Sample".$i); 
        $batchOp->addInsertEntity($tableName, $entity); 
    } 
    for ($i = 51; $i <= 100; ++$i) 
    { 
        $entity = new Entity(); 
        $entity->setPartitionKey("pk"); 
        $entity->setRowKey(''.$i); 
        $entity->addProperty("PropertyName", EdmType::STRING, "Example".$i); 
        $batchOp->addInsertEntity($tableName, $entity); 
    } 
    try { 
        $tableClient->batch($batchOp); 
    } 
    catch(ServiceException $e){ 
        $code = $e->getCode(); 
        $error_message = $e->getMessage(); 
        echo $code.": ".$error_message.PHP_EOL; 
    } 
} 
 
function getTotalCount($tableClient, $tableName, $filter) 
{ 
    $options = new QueryEntitiesOptions(); 
    $options->setSelectFields(array('pk')); 
    $options->setFilter(Filter::applyQueryString($filter)); 
    try { 
        $result = $tableClient->queryEntities($tableName, $options); 
        $entities = $result->getEntities(); 
        return count($entities); 
    } 
    catch(ServiceException $e){ 
        $code = $e->getCode(); 
        $error_message = $e->getMessage(); 
        echo $code.": ".$error_message.PHP_EOL; 
        return 0; 
    } 
} 
 
function queryPaginationEntitiesSample($tableClient, $tableName, $numPerPage, $page, $filter) 
{ 
    try { 
        $options = new QueryEntitiesOptions(); 
        $options->setFilter(Filter::applyQueryString($filter)); 
        if($page== 1){ 
            $options->setTop($numPerPage); 
            $result = $tableClient->queryEntities($tableName, $options); 
            $entities = $result->getEntities(); 
        } 
        else{ 
            //skip $numPerPage * ($page-1) records 
            $options->setTop($numPerPage * ($page-1)); 
            $options->setSelectFields(array('pk')); 
            $result = $tableClient->queryEntities($tableName, $options); 
            $nextRowKey = $result->getNextRowKey(); 
            $nextPartitionKey = $result->getNextPartitionKey(); 
 
            $options = new QueryEntitiesOptions(); 
            $options->setFilter(Filter::applyQueryString($filter)); 
            $options->setTop($numPerPage); 
            $options->setNextRowKey($nextRowKey); 
            $options->setNextPartitionKey($nextPartitionKey); 
            $result = $tableClient->queryEntities($tableName, $options); 
            $entities = $result->getEntities(); 
        } 
 
        return $entities; 
    } 
    catch(ServiceException $e){ 
        $code = $e->getCode(); 
        $error_message = $e->getMessage(); 
        echo $code.": ".$error_message.PHP_EOL; 
        return null; 
    } 
} 
?> 
 
 
<html> 
<head> 
    <link rel="stylesheet" href="//maxcdn.bootstrapcdn.com/bootstrap/3.2.0/css/bootstrap.min.css" /> 
</head> 
<body> 
    <table class="table table-bordered"> 
        <tr> 
            <th>Name</th> 
        </tr> 
        <?php 
        foreach ($results as $result) { 
            echo "<tr><td>" . $result->getProperty("PropertyName")->getValue() ."</td></tr>"; 
        } 
        ?> 
 
    </table> 
    <?php 
    echo $paginator; 
    ?> 
 
</body> 
</html>
```

## More information
- [Microsoft Azure Storage SDK for PHP][5]
- [PHP Paginator][6]
- [How to use table storage from PHP][7]
- [Querying Tables and Entities][8]

[1]: https://visualstudiogallery.msdn.microsoft.com/6eb51f05-ef01-4513-ac83-4c5f50c95fb5
[2]: images/1.png
[3]: images/2.png
[4]: images/3.png
[5]: https://github.com/Azure/azure-storage-php
[6]: https://github.com/jasongrimes/php-paginator
[7]: https://github.com/jasongrimes/php-paginatorHow%20to%20use%20table%20storage%20from%20PHP
[8]: https://msdn.microsoft.com/library/azure/dd894031.aspx?f=255&MSPPError=-2147217396
