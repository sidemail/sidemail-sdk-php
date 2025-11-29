# Sidemail PHP library

Official Sidemail.io PHP library provides convenient access to the Sidemail API from PHP applications.

## Requirements

- PHP 8.1+
- `ext-curl`

## Installation

Install this package with:

```bash
composer require sidemail/sidemail
```

## Usage

The package needs to be configured with your project's API key, which you can find in the Sidemail Dashboard. Here is how to send your first email:

```php
<?php

require __DIR__ . '/vendor/autoload.php';

use Sidemail\Sidemail;

// Create Sidemail instance and set your API key.
$sidemail = new Sidemail(apiKey: 'xxxxx');

$response = $sidemail->sendEmail([
    'toAddress'     => 'user@email.com',
    'fromAddress'   => 'you@example.com',
    'fromName'      => 'Your app',
    'templateName'  => 'Welcome',
    'templateProps' => ['foo' => 'bar'],
]);

echo "Email sent! ID: {$response->id}";
```

The response will look like this:

```json
{
  "id": "5e858953daf20f3aac50a3da",
  "status": "queued"
}
```

Shortcut `$sidemail->sendEmail(...)` calls `$sidemail->email->send(...)` under the hood.

### Authentication

Explicit key:

```php
use Sidemail\Sidemail;

$sidemail = new Sidemail(apiKey: 'your-api-key');
```

Or if you set environment variable `SIDEMAIL_API_KEY`, then simply:

```php
use Sidemail\Sidemail;

$sidemail = new Sidemail(); // reads SIDEMAIL_API_KEY
```

### Client configuration

```php
use Sidemail\Sidemail;

$sidemail = new Sidemail(
    apiKey: 'your-api-key',
    baseUrl: 'https://api.sidemail.io/v1', // override for testing/mocking
    timeout: 10.0, // per-request timeout (seconds)
    httpClient: $customHttpClient, // custom HttpClient implementation (proxies, retries, etc.)
);
```

## Email sending examples

### Send password reset email template

```php
$sidemail->sendEmail([
    'toAddress'     => 'user@email.com',
    'fromAddress'   => 'you@example.com',
    'fromName'      => 'Your app',
    'templateName'  => 'Password reset',
    'templateProps' => ['resetUrl' => 'https://your.app/reset?token=123'],
]);
```

### Schedule email delivery

```php
$sidemail->sendEmail([
    'toAddress'     => 'user@email.com',
    'fromName'      => 'Startup name',
    'fromAddress'   => 'your@startup.com',
    'templateName'  => 'Welcome',
    'templateProps' => ['firstName' => 'John'],
    // Deliver email in 60 minutes from now
    'scheduledAt'   => (new DateTime('+60 minutes'))->format(DateTime::ATOM),
]);
```

### Send email template with dynamic list

Useful for dynamic data where you have `n` items that you want to render in email. For example, items in a receipt, weekly statistic per project, new comments, etc.

```php
$sidemail->sendEmail([
    'toAddress'     => 'user@email.com',
    'fromName'      => 'Startup name',
    'fromAddress'   => 'your@startup.com',
    'templateName'  => 'Template with dynamic list',
    'templateProps' => [
        'list' => [
            ['text' => 'Dynamic list'],
            ['text' => 'allows you to generate email template content'],
            ['text' => 'based on template props.'],
        ],
    ],
]);
```

### Send custom HTML email

```php
$sidemail->sendEmail([
    'toAddress'   => 'user@email.com',
    'fromName'    => 'Startup name',
    'fromAddress' => 'your@startup.com',
    'subject'     => 'Testing html only custom emails :)',
    'html'        => '<html><body><h1>Hello world! 👋</h1></body></html>',
]);
```

### Send custom plain text email

```php
$sidemail->sendEmail([
    'toAddress'   => 'user@email.com',
    'fromName'    => 'Startup name',
    'fromAddress' => 'your@startup.com',
    'subject'     => 'Testing plain-text only custom emails :)',
    'text'        => 'Hello world! 👋',
]);
```

## Error handling

All errors thrown by this library are instances of `Sidemail\SidemailException`.
This exception provides additional error details via getter methods:

| Property        | Description                           |
| --------------- | ------------------------------------- |
| getMessage()    | Exception message (human-readable)    |
| getHttpStatus() | HTTP status code if available         |
| getErrorCode()  | Sidemail error code if provided       |
| getMoreInfo()   | Additional info or documentation link |

```php

use Sidemail\Sidemail;
use Sidemail\SidemailException;

try {
    $response = $sidemail->sendEmail([
        'toAddress'    => 'user@email.com',
        'fromAddress'  => 'you@example.com',
        'fromName'     => 'Your app',
        'templateName' => 'Welcome',
    ]);
} catch (SidemailException $e) {
    // All Sidemail errors are caught here
    echo "Error: " . $e->getMessage() . PHP_EOL;
    echo "HTTP status: " . var_export($e->getHttpStatus(), true) . PHP_EOL;
    echo "Error code: " . var_export($e->getErrorCode(), true) . PHP_EOL;
    echo "More info: " . var_export($e->getMoreInfo(), true) . PHP_EOL;
}
```

## Attachments helper

You can use the `Sidemail::fileToAttachment` static helper to easily encode file data for attachments:

