# The Tangled Web - A Guide to Securing Modern Web Applications

**Author:** Michal Zalewski

<p align="left">
  <img src="./images/the-tangled-web.jpg" width="200">
</p>


---
## Overview

This book provided foundational knowledge into historical and modern web technologies, browser security architecture, client-side trust boundaries, session management, parsing behaviour, and common web application attack surfaces. 

### My Review

Michal Zalewski has an impressive writing style, and despite being published years ago, much of the material still holds strong relevance within modern web security. 
It's worth noting, many of the security concerns, theories, and predictions discussed throughout the book have proven remarkably accurate over time as modern browsers, web applications, and attack techniques evolved.
Zalewski should be commended for his ability to highlight less immediately obvious considerations in web security design, such as character encodings and alternate alphabets, Unicode edge cases, MIME type confusion, and other browser parsing and behavioural inconsistencies.

The material helped build a stronger understanding of how browsers process and trust content, how web applications communicate with users and servers, and how vulnerabilities can emerge through insecure implementation or misunderstood browser behaviour.


---
## Key Concepts Applied

- Browser security models
- HTML, JavaScript, and CSS
- Parsing and browser behaviour
- Same-Origin Policy (SOP)
- Content Security Policy (CSP)
- Cross-Site Scripting (XSS)
- Cross-Site Request Forgery (CSRF)
- Session management
- Cookie security
- JWT handling and browser storage
- HTTP security concepts
- Client-side trust boundaries
- Character encoding and Unicode edge cases
- Browser parsing inconsistencies
- MIME type confusion and content sniffing
- Authentication and session trust models
- URL parsing and canonicalisation
- Browser sandboxing concepts
- Security implications of browser caching and history


---
## Related Projects

- [OWASP Juice Shop Security Assessment](../../projects/break/juice-shop/README.md)


---
## Practical Application

Knowledge gained throughout this book was directly applied during web application security testing activities conducted against OWASP Juice Shop using OWASP ZAP. 

The book also helped contextualise how browser behaviour, client-side trust, and insecure application design contribute to modern web application vulnerabilities.


---



