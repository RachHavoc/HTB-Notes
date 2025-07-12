 [_Security Headers_](https://securityheaders.com/), will analyze HTTP response headers and provide basic analysis of the target site's security posture. We can use this to get an idea of an organization's coding and security practices based on the results.

![Figure 15: Scan results for www.megacorpone.com](https://static.offsec.com/offsec-courses/PEN-200/imgs/infopass/979122077f753bc023680bcaca0c6ed8-infopass_scans_01new.png)

Missing some security headers. 

_SSL Server Test_ from [Qualys SSL Labs](https://www.ssllabs.com/ssltest/)

This tool analyzes a server's SSL/TLS configuration and compares it against current best practices. It will also identify some SSL/TLS related vulnerabilities, such as [Poodle](https://www.cisa.gov/news-events/alerts/2014/10/17/ssl-30-protocol-vulnerability-and-poodle-attack) or [Heartbleed](https://heartbleed.com/)

![Figure 16: SSL Server Test results for www.megacorpone.com](https://static.offsec.com/offsec-courses/PEN-200/imgs/infopass/e4846f8d418a351e2f0880dae176308d-infopass_scans_02new.png)

Disabling the _TLS_DHE_RSA_WITH_AES_256_CBC_SHA_ suite has been [recommended for several years](https://msrc-blog.microsoft.com/2013/11/12/security-advisory-2868725-recommendation-to-disable-rc4/), for example, due to multiple vulnerabilities both on AES Cipher Block Chaining mode and the SHA1 algorithm