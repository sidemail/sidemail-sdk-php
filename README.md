# Sidemail PHP library

The Sidemail PHP library provides convenient access to the Sidemail API from applications written in PHP.

## Requirements

- PHP 8.1+
- `ext-curl`

## Installation

Install this package with:

```bash
composer require sidemail/sidemail
```

## Usage

First, the package needs to be configured with your project's API key, which you can find in the Sidemail Dashboard after you signed up.

Initiate the SDK:

```php
<?php

require __DIR__ . '/vendor/autoload.php';

use Sidemail\Sidemail;

// Create Sidemail instance and set your API key.
$sidemail = new Sidemail(apiKey: 'xxxxx');
```

Then, you can call `$sidemail->sendEmail` to send emails like so:

```php
$sidemail->sendEmail([
    'toAddress'     => 'user@email.com',
    'fromAddress'   => 'you@example.com',
    'fromName'      => 'Your app',
    'templateName'  => 'Welcome',
    'templateProps' => ['foo' => 'bar'],
]);
```

The response will look like this:

```json
{
  "id": "5e858953daf20f3aac50a3da",
  "status": "queued"
}
```

Learn more about Sidemail API:

- [See all available API options](https://sidemail.io/docs/send-transactional-emails#discover-all-available-api-parameters)
- [See all possible errors and error codes](https://sidemail.io/docs/send-transactional-emails#api-errors)

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

## Auto-pagination

The SDK provides automatic pagination for list and search endpoints that return paginated results. This allows you to iterate through all results without manually handling pagination cursors.

```php
$result = $sidemail->contacts->list();

foreach ($result->autoPaging() as $contact) {
    echo $contact['emailAddress'];
    // Process each contact across all pages automatically
}
```

**Supported methods:**

- `$sidemail->contacts->list()`
- `$sidemail->email->search()`

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

### Delete a contact

```php
$response = $sidemail->contacts->delete('marry@lightning.com');
```

## Project methods

### Create a linked project

A linked project is automatically associated with a regular project based on the `apiKey` provided into `Sidemail`. To personalize the email template design, make a subsequent update API request. Linked projects will be visible within the parent project on the API page in your Sidemail dashboard.

```php
// Create a linked project && save API key from $response->apiKey to your datastore
$response = $sidemail->project->create([
    'name' => 'Customer X linked project',
]);

// $user->save(['sidemailApiKey' => $response->apiKey]) ...
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

## More info

Visit [Sidemail docs](https://sidemail.io/docs/) for more information.
