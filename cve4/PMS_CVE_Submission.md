# CVE Submission: Stored XSS in PMS Mail and Feedback Functionality

## 1. Product and Scope
- Product: PMS (PHP-based Project Management System)
- Affected component: Faculty and Student mail modules, feedback submission, and related feedback display pages
- Affected pages:
  - [PMS/FACULTY/mail.php](PMS/FACULTY/mail.php)
  - [PMS/STUDENT/mail.php](PMS/STUDENT/mail.php)
  - [PMS/FACULTY/view.php](PMS/FACULTY/view.php)
  - [PMS/STUDENT/project.php](PMS/STUDENT/project.php)
- Severity: High

## 2. Vulnerability Description
PMS contains a stored cross-site scripting (XSS) vulnerability in its mail and feedback features. User-controlled content is written to the database and later rendered in HTML without output encoding. Because the application neither sanitizes input nor escapes output, an attacker can store a malicious payload that executes in the victim's browser when the message list or feedback page is opened.

The root cause is unsafe handling of untrusted content in both storage and presentation layers:
- message content is inserted into the database without filtering
- message content is echoed directly into HTML without `htmlspecialchars()` or equivalent encoding

## 3. Vulnerable Code
The vulnerability is caused by the following code paths in the mail modules and the feedback workflow.

### 3.1 Faculty mail module
In [PMS/FACULTY/mail.php](PMS/FACULTY/mail.php), the message body is written to the database and later echoed back into the page without output encoding:

```php
if (isset($_POST['submit']))
{
  $to=$_POST['student']; 
  $msg=$_POST['msg'];

  if (!empty($to))
  {
    $sql= "INSERT INTO `pmas`.`mail` (`mail_id`, `s_id`, `f_id`, `msg`) VALUES ('', '$to', '$user', '$msg');";
    mysqli_query($conn, $sql);
  }
}
```

```php
echo "<td>".$std['msg']."<td/>";
```

### 3.2 Student mail module
In [PMS/STUDENT/mail.php](PMS/STUDENT/mail.php), the same pattern exists in the student mail workflow:

```php
if (isset($_POST['submit']))
{
  $to=$_POST['to']; 
  $msg=$_POST['msg'];

  if (!empty($to))
  {
    $sql= "INSERT INTO `pmas`.`st_mail` (`s_mail_id`, `s_id`, `f_id`, `mag`) VALUES ('', '$user', '$to', '$msg');";
    mysqli_query($conn, $sql);
  }
}
```

```php
echo "<td>".$std['mag']."<td/>";
```

### 3.3 Feedback submission path
In [PMS/FACULTY/view.php](PMS/FACULTY/view.php), the feedback value is written into the project record without input sanitization or output encoding:

```php
$sql2= "UPDATE `pmas`.`project` SET `remark` = '$feed' WHERE `project`.`p_id` = '$prid';";
```

### 3.4 Feedback display page
In [PMS/STUDENT/project.php](PMS/STUDENT/project.php), the stored feedback value is rendered directly into a `<textarea>` without encoding:

```php
<textarea name="feedback" rows="5" cols="30" readonly="readonly" placeholder="FEEDBACK"><?php echo $std['remark'];?> </textarea>
```

## 4. Vulnerability Exploitation
The issue can be exploited by submitting a crafted HTML/JavaScript payload through the mail compose form or through the feedback submission path. When a user later views the sent mail, inbox, or feedback display page, the payload is rendered as active content and executes in the browser session.

This makes the issue suitable for:
- client-side code execution in the victim's browser
- UI redressing or phishing overlays
- session and action abuse if browser and cookie protections are weak

## 5. Proof of Concept
Payload:
```html
<img src=x onerror=alert(1)>
```

Reproduction steps:
1. Log in to PMS with a valid Faculty or Student account.
2. Open the mail compose page.
3. Place the payload above in the message field.
4. Submit the form to store the message.
5. Open the Sent Mail or Inbox view.

## 6. Result
The payload is stored successfully and later executed when the message is rendered in the mail view. In testing, the browser displayed an `alert(1)` dialog, confirming that the application renders stored HTML content without escaping it.

## 7. Conclusion
This issue is a stored XSS condition in the PMS mail and feedback workflow. The vulnerability is reproducible through user-controlled content, persists in the backend data store, and executes when the stored content is rendered back into the application interface.
