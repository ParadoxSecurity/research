# HTTP Request Headers: Small Inputs with Big Security Impact

## Introduction

<img width="1585" height="733" alt="git1" src="https://github.com/user-attachments/assets/7a25282f-5f23-43e7-b4c2-799518844e54" />

In the above report [Clario | Report #737323](https://hackerone.com/reports/737323) , the researcher used the `X-Rewrite-URL` header to bypass a front-end Nginx restriction that returned **403 Forbidden** when accessing the `/admin/login` path. Instead of requesting the protected endpoint directly, the request was sent to an allowed path while the real destination was placed inside the `X-Rewrite-URL` header. The front-end server ignored this header, but the back-end application processed it and routed the request to the restricted endpoint.

This behavior also applies to `X-Original-URL`, which is supported by several frameworks for URL rewriting. When security controls are enforced only at the reverse proxy or web server, but the back-end trusts these headers, an attacker may be able to bypass access controls. This type of issue exists because different components in the request chain interpret HTTP headers differently.

## Why HTTP Headers Matter

HTTP headers are designed to carry additional information about a request. While many are standardized, applications and frameworks often introduce custom headers for load balancing, reverse proxies, URL rewriting, authentication, or client identification.

Problems arise when one component trusts a header while another ignores it. A reverse proxy, WAF, CDN, and application server may each parse the same request differently. Attackers actively search for these inconsistencies because they can lead to authorization bypasses, cache poisoning, account takeover, and other high-impact vulnerabilities. Recent research has shown that parsing differences between intermediaries remain a common source of security issues.

## Common Headers That Can Introduce Vulnerabilities

### 1. X-Original-URL / X-Rewrite-URL

These headers are primarily used for URL rewriting. If the front-end validates only the request path while the back-end uses one of these headers to determine the actual destination, attackers may bypass access control checks and reach restricted resources such as administrative panels or internal APIs. This is a well-known authorization bypass technique documented by OWASP.

### 2. Host and X-Forwarded-Host

Applications frequently use the `Host` or `X-Forwarded-Host` header when generating absolute URLs. If these values are not validated, an attacker may supply a malicious domain.

One of the most common consequences is **password reset link poisoning**. During password recovery, the application generates a reset URL using the attacker-controlled host value. The victim receives an email containing a valid reset token pointing to the attacker's domain, allowing the attacker to capture the token and potentially take over the account. The same issue can also lead to web cache poisoning, open redirects, or access to unintended virtual hosts.

### 3. X-Forwarded-For and Related IP Headers

Headers such as `X-Forwarded-For`, `X-Client-IP`, `X-Remote-IP`, and similar fields are commonly added by proxies to identify the original client IP address.

If an application blindly trusts these values, attackers can spoof their source IP. This may bypass IP-based access restrictions, evade rate limits, or gain access to functionality intended only for localhost or internal networks.

### 4. Referer

Although the `Referer` header is useful for analytics, some applications incorrectly use it as a security mechanism. Developers may rely on it to verify whether a request originated from a trusted page before allowing sensitive actions.

Since the `Referer` header can often be modified or omitted, using it as an authorization mechanism is insecure. In real-world applications, researchers have discovered cases where manipulating this header bypassed weak anti-CSRF or business logic protections.

### 5. Content-Length + Transfer-Encoding

The classic combination behind HTTP Request Smuggling. If a front-end proxy and back-end server disagree on which header to trust, attackers can "smuggle" a hidden request to bypass WAFs, poison caches, hijack user requests, or access internal endpoints.

### 6. Content-Type

Many applications validate input based on the declared content type instead of the actual body. Changing **application/json** to **application/x-www-form-urlencoded**, **multipart/form-data**, or even invalid values can bypass input validation, WAF rules, or upload restrictions.

### 7. Origin

Used for CORS validation. If an application trusts arbitrary Origin values or performs weak matching (e.g., **\*.example.com.attacker.com**), it can expose sensitive cross-origin data.

### 8. X-HTTP-Method-Override / X-Method-Override

Some frameworks honor these headers to change the HTTP method. A **POST** request may become a **PUT** or **DELETE**, occasionally bypassing proxy or firewall restrictions that only inspect the original method.

---

There are many more HTTP headers that can contribute to security vulnerabilities, but covering every possible case is not possible. I encourage readers to explore the further reading resources below for a more comprehensive understanding of header-based security issues.

Vulnerabilities are often not caused by the header itself, but by inconsistent interpretation of the same header across multiple components.

## Lessons for Security Researchers

During web application testing, HTTP headers deserve the same attention as URL parameters, cookies, and request bodies. A vulnerability is often not caused by the header itself, but by how different components interpret it.

A good testing approach is to identify whether the application is behind a reverse proxy, load balancer, CDN, or WAF, then experiment with commonly trusted headers such as `X-Original-URL`, `X-Rewrite-URL`, `Host`, `X-Forwarded-Host`, `X-Forwarded-For`, and `Referer` etc.,. If one component ignores a header while another trusts it, that difference can sometimes be turned into a security vulnerability.

## Conclusion

HTTP request headers appear simple, but they frequently influence routing, authentication, client identification, and URL generation. Many high-severity vulnerabilities originate not from complex code, but from misplaced trust in user-controlled headers. Understanding how each layer of an application processes these headers is therefore an essential skill for both security researchers and developers. Proper validation, consistent parsing across infrastructure components, and avoiding security decisions based solely on client-supplied headers significantly reduce the risk of these attacks.

## Further Reading

* [HTTP headers - HTTP | MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers)
* [HTTP Host header attacks | Web Security Academy](https://portswigger.net/web-security/host-header)
