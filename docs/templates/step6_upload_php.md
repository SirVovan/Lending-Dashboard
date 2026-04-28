## §UploadPHP — upload.php

```php
<?php
define('ADMIN_PASSWORD', 'CHANGE_ME_PASSWORD');
define('UPLOAD_DIR',     __DIR__ . '/images/uploads/');
define('UPLOAD_URL',     'images/uploads/');
define('MAX_FILE_SIZE',  5 * 1024 * 1024);
define('ALLOWED_MIME',   ['image/jpeg', 'image/png', 'image/gif', 'image/webp', 'image/svg+xml']);

header('Content-Type: application/json; charset=utf-8');
header('X-Content-Type-Options: nosniff');

if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    http_response_code(405);
    exit(json_encode(['ok' => false, 'error' => 'Method not allowed']));
}

$auth = isset($_POST['_auth']) ? $_POST['_auth'] : '';
if ($auth !== ADMIN_PASSWORD) {
    http_response_code(403);
    exit(json_encode(['ok' => false, 'error' => 'Unauthorized']));
}

if (!isset($_FILES['file']) || $_FILES['file']['error'] !== UPLOAD_ERR_OK) {
    $err = isset($_FILES['file']['error']) ? $_FILES['file']['error'] : 'no file';
    http_response_code(400);
    exit(json_encode(['ok' => false, 'error' => 'Upload error: ' . $err]));
}

$file = $_FILES['file'];

if ($file['size'] > MAX_FILE_SIZE) {
    http_response_code(413);
    exit(json_encode(['ok' => false, 'error' => 'File too large (max 5MB)']));
}

if (!function_exists('finfo_open')) {
    http_response_code(500);
    exit(json_encode(['ok' => false, 'error' => 'finfo extension not available']));
}
$finfo = finfo_open(FILEINFO_MIME_TYPE);
$mime  = finfo_file($finfo, $file['tmp_name']);
finfo_close($finfo);

if (!in_array($mime, ALLOWED_MIME, true)) {
    http_response_code(415);
    exit(json_encode(['ok' => false, 'error' => 'Invalid file type: ' . $mime]));
}

$ext_map = ['image/jpeg'=>'jpg','image/png'=>'png','image/gif'=>'gif','image/webp'=>'webp','image/svg+xml'=>'svg'];
$ext = $ext_map[$mime];

$filename = uniqid('img_', true) . '.' . $ext;
$dest     = UPLOAD_DIR . $filename;

if (!is_dir(UPLOAD_DIR)) {
    if (!mkdir(UPLOAD_DIR, 0755, true)) {
        http_response_code(500);
        exit(json_encode(['ok' => false, 'error' => 'Cannot create upload directory']));
    }
}

if (!move_uploaded_file($file['tmp_name'], $dest)) {
    http_response_code(500);
    exit(json_encode(['ok' => false, 'error' => 'Failed to save file']));
}

exit(json_encode(['ok' => true, 'path' => UPLOAD_URL . $filename]));
```
