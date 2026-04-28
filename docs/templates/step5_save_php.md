## §SavePHP — save.php

```php
<?php
define('ADMIN_PASSWORD', 'CHANGE_ME_PASSWORD');
define('CONTENT_FILE',   __DIR__ . '/content.json');
define('BACKUP_FILE',    __DIR__ . '/content.json.bak');
define('MAX_SIZE',       512 * 1024);

header('Content-Type: application/json; charset=utf-8');
header('X-Content-Type-Options: nosniff');

if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') exit;
if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    http_response_code(405);
    exit(json_encode(['ok' => false, 'error' => 'Method not allowed']));
}

$raw = file_get_contents('php://input');

if (strlen($raw) > MAX_SIZE) {
    http_response_code(413);
    exit(json_encode(['ok' => false, 'error' => 'Payload too large']));
}

$data = json_decode($raw, true);
if (json_last_error() !== JSON_ERROR_NONE) {
    http_response_code(400);
    exit(json_encode(['ok' => false, 'error' => 'Invalid JSON: ' . json_last_error_msg()]));
}

$auth   = isset($data['_auth'])   ? $data['_auth']   : '';
$action = isset($data['_action']) ? $data['_action'] : 'save';

if ($auth !== ADMIN_PASSWORD) {
    http_response_code(403);
    exit(json_encode(['ok' => false, 'error' => 'Unauthorized']));
}

if ($action === 'auth') exit(json_encode(['ok' => true]));

if ($action === 'save') {
    unset($data['_auth'], $data['_action']);

    if (!isset($data['_meta'])) {
        http_response_code(400);
        exit(json_encode(['ok' => false, 'error' => 'Missing _meta key']));
    }

    $json = json_encode($data, JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT | JSON_UNESCAPED_SLASHES);
    if ($json === false) {
        http_response_code(500);
        exit(json_encode(['ok' => false, 'error' => 'JSON encode failed']));
    }

    $tmp = CONTENT_FILE . '.tmp.' . uniqid();

    if (file_put_contents($tmp, $json, LOCK_EX) === false) {
        http_response_code(500);
        exit(json_encode(['ok' => false, 'error' => 'Write error (check file permissions)']));
    }

    if (file_exists(CONTENT_FILE)) copy(CONTENT_FILE, BACKUP_FILE);

    if (!rename($tmp, CONTENT_FILE)) {
        @unlink($tmp);
        http_response_code(500);
        exit(json_encode(['ok' => false, 'error' => 'Rename failed']));
    }

    exit(json_encode(['ok' => true]));
}

http_response_code(400);
exit(json_encode(['ok' => false, 'error' => 'Unknown action: ' . $action]));
```
