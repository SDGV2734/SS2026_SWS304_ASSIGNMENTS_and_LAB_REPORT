# Assignment 4

$$
\huge \begin{array}{c} \mathbf{\textsf{Royal University of Bhutan,}} \\ \mathbf{\textsf{College of Science and Technology}} \end{array}
$$

$$
\Large \mathbf{\textsf{SWS304: Advanced Web Attacks and Exploitation  }}
$$

$$
\Large \mathbf{\textsf{Computing Technologies Department  }}
$$

$$
\large \mathbf{\textsf{Assignment 4  }}
$$

$$
\large \mathbf{\textsf{Submitted by: Sonam Dorji Ghalley  }}
$$

$$
\large \mathbf{\textsf{Student No: 02230299  }}
$$

# Prototype Pollution Vulnerability Assessment and Exploitation Report

## Introduction

A flaw in JavaScript called **prototype pollution** lets hackers tweak how objects get their traits via inheritance. Because every object draws abilities from Object.prototype, sneaky changes there ripple outward silently. When altered by bad actors, these base features might twist app logic without clear signs. Effects stretch differently based on setup - sometimes access gets wider than intended. In some cases, manipulated prototypes open doors to injected scripts right into web pages. Unexpected control shifts could let users do things they should never reach. Servers may fall at risk when core assumptions about data break behind the scenes.

Four hands-on labs from PortSwigger Web Security Academy form the core of this work, exploring how prototype pollution can happen in browsers or on servers. Each step taken is laid out here - how tests were run, what showed up during attempts, ways flaws were triggered. Ways to fix such issues appear later, drawn from what emerged during testing. Method comes before results, just like confusion often follows code that trusts too much. What breaks may also reveal where defenses are thin. Insight grows not from theory alone but from poking real systems and watching them bend. Examples guide the reader without spelling every click or keystroke. Seeing matters more than assuming when tangled logic hides in object chains. Solutions fit the specific cracks found, not vague best practices pulled from guides. Details stay close to lab conditions, since behavior shifts outside controlled setups.

## Lab 1 — Client-Side Prototype Pollution via Browser APIs

#### Objective

To demonstrate how insecure processing of URL parameters can modify `Object.prototype` and affect application behaviour.

#### Walkthrough

This is how the website looks like upon accessing 

![image.png](Assignment%204/image.png)

As a normal user I went through the website and i could see a search option on the top of the website. So I tried searching something like “test” to see how the website behaves

![image.png](Assignment%204/image%201.png)

I got nothing here so i did the following to check from the from the guide

I tried adding the test payload from to the URL query string to check if i can add arbitrary property into the object.prototype :`/?__proto__[foo]=bar` 

![image.png](Assignment%204/image%202.png)

after entering the URL i went to the DEVtool and went to the console tab. Than I typed `object.prototype`  and hit enter. 

![image.png](Assignment%204/image%203.png)

on the output I can see  the new property “foo” with the value “bar” , So this confirm that it has  prototype pollution source. 

I went to the source tab and found `searchLoggerConfigurable.js`  file and something interesting caught my eye in the code where it has a `transport_url`  property and it creates a new `<script>` element and uses that URL as the src.

![image.png](Assignment%204/image%204.png)

It turned out the transport_url was already there in the config object, making direct changes appear impossible. Yet upon closer inspection, the clue lay in how Object.defineProperty() had been applied. The property wasn’t just present - it was fixed as neither writable nor configurable. Strangely enough, though, no actual value had ever been assigned to it. Only its behavior was sealed off, leaving the field empty but untouchable.

When no value showed up in the property descriptor, I thought - maybe slipping a value into Object.prototype would work. Tried it by reaching through: `/?**proto**[value]=world`. Would it catch that change? Turned out, yes

i checked inspect and saw that a `<script>`  element was had appeared on the src page. this means that the browser was reading the value from the pollution prototype.

![image.png](Assignment%204/image%205.png)

