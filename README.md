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

### Filtered Search
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

#### Search Without Filters
The same search method can be used with no filters as well, e.g. to get a list of all known Speakers:
```
try {
  $speakers = $php_cvent_wrapper->search('Speaker');
  echo "Here's a list of all of the Speaker IDs:" . PHP_EOL;
  foreach ($speakers as $speaker_id) {
    echo 'Speaker ID is ' . $speaker_id . PHP_EOL;
  }
} catch (\Exception $e) {
  echo 'Failed to search for a list of Speakers' . PHP_EOL;
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

### Search using Includes (i.e. multiple values)
The following will find a list of Active and Completed events that have been
modified in the past week. Notice that the filter uses `ValueArray` instead of
just `Value`.
```
try {
  $events = $php_cvent_wrapper->search_and_retrieve(
    'Event',
    array(
      (object)array(
        'Field' => 'LastModifiedDate',
        'Operator' => 'Greater than',
        'Value' => date('Y-m-d\TH:m:s', strtotime('-1 week')),
      ),
      (object)array(
        'Field' => 'EventStatus',
        'Operator' => 'Includes',
        'ValueArray' => array(
          'Active',
          'Completed',
        )
      ),
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
  echo 'Failed to search and retrieve info. for recently modified active and completed events' . PHP_EOL;
  echo $e->getMessage();
  die();
}
```

### Registration Questions
Registration Questions are stored as objects within the Registration object, and so these can be "flattened" out into a single field and/or converted to an array for more granular processing. Note that the 5th param. in `search_and_retrieve()` sets the 4th param. in `search()`, which is the `$always_flat` flag. It defaults to TRUE (assuming that most of the time one would not want an array in the otherwise-flat record data), but when set to FALSE registration data is returned as an array.
```
// `$always_flat` is the 4th param in retrieve
$registrations = $php_cvent_wrapper->retrieve(
  'Registration',
  $Ids,
  $Fields,
  FALSE
);

// `$always_flat` is the 5th param in search_and_retrieve
$registrations = $php_cvent_wrapper->search_and_retrieve(
  'Registration',
  $Filters,
  $Fields,
  'AndSearch'
  FALSE
);
```

The returned data will include the Answer as text regardless of whether `$always_flat` is true or false, so dumping the `$registration['Answer']` as below will give the following output:

```
$registration = current(registrations);
var_dump($registration['Answer']);
/*
string(204) "Question:
Here is an example event question (001)
Response:
Here is a response to the question (001)

Question:
Here is an another example event question (002)
Response:
And here is another response (002)"
*/
```

But if `$always_flat` is set to false, you can dump an additional array for potentially easier question separation:

```
$registration = current(registrations);
var_dump($registration['Answer Array']);
/*
array(2) {
  ["Here is an example event question (001)"]=>
  string(40) "Here is a response to the question (001)"
  ["Here is an another example event question (002)"]=>
  string(47) "Here is an another example event question (002)"
}
*/
```
