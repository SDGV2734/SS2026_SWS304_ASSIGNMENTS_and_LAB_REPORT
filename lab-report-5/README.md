# LAB REPORT 05

## Title: **Insecure Deserialization**

#### **Name:** Sonam Dorji Ghalley

#### **Student ID:** 02230299

#### **Course/Unit:** SWS304

## **1. Lab Objective**

The laboratory work concentrates on discovering and using Insecure Deserialization vulnerabilities which the TryHackMe Insecure Deserialization room demonstrates. Deserialization flaws occur when applications reconstruct objects from untrusted, attacker-controlled data without proper validation, which enables Remote Code Execution (RCE) attacks and authentication bypass and privilege escalation attacks.

## **2. Key Concepts**

Serialization transforms an object into a byte stream which can be stored or transmitted. The system uses deserialization to restore data to its original state. PHP uses its two main functions `serialize()` and `unserialize()` to handle data serialization tasks. When applications pass user-supplied serialized data to unserialize() without validation, attackers can manipulate object properties or inject malicious class instances to achieve arbitrary code execution through PHP magic methods such as `__wakeup()` and `__destruct()`.

## **3. Questions & Answers**

**Question 1: In PHP, what is the specific function used to convert an object into a storable byte stream?**

**Answer: serialize()**

The PHP `serialize()` function transforms objects and data structures into storable string representations that use byte streams. The function `unserialization()` enables users to restore the original object from its string representation. The mechanism enables insecure deserialization vulnerabilities because it allows user-provided data to be sent directly to `unserialize()` function.

**Question 2: When looking at the serialized note a:2:{s:5:"title";s:12:"My THM Note";...}, what does the a:2 at the beginning signify?**

**Answer: An array with 2 elements**

The PHP serialized format uses the code "a" to indicate an array type and the subsequent integer 2 to show the count of key-value pairs contained within. Other type identifiers include "s" for string, "i" for integer, "b" for boolean, and "O" for object.

**Question 3: Which PHP magic method is automatically invoked the moment an object is reconstructed from a string?**

**Answer: `__wakeup()`**

The `__wakeup()` function represents a PHP magic method which receives automatic invocation when the unserialize() function performs its object reconstruction process. The function serves as a primary exploitation target because attackers can create harmful serialized payloads which they know will trigger their inserted deserialization logic through the `__wakeup()` function.

**Question 4: A cookie value starts with TzoxMzo.... After Base64 decoding, it starts with O:8:". What programming language is likely being used?**

**Answer: PHP**

The "O:" prefix serves as the PHP serialized format identifier which denotes Object representation. "O:8:" signifies an object whose class name consists of 8 characters. This fingerprint serves as a unique identifier for PHP serialization which developers typically use in cookies and session data storage.

**Question 5: What common file extension suffix do attackers add to a PHP filename (e.g., index.php~) to look for leaked source code backups?**

**Answer: ~ (tilde)**

The text editors Vim and Emacs create automatic backup files through their system which appends a tilde character to the original file name. When web servers have their settings incorrectly established backup files become accessible to attackers who will download `index.php~` to obtain unprotected PHP source code which reveals system logic and login information and class design details.

**Question 6: Access http://MACHINE_IP/case1. Decode the cookie. What is the exact length of the user string value in the serialized object?**

**Answer: 4**

The cookie from /case1 is Base64 encoded. After decoding, the serialized PHP object contains a "user" key whose string value is "user" — 4 characters. In PHP serialization this appears as s:4:"user" where s:4 denotes a 4-character string.

**Question 7: After modifying the cookie to set isSubscribed to true and refreshing the page, what is the Flag displayed?**

**Answer: THM{c00k13_m4n1pul4t10n}**

The server gives premium access and shows the hidden flag after Base64 decoding the original cookie and changing the isSubscribed field from `b:0 (false)` to `b:1 (true)` and then Base64 encoding it again and updating the cookie value. The process shows how client-side trust failures exist because of insecure deserialization methods.

**Question 8: In the provided test.php file, which class contains the malicious exec() call within the __wakeup method?**

**Answer: MaliciousUserData**

The test.php file defines a class MaliciousUserData with a `__wakeup()` magic method containing an `exec()` call. When a serialized MaliciousUserData object is passed to `unserialize(), __wakeup()` fires automatically, executing arbitrary OS commands — a standard example of PHP Object Injection which results in Remote Code Execution.

**Question 9: When generating your reverse shell payload, what are the first 10 characters of the Base64 string generated for the MaliciousUserData object?**

**Answer: TzoxNjoi**

After crafting the MaliciousUserData serialized PHP object with a reverse shell payload in the `exec()` call, the string is Base64 encoded for delivery as a cookie. The Base64 encoding of the O:16:"MaliciousUserData":... object begins with “TzoxNjoi”, encoding the O:16: prefix.

**Question: Establish a Netcat listener and trigger the exploit. What is the content of /var/www/flag.txt?**

**Answer: THM{in53cur3_d3s3r14l1z4t10n}**

The process begins with Netcat listener installation by using `nc -lvnp <port>` command which establishes a listening point for incoming connections. The attacker then sends a Base64 cookie which has been specially designed to exploit the vulnerability in order to create a reverse shell connection. The attacker gains access to the final flag by using the cat command to read `/var/www/flag.txt` which confirms their successful execution of Remote Code Execution through insecure deserialization.

## **4. Summary & Lessons Learned**

**This lab demonstrated the real-world impact of insecure deserialization vulnerabilities in PHP web applications. Key takeaways:**

- Never pass user-supplied data directly to `unserialize()` without validation and integrity checks.
- PHP magic methods (`__wakeup, __destruct`) execute automatically and are prime RCE vectors in deserialization attacks.
- Cookies and session tokens containing Base64-encoded serialized objects must always be treated as attacker-controlled data.
- Source code backup files (e.g., `index.php~`) can expose class definitions enabling targeted gadget chain attacks.
- Mitigations: use signed/encrypted tokens (HMAC), avoid deserializing user input, use allowlists for class instantiation.