Now I just needed to turn this command to actual XSS payload by replacing the “world” to: 
`1.  /?__proto__[value]=data:,alert(1);` 

Observing that the alert(1) is called and the lab is now solved.

![image.png](Assignment%204/image%206.png)

![image.png](Assignment%204/image%207.png)

#### Summary

A lab test showed how bad data from web addresses can mess up an app's core structure. When a harmful key slips into the main object template, every object on the page picks it up automatically. Real apps might run wrong code, change webpage elements unexpectedly, or even let attackers steal accounts. The flaw spreads silently through normal object links in JavaScript. Attackers don’t need special access - just a working link. Once inside, their changes echo everywhere objects are used. This kind of bug hides well but triggers big failures. Every script trusting default object setups becomes part of the problem. Damage depends on what the injected property overrides. Some cases break features quietly; others open full control paths. Testing reveals these issues only when edge inputs get checked. Most detection tools miss such subtle contamination during scans. Fixing it means blocking unwanted writes at the root level. Default safeguards in modern browsers help - but rarely stop everything. Developers must assume any incoming value carries risk. Treating URLs as untrusted sources stops most attacks early. Even small scripts face danger if they process query strings loosely. Without strict checks, one flawed merge can unravel safety layers.

#### **Questions**

**Question 1** 

Why does adding ? __proto__[x]=y to the URL cause Object.prototype to be modified — and not
just the single object created from the URL parameters? Explain using your knowledge of the
prototype chain from Chapter 6.

- The **proto** property directly references an object's prototype. When user input writes values into **proto**, JavaScript updates Object.prototype instead of creating a normal object property. Since all ordinary objects inherit from Object.prototype, the polluted property becomes accessible across the entire application.

**Question 2**

Name ONE defence a developer could add to the JavaScript code to prevent this attack, and
explain in 2 sentences how it would stop the pollution

- Developers should implement input validation and reject dangerous keys such as **proto**, constructor, and prototype. Another secure approach is creating objects using Object.create(null), which removes prototype inheritance and prevents pollution.

## Lab 2 — DOM XSS via Client-Side Prototype Pollution

#### Objective

To demonstrate escalation of prototype pollution into DOM-based Cross-Site Scripting.

#### Walkthrough

This is what the website looks like when accessing the lab 

![image.png](Assignment%204/image%208.png)

So i started with checking the prototype pollution source cusing the same query in the earlier lab:

 **`/?__proto__[foo]=bar`** 

![image.png](Assignment%204/image%209.png)

Same like earlier I went back to the console tab and typed `object.prototype` and as expected it it showed a “foo ” property with the value “bar” was there

![image.png](Assignment%204/image%2010.png)

And went to the source tab looked through the javascript files. Where i saw the `searchLogger.js`  file. This file has a `transport_url`  property and creates a script element and sets its src to that value.

![image.png](Assignment%204/image%2011.png)

Now tried polluting the `object.prototype` with the `transport_url` by setting to a test value: 
`/?__proto__[transport_url]=foo` 

And moving back to the inspector mode we can see the script tag had been injected with src = “foo” meaning i can replace this with `/?__proto__[transport_url]=data:,alert(1);` to trigger a XSS 

![image.png](Assignment%204/image%2012.png)

with this we can see the alert on the screen and also solving the lab

![image.png](Assignment%204/image%2013.png)

![image.png](Assignment%204/image%2014.png)

#### Summary

The incident unfolded across two phases. Not long after, someone manipulated Object.prototype through data supplied by users. Right afterward, a DOM endpoint picked up the tainted property and added it directly into the webpage, running arbitrary script. Only then did it become clear - flawed object handling could lead to active browser exploits without direct injection.

#### Question

**Question 1** 

Why would injecting the XSS payload directly into the sink (without prototype pollution) NOT
work in this lab? What role does the prototype chain play in connecting your injected value to the
DOM sink?