```php
$pdfData = file_get_contents('./invoice.pdf');
$attachment = Sidemail::fileToAttachment('invoice.pdf', $pdfData);

$sidemail->sendEmail([
    'toAddress'   => 'user@email.com',
    'fromAddress' => 'you@example.com',
    'subject'     => 'Invoice',
    'text'        => 'Invoice attached.',
    'attachments' => [$attachment],
]);
```

## Auto-pagination

The package provides automatic pagination for list and search endpoints that return paginated results. This allows you to iterate through all results without manually handling pagination cursors.

```php
$result = $sidemail->contacts->list();

foreach ($result->autoPaginate() as $contact) {
    echo $contact['emailAddress'];
    // Process each contact across all pages automatically
}
```

**Supported methods:**

- `$sidemail->contacts->list()`
- `$sidemail->contacts->query()`
- `$sidemail->email->search()`
- `$sidemail->messenger->list()`

## Email methods

### Search emails

Searches emails based on the provided query and returns found email data. This endpoint is paginated and returns a maximum of 20 results per page. The email data are returned sorted by creation date, with the most recent emails appearing first. This endpoint supports [auto-pagination](#auto-pagination).

```php
$result = $sidemail->email->search([
    'query' => [
        'toAddress'     => 'john.doe@example.com',
        'status'        => 'delivered',
        'templateProps' => ['foo' => 'bar'],
    ],
]);

echo "Found emails: ";
print_r($result->data);
echo "Has more: " . ($result->hasMore ? 'true' : 'false');
```

### Retrieve a specific email

Retrieves the email data. You need only supply the email ID.

```php
$email = $sidemail->email->get('SIDEMAIL_EMAIL_ID');
echo "Email data: ";
print_r($email);
```

### Delete a scheduled email

Permanently deletes an email. It cannot be undone. Only scheduled emails which are yet to be send can be deleted.

```php
$response = $sidemail->email->delete('SIDEMAIL_EMAIL_ID');
echo "Email deleted: " . ($response->deleted ? 'true' : 'false');
```

## Contact methods

### Create or update a contact

```php
try {
    $response = $sidemail->contacts->createOrUpdate([
        'emailAddress' => 'marry@lightning.com',
        'identifier'   => '123',
        'customProps'  => [
            'name' => 'Marry Lightning',
            // ... more of your contact props ...
        ],
    ]);

    echo "Contact was '{$response->status}'.";
} catch (Sidemail\SidemailException $e) {
    // Uh-oh, we have an error! Your error handling logic...
    error_log($e->getMessage());
}
```

### Find a contact

```php
$response = $sidemail->contacts->find('marry@lightning.com');
```

### List all contacts

Lists all contacts in your project. This endpoint supports [auto-pagination](#auto-pagination).

```php
$result = $sidemail->contacts->list();

print_r($result->data);                       // array of contacts
echo $result->hasMore ? 'true' : 'false';     // boolean if more data
echo $result->paginationCursorNext;           // cursor for next page
```

### Query contacts (filtering)

Filter contacts. This endpoint supports [auto-pagination](#auto-pagination).

```php
$result = $sidemail->contacts->query([
    'limit' => 100,
    'query' => ['customProps.plan' => 'pro'],
]);

foreach ($result->autoPaginate() as $contact) {
    echo $contact->emailAddress;
}
```

### Delete a contact

```php
$response = $sidemail->contacts->delete('marry@lightning.com');
```

## Project methods

### Create a linked project

A linked project is automatically associated with a regular project based on the `apiKey` provided into `Sidemail`. To personalize the email template design, make a subsequent update API request. Linked projects will be visible within the parent project on the API page in your Sidemail dashboard.

```php
$response = $sidemail->project->create([
    'name' => 'Customer X linked project',
]);
// Important! Save $response->apiKey for later use
```

### Update a linked project

Updates a linked project based on the `apiKey` provided into `Sidemail`.

```php
$sidemail->project->update([
    'name' => 'New name',
    'emailTemplateDesign' => [
        'logo' => [
            'sizeWidth' => 50,
            'href'      => 'https://example.com',
            'file'      => 'PHN2ZyBjbGlwLXJ1bGU9ImV2ZW5vZGQi...', // base64 encoded image
        ],
        'font'   => ['name' => 'Acme'],
        'colors' => ['highlight' => '#0000FF', 'isDarkModeEnabled' => true],
        'unsubscribeText'          => 'Darse de baja',
        'footerTextTransactional'  => "You're receiving these emails because you registered for Acme Inc.",
    ],
]);
```

### Get a project

Retrieves project data based on the `apiKey` provided into `Sidemail`. This method works for both normal projects created via Sidemail dashboard and linked projects created via the API.

```php
$response = $sidemail->project->get();
```

### Delete a linked project

Permanently deletes a linked project based on the `apiKey` provided into `Sidemail`. It cannot be undone.

```php
$sidemail->project->delete();
```

## Messenger API (newsletters)

```php
$list = $sidemail->messenger->list(['limit' => 20]);
$messenger = $sidemail->messenger->get('messenger-id');
$created = $sidemail->messenger->create(['subject' => 'My Messenger', 'markdown' => 'Broadcast message...']);
$updated = $sidemail->messenger->update('messenger-id', ['name' => 'Updated name']);
$deleted = $sidemail->messenger->delete('messenger-id');
```

## Sending domains API

```php
$list = $sidemail->domains->list();
$domain = $sidemail->domains->create(['name' => 'example.com']);
$deleted = $sidemail->domains->delete('domain-id');
```

## More info

Visit [Sidemail docs](https://sidemail.io/docs/) for more information.
