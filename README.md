# php-cvent-wrapper
The [Cvent SOAP API](https://developers.cvent.com/documentation/soap-api/) offers the ability to read Event data and perform CRUD operations against many Person-style objects. This wrapper should simplify the Login, Search and Retrieve operations for read-only access to the system.

## Composer Install

```
composer require matthewpoer/php-cvent-wrapper:dev-master
```

## Sandbox or Production?
By default, this wrapper will access the Production Cvent API, but setting the `$sandbox` param. on invocation of the class will send your calls to the Sandbox API instead, e.g.

```
// Production
$wrapper = new php_cvent_wrapper();

// Sandbox
$wrapper = new php_cvent_wrapper(TRUE);
```

## Authentication Notes
Accessing the Cvent API, both for Sandbox and Production accounts, requires an Account Number, Username and Password for an API Account, which are treated differently by Cvent than typical user accounts. One must also configure Cvent to whitelist incoming API requests' IP addresses as an additional security measure. See Cvent's notes on [Login Basic Steps](https://developers.cvent.com/documentation/soap-api/call-definitions/authentication/login/).

## Shoutouts
Portions of this code were inspired by [php-cvent](https://github.com/gcanivet/php-cvent).

## Sample Code

### Authorization
```
<?php
require_once('config.php');
require_once('vendor/autoload.php');

// Authorization
$php_cvent_wrapper = new php_cvent_wrapper(TRUE);
try {
  $result = $php_cvent_wrapper->login(
    CVENT_ACCOUNT_NUMBER,
    CVENT_USERNAME,
    CVENT_PASSWORD
  );
  if(!$result) {
    die('Cvent authentication failed for an unknown reason' . PHP_EOL);
  }
} catch (\CventAuthorizationFailureException | \CventAuthorizationLockoutException $e) {
  echo 'Cvent Auth Error: ' . $e->getMessage();
  die();
} catch (\Exception $e) {
  echo 'Failed to authenticate with Cvent' . PHP_EOL;
  echo $e->getMessage();
  die();
}

echo "Authentication was successful." . PHP_EOL;
```

### Get a List of Fields for an Object
```
try {
  $fields = $php_cvent_wrapper->describe_object_fields('Registration');
} catch (\Exception $e) {
  echo 'Failed to get fields list' . PHP_EOL;
  echo $e->getMessage();
  die();
}
```

### Search
```
try {
  $users = $php_cvent_wrapper->search(
    'User',
    array(
      (object)array(
        'Field' => 'UserRole',
        'Operator' => 'Equals',
        'Value' => 'Administrators',
      )
    )
  );
  echo "Here's a list of all of the User IDs for Administrators:" . PHP_EOL;
  foreach ($users as $user_id) {
    echo 'User ID is ' . $user_id . PHP_EOL;
  }
} catch (\Exception $e) {
  echo 'Failed to search for a list of Administrators' . PHP_EOL;
  echo $e->getMessage();
  die();
}
```

### Retrieve
```
try {
  $user_data = $php_cvent_wrapper->retrieve(
    'User',
    $users,
    array(
      'Email',
      'Id',
      'UserType',
      'UserRole',
    )
  );
  foreach($user_data as $user_info) {
    echo 'The email address for user ' . $user_info['Id'] . ' is ' . $user_info['Email'] . PHP_EOL;
  }
} catch (\Exception $e) {
  echo 'Failed to retrieve user data for our list of Administrators' . PHP_EOL;
  echo $e->getMessage();
  die();
}
```

### Search and Retrieve in a Single Call
```
//
try {
  $events = $php_cvent_wrapper->search_and_retrieve(
    'Event',
    array(
      (object)array(
        'Field' => 'EventStartDate',
        'Operator' => 'Greater than',
        'Value' => date('Y-m-d\TH:m:s'),
      )
    ),
    array(
      'EventCode',
      'EventStartDate',
      'EventTitle',
      'Id',
    )
  );
  foreach($events as $event) {
    echo 'Event ' . $event['EventTitle'] . ' will begin on ' . $event['EventStartDate'] . PHP_EOL;
  }
} catch (\Exception $e) {
  echo 'Failed to search and retrieve info. for future events' . PHP_EOL;
  echo $e->getMessage();
  die();
}
```