- The incident unfolded across two phases. Not long after, someone manipulated `Object.prototype` through data supplied by users. Right afterward, a DOM endpoint picked up the tainted property and added it directly into the webpage, running arbitrary script. Only then did it become clear - flawed object handling could lead to active browser exploits without direct injection.

**Question 2**

A developer adds DOMPurify to sanitise all innerHTML assignments. Does this fully fix the
vulnerability? What else needs to be fixed, and where?

- DOMPurify reduces XSS risk but does not fully solve prototype pollution. Developers must also remove unsafe object merging and validate user input before object assignment.

## Lab 3 — Privilege Escalation via Server-Side Prototype Pollution

#### Objective

To exploit server-side prototype pollution and gain administrative privileges.

#### Walkthrough

This how the lab looks like 

![image.png](Assignment%204/image%2015.png)

As I was exploring the website i saw login page so I tried logging with the provided credential `wiener:peter` in the lab  

![image.png](Assignment%204/image%2016.png)

![image.png](Assignment%204/image%2017.png)

So i opened burpsuite and intercepted this request saw the following 

![image.png](Assignment%204/image%2018.png)

nothing interesting here but after i login i could see the a bill and delivery form 

![image.png](Assignment%204/image%2019.png)

I filled this form and intercepted the request on submit and got the following on burpsuite 

![image.png](Assignment%204/image%2020.png)

found the POST /my-account/change-address and looking at the request body, I could see the address fields were being sent as a JSON and the response then came back with a JSON showing the user account, and I could see it had been updated to reflect the new address.

In Repeater, I added a `__proto__` key to the JSON body with an arbitrary property, so that i can see whether the prototype pollution was possible or not:

`"__proto__": { "foo":"bar" }`

The response showed `foo: "bar"` inside the user JSON object even though there was no `__proto__` property. This indicates prototype pollution was successful, as the property was inherited from `Object.prototype` and appeared in the object automatically.

![image.png](Assignment%204/image%2021.png)

Now after looking at the response carefully, i saw the isAdmin property, which was set to false for my account. I changed the payload to inject `isAdmin: true` through the prototype:
`"**proto**": { "isAdmin": true }`

The response shows `isAdmin: true` in the user object. The server was reading isAdmin from the polluted prototype because the user object had no isAdmin property.

![image.png](Assignment%204/image%2022.png)

which in returned me solving the lab

![image.png](Assignment%204/image%2023.png)

#### Summary

Server-side prototype pollution occurred because JSON input was merged into server-side objects without validating prototype properties. The injected isAdmin attribute became inherited by newly created objects, allowing privilege escalation.

#### Questions

**Question 1**

After sending the `__proto__` payload once, the pollution affected ALL objects created in the
Node.js process - including objects for other users' requests. Why? Use the prototype chain to
explain what happened inside the server's JavaScript runtime.

- Node.js applications typically run inside a long-lived JavaScript process where many requests are handled using the same runtime environment. Because JavaScript objects inherit properties through the prototype chain, modifying `Object.prototype` affects not only the current request but also objects created afterward. Once Object.prototype becomes polluted, newly created objects may automatically inherit attacker-controlled properties even if those objects were never directly modified. This can lead to persistent privilege escalation, altered application logic, unexpected behaviour across different users, and security controls being bypassed. The polluted state often remains active until the server process is restarted or the application explicitly removes the malicious properties.

**Question 2**

What is the correct code-level fix for the vulnerable merge function on the server? Write a short
example or description of how the fix would block the `__proto__`  key.

- The server should reject reserved keys and replace unsafe merge functions with secure alternatives.
- example

```
if(key==="proto" || key==="constructor" || key==="prototype"){
return;
}
```

## Lab 4 — Bypassing Flawed Input Filters

#### Objective

To bypass inadequate filtering and demonstrate alternative prototype manipulation.

#### Walkthrough

Upon accessing the lab like the previous lab i went to the login page and logged in using the credentials given 

![image.png](Assignment%204/image%2024.png)

Than i was directed to this page to fill in the form 

