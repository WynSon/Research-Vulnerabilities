#                                                           Insecure Deserialization Vulnerability

## 1. What is serialization and deserialization ?

__Serialization__ refers to a process of converting an object into a format which can be persisted to disk ( for example saved to a file or database), sent it through streams or over network.
The format in which an object is serialization into, can either be binary or structured text ( XML, JSON, YAML, ..etc ). Json and XML are two of the most commonly used serialization formats within webapp.

__Deserialization__ on the other hand, is the opposite of serialization, that is transforming serialized data coming from a file, steam or network socket into an object.

![image](https://user-images.githubusercontent.com/83699106/132938468-cc0f81cc-9f9b-49ec-ba16-f38c67570f37.png)

Many  programing languages offer native support for serialization and How objects are serialized rely on its language. In some languages such as Ruby and Python, serialization maybe referred to as marshalling(Ruby) and pickling (Python).


## 2. What is insecure deserialization ?

__Insecure Deserialization__ is a web vulnerability ranked 8th in top 10 OWASP 2017, frequently occur when user-controllable data is deserialized by a website. This potentially enable an attacker to manipulate serialied objects in order to pass harmful data into the application code.

![image](https://user-images.githubusercontent.com/83699106/132939080-d5d8c54f-9543-4f63-a3b3-3dcedebd83d8.png)


This data is after deserialized can take advantage to change the process flow of application.

## 3. What is the impact of insecure deserialization ?

The impact of this vulnerability can be very severe because it provide an entry point to a massively increased attack surface. It allows an attacker to reuse existing application code in harmful ways, resulting in numerous other vulnerabilities such as:

__RCE__

__Path Traversal__

__Arbitrary file read__

__SQL injection__

__Privilege escalation__

__Dos attacks__

__...etc..__.

## 4. How to indentify  insecure deserialization vulnerability ?

General speaking, insecure deseialization is relatively simple to detect regardless of whether whitebox or blackbox testing.
You should look at all data being passed into the website and try to indentify anything that looks like serialized data. It can be identified relatively easily if you know the format that different languages use.
After can identify serialized data, let's check whether you can able to control it.


## 5. Exploit insecure deserialization

__Some common way to exploit__

__1. Modifying object attributes__

Consider a website that uses a serialized User object to store data about a user's session in a cookie. An attacker might decode it to find the following byte stream:

```
0:4:"User":2:{s:4:"name"s:6:"spider";s:7:"isAdmin":b:0;}

```
The `isAdmin` attr is an abvious point of interest. An attcker can modify this attribute into `1` and re-encode then overwrite their current cookie value.
Assume the website uses this cookie to identify permission of a user to access special resource and functionally.

```
$user = unserialize($_COOKIE);
if ($user->isAdmin === true){
// allow access permission }

```

This is a simple way to exploit, but isn't common in the wild.

__2. Modify data types__

PHP has a loose comparision operator "__==__" because it only compare value but not check the data type. if you perform a loose comparison between an integer and a string, PHP will attempt to convert the string to an integer.
For example:

```
5 == "5 days " # return true
0 == "not a number " # return true

Assume a case if application use loose comparison operator to authencation and attacker modified datatype to number with 0 as value:
$login = unserialize($_COOKIE)
if($login['password'] == $passwd){
//if $passwd doesn't start with a number, it will return true and sucessfully 
}

```

__3. Magic Methods__

In PHP, It has "__Magic Methods__" are automatically invoke whenever a particular event or scenario occurs. Magic method are common feature of OOP programing in various languages. They are sometime indicated by prefixing or surrounding the method name with double-unserscores.

__*Some common magic methods*__
![image](https://user-images.githubusercontent.com/83699106/132940473-9c157e9b-f7b3-4983-93d1-ffd997d81b86.png)

Magic methods are widely used and do not represent a vulnerability on their own. But they can become dangerous when the code that they execute handles attacker-controlable data .

For example, __PHP's unserialize()__ method looks for and invokes an objct's __\__wakeup()__ magic method

__4. Injecting arbitrary objects__

Injecting arbitrary object types can open up many more possibilities.

In OOP, the methods available to an object are determined by its class. Therefore, f an attacker can manipulate which class of object is being passed in as serialized data, they can influence what code is executed after, and even during. deserialization.

Deserialization methods don't typically check what they are deserializing. This mean that you can pass in objects of any serializable class that is available to the website and the object will be deserializd.

If an attacker has access to the source code, they can study all of the availabe classes in detail. To construct a simple exploit, they would look for classes containing deserialization magic methods, then check whether any of them perform dangerous operations on controllable data. The attacker an then pass in a object of this class to use its magic method for an exploit.

__5. Gadget chains__

![image](https://user-images.githubusercontent.com/83699106/132945197-d54f18ba-c356-4582-bcba-970374685598.png)


A __"gadget"__ is a snippet of code that exists in the application that can help an attacker to achieve a particular goal. An individual may not directly do anything harmful with user input. However, the attacker's goal might simply be to invoke a method that will pass their input into another gadget. By changing multiple gadgets together in this way, an attacker can potentially pass their input into a dangerous "sink gadget", where it can cause maximun damage.


__6. PHAR deserialization__

You can exploit deserialization vulnerabilities where the website explicitly deserializes user input. However, in PHP it is sometimes possible to exploit deserialization even if there is no obvious use of the __unserialize()__ method.

PHP provides several URL-style wrappers that you can use for handling different protocols when accessing file paths. One of these is the phar:// wrapper, which provides a stream interface for accessing PHP Archive (.phar) files.

![image](https://user-images.githubusercontent.com/83699106/132945156-b142c8f4-0589-4f1e-b61f-f48dbd19a9f7.png)

The PHP documentation reveals that PHAR manifest files contain serialized metadata. Crucially, if you perform any filesystem operations on a phar:// stream, this metadata is implicitly deserialized. This means that a phar:// stream can potentially be a vector for exploiting insecure deserialization, provided that you can pass this stream into a filesystem method.

In the case of obviously dangerous filesystem methods, such as include() or fopen(), websites are likely to have implemented counter-measures to reduce the potential for them to be used maliciously. However, methods such as file_exists(), which are not so overtly dangerous, may not be as well protected.

This technique also requires you to upload the PHAR to the server somehow. One approach is to use an image upload functionality, for example. If you are able to create a polyglot file, with a PHAR masquerading as a simple JPG, you can sometimes bypass the website's validation checks. If you can then force the website to load this polyglot "JPG" from a phar:// stream, any harmful data you inject via the PHAR metadata will be deserialized. As the file extension is not checked when PHP reads a stream, it does not matter that the file uses an image extension.

As long as the class of the object is supported by the website, both the __wakeup() and __destruct() magic methods can be invoked in this way, allowing you to potentially kick off a gadget chain using this technique.

# 6. Insecure deseialization in PHP

In __PHP__, Serialization and Deserialization are supported through 2 method __serialize()__ and __unserialize()__

__1. serialized():__ Converting data input (__object__) into a __string__ stored it.

![Serialize method](https://user-images.githubusercontent.com/83699106/132939741-744c0022-9028-4243-acc1-3097272b20a4.png)

__*For example*__ Consider User Object has the attributes:

```
$user->name = "spider";
$user->isAdmin = true;
```
when serialized, it may look something like this:
```
0:4:"User":2:{s:4:"name"s:6:"spider";s:7:"isAdmin":b:1;}
```
This string above represent the information below:

`O:4;"User":2:{}`: Object has 4 character named __User__ and 2 attribute.

`s:4:"name"` :First attribute is string has 4 character named "__name__"

`s:6:"spider"`: The value of first attr is string with 6 character "__spider__"

`s:7:"isAdmin":b:1`: Second attr is string with 7 character named __isAdmin__ and has value type __boolean__ is 1.

__Some common format in use__

![Some common format in use](https://user-images.githubusercontent.com/83699106/132940077-0a7f12ba-a042-4929-aef1-85effba1242f.png)

__2. unserialize():__ Input data is a string of character and convert it into original Object.

![image](https://user-images.githubusercontent.com/83699106/132940297-e36006a0-e8db-492e-8cf5-b244addb4382.png)

__POP - Property Oriented Programming __
 
This techniques takes advantage gadget chains and combine them together into a completely payload used to exploit.

For example:

![image](https://user-images.githubusercontent.com/83699106/132945412-15af12bf-9afd-455e-9203-4c56ef7857b4.png)

To perform a successfully insecure deserialization attack lead to RCE in PHP. Ensure two following condition:

- Object-controlable has used magic function
- Gaget chains available in source code to control __unserialize()__ process.

# 7. Insecure Deserialization in JAV

Some languages such as Java use binary serialization formats. This make more difficult to read, but you can still identify serialized data if you can know how to recognize a few tell-tale signs.

Eg: Serialized Java objects always begin with the same bytes, which are encode as `ac ed` in Hex and `ro0` in base64.

Any class implements java.io.Serializable interface can be serialized and deseialized. If yo have source code access, take note of any code that uses the readObject() method, which is used to read and deserialized data from an Inputstream


__*Continue ................*__
__*..............................*__



# 8. Mitigation

- Maximum restriction use __unserialize__ with data input from untrust user
- Using XML, JSOn as alternative
- Using Whitelist with objects are allowed

# Reference

[Web Security Academy](https://portswigger.net/web-security/deserialization/exploiting#how-to-identify-insecure-deserialization)

[VNPT Security](https://medium.com/@vnptsec/insecure-deserialization-8b6594cef727)

[Acunetix](https://www.acunetix.com/blog/articles/what-is-insecure-deserialization/)

[Blog](https://blog.gypsyengineer.com/en/security/detecting-jackson-deserialization-vulnerabilities-with-codeql.html)
