Semcms foreign trade website management system has a SQL injection vulnerability, allowing attackers to access sensitive information through this loophole.
Semcms portal website: http://www.sem-cms.com/
Affected versions: <=V4.8 PHP version (currently the latest)
Affected plugin: Backend management interface -> Comprehensive management module -> User management -> Edit
Source code download link: SemCms PHP, ASP version foreign trade website [multilingual] source code free download - SemCms foreign trade website management system (sem-cms.com)
FOFA space mapping: app="SEMCMS"
![image](https://github.com/gatsby2003/Semcms/assets/135791683/c1c5f490-e1ca-4fbd-84ef-b0a3b9bc2a56)
Code audit process:
1. In the include directory of the project source code, there is a file named control.php. Searching for the inject_check_sql function in this file reveals that it is a global SQL query string filtering function. It uses regular expressions to match a series of commonly used characters in SQL statements: preg_match('select|and|insert|=|%|<|between|update|\'|\*|union|into|load_file|outfile/i',$sql_str);
Keywords like select, and, insert, %, =, <, between, update, *, union, into, ', etc., are all filtered. All SQL query string parameters are first passed to the $str variable and then called by the verify_str() function. Within the verify_str() function, the inject_check_sql() function is called for validation. This serves as a global SQL statement validator.
![image](https://github.com/gatsby2003/Semcms/assets/135791683/0a1ee01e-b5dc-4ab3-b05d-2cec03af8f81)
2. In the Admin directory of the project source code, there is a file named SEMCMS_User.php, corresponding to the user management module in the backend. When $type="edit", it is for editing user content. In this scenario, a SQL statement is found: $row = mysqli_fetch_array($db_conn->query("SELECT * FROM sc_user WHERE ID=".$_GET["ID"]));
This SQL statement directly retrieves the ID parameter from the GET request and queries all content based on the ID parameter to fetch the result.
![image](https://github.com/gatsby2003/Semcms/assets/135791683/5007ae1e-ebf9-461e-b15e-1b8741bf043b)
Furthermore, the retrieved data is used to populate and render the table content based on field names. Since the SQL statement only undergoes blacklisting filtering through verify_str() calling inject_check_sql(), and there is no other SQL injection protection in place, and no quote construction closure, there is a high possibility of SQL injection vulnerability. Unfortunately, filtering out characters like select, union, and, = prevents traditional SQL injection validation from being performed from a verification perspective.
![image](https://github.com/gatsby2003/Semcms/assets/135791683/39b30337-e16d-4c96-8cae-d519d0b5452b)
We already know that it returns the content fields of an existing user, and the user's content fields are influenced by the ID parameter. When the ID is fixed, the content fields will not change. We can manipulate the ID parameter to display the content of a specific user and then use functions like if(), length() to manipulate the database length and name for SQL injection. Below is the local penetration testing process.

Case Reproduction:
Local Environment: gatsby.com (local domain)
Firstly, access the backend management interface, go to the user management section under the comprehensive management module, click on edit, and intercept the request using a packet sniffer.
![image](https://github.com/gatsby2003/Semcms/assets/135791683/665d88bf-1ec7-499e-a0fd-d1ea936ab0dd)

GET /semcms/admin/SEMCMS_User.php?type=edit&ID=4&page=1 HTTP/1.1
Host: gatsby.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:122.0) Gecko/20100101 Firefox/122.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate, br
Connection: close
Referer: http://gatsby.com/semcms/admin/SEMCMS_User.php
Cookie: scusername=%E6%80%BB%E8%B4%A6%E5%8F%B7; scuseradmin=Admin; scuserpass=21232f297a57a5a743894a0e4a801fc3
Upgrade-Insecure-Requests: 1

To validate our hypothesis, we can manipulate the ID parameter with different values and observe how the user content changes accordingly. Since the parameter being passed is numeric and interacts with the database, we can test for SQL injection vulnerabilities by providing different ID values and analyzing the response to confirm if the application is susceptible to SQL injection attacks.
![image](https://github.com/gatsby2003/Semcms/assets/135791683/ab4aac3d-54f4-4421-8ffa-6e22274b8779)
![image](https://github.com/gatsby2003/Semcms/assets/135791683/cc9d573d-f703-4391-ae84-2c13d44c44a5)
Perform SQL injection next, knowing that the default database for the installed CMS is 'mycms'.
Verify that the current database length is 5, so that the ID of the expression = 4, and the return result corresponds to the personal user information of the SEC user with ID=4
Payload: ID=(length(database())-1) 
If the personal user information with user ID=4 is successfully returned, then we can judge (length(database())-1)=4 to get length(database())=5 and the database name length is 5 when "=" is disabled
![image](https://github.com/gatsby2003/Semcms/assets/135791683/267ccc3b-00e6-46a7-a76c-de29176389b0)
Then, you can determine the database name, and you can use regular matching to determine it one by one
Payload: ID=if(database()+RLIKE+"^m{1}",4,3)
When the first character of the database name is m, ID=4 is returned, otherwise ID=3, ID=3 and ID=4 are returned.
![image](https://github.com/gatsby2003/Semcms/assets/135791683/93e9d391-4e6f-412f-888a-0b240013de9e)
The same goes for the second character of the database
Payload:ID=if(database()+RLIKE+"^m[y]{1}",4,3)
Here the database name is used to match a string that starts with m and the second character is y, and the matching success returns ID=4, otherwise ID=3, and when the match is successful, ID=4 is returned
![image](https://github.com/gatsby2003/Semcms/assets/135791683/ace89938-3068-484c-b5d6-292441e90be3)
If you change the payload to match a string with the database name starting with m and the second character being a, the matching failure returns ID=3 user content

![image](https://github.com/gatsby2003/Semcms/assets/135791683/375e2251-adf1-4ae5-ab04-391c8b9dcc08)