![image.png](Assignment%204/image%2025.png)

than like usual submitted the form but intercepted using burpsuite

![image.png](Assignment%204/image%2026.png)

I send this request to the repeater and  than send it and got the response as following 

![image.png](Assignment%204/image%2027.png)

I could see the user object including an isAdmin property set to false.

than i modified the JSON objects and added a new property named  `__proto__` containing an object with a json spaces property.

`"__proto__": {  "json spaces":10 }`

![image.png](Assignment%204/image%2028.png)

I went to the raw tab in the response section and checked that the JSoN indentation seemed unaffected.

![image.png](Assignment%204/image%2029.png)

So i modified the request to pollute the prototype thorough the constructor property instead: 

```json
"constructor": {
    "prototype": {
        "json spaces":10
    }
}
```

and on send the this newly modified request and viewing the response in the raw tab i noticed that JSON indentation have changed based on the value of injected property. Which shows that polluting the prototype is successful 

![image.png](Assignment%204/image%2030.png)

So now i used the same JSON object but changed the value to `"isAdmin": true`  

and the in the response we can see that now the admin is true. and solving the lab 

```json
"constructor": {
    "prototype": {
        "isAdmin":true
    }
}
```

![image.png](Assignment%204/image%2031.png)

![image.png](Assignment%204/image%2032.png)

#### Summary

Filtering only the **proto** property failed because constructor.prototype provides another route to Object.prototype. Attackers can exploit alternative object paths when protections rely only on blocklists.
****

#### Questions

**Question 1**

Explain WHY the path constructor.prototype leads to the same Object.prototype as `__proto__`
Use the prototype chain diagram from Chapter 6 to support your answer.

- Every JavaScript object contains a constructor reference which points back to Object. Object.constructor.prototype resolves to Object.prototype and therefore reaches the same inheritance structure.

**Question 2**

The developer now also blocks the key 'constructor'. Is the application fully protected? Describe
what a truly robust fix looks like — not just a blocklist approach.

- Blocking constructor alone is insufficient. A robust defence requires allowlisting inputs, safe merge functions, schema validation, immutable object patterns, and creating objects without prototypes.

## Conclusion

Something showed up during this task - how messing with prototypes shakes both browsers and servers. Not just one trick hides here; it opens doors through hands-on labs where small changes ripple into big trouble. Picture a chain snapping: when inherited traits get rewritten, apps start acting odd without warning. That shift? It feeds flaws like sneaky script injections in web pages, jumps to higher permissions, slips past locked features, even dodges half-baked safeguards meant to stop harm. Each lab peeled back layers - the same flaw wearing different masks across systems. What looked isolated turned out linked by weak handling of objects in code.

From the front-end tests came proof: when user data gets handled carelessly, it can twist internal structures and shift how browsers act during a full session. Not direct manipulation, yet still - through subtle influence - attackers found paths to run unwanted scripts. On the back end, things got heavier. Altered templates stuck around inside running services, messing up who was allowed what, letting outsiders step into higher roles. Broken checks opened doors they should never reach.

One key takeaway from these tests: basic filters alone won’t stop prototype pollution. When usual targets like proto get blocked, attackers still find ways - constructor.prototype might be open. A single barrier rarely holds. Stronger defense means stacking strategies - not just checking inputs carefully but also controlling how objects merge. Enforce clear schemas. Only permit known safe properties. Try building objects differently, say with Object.create(null), to cut risk at the root.

This task gave hands-on practice spotting, using, and grasping prototype pollution flaws in current web apps. Learning these details highlights why safe coding matters, along with early testing, while building software. Stronger defenses come from catching issues before they spread. Weak spots in prototypes can lead to bigger problems if ignored. Building with care helps block common attack paths. Seeing how damage unfolds makes prevention feel more urgent. Small mistakes in code structure may open wide doors later. Testing early shapes how solid the final product becomes. Awareness of such risks changes how developers approach design choices. Each step forward in skill tightens protection across systems.