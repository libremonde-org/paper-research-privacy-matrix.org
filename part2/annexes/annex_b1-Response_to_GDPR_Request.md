# Annex B.1 - Transcript of PDF file "Response to GDPR Request"

This annex is part of the [Notes on privacy and data collection of Matrix.org, Part 2](../README.md) research document.

**If you would like to link to this document, you may only do so using the header link of the Annexes section title found on the [main document](../README.md) to ensure vital context is seen by any reader.**

This transcript is provided for educational purposes only, in the public interest of individuals who want to understand and see how a GDPR Request in the scope of the Matrix protocol. Use of this document, for any other purpose, is **NOT** allowed without the express approval of the authors of the research document. The transcript is personal data of an individual performing a GDPR Request under EU GDPR law.

---

> 1. *all* personal data concerning me that you have stored, which is processed by at least all services referenced in this document: https://gist.github.com/maxidorius/5736fd09c9194b7a6dc03b6b8d7220d0;

*[Editor: Answer is given via the data files. No written reply here was expected. As expected, no reply was added to the PDF itself.]*

> 2. The lawful basis under which you are processing the data;

Legitimate Interest as per section 2.1.1 of the Privacy Notice - *Legal Basis for Processing*

> 3. If required for the processing, the data that proves my consent to the service as well as the IP address, date, time, and mean of collection of my explicit consent of the processing;

Legitimate Interest does not require explicit consent.

> 4. the purposes of the processing;

The purposes of processing are described in section 2.1.1 of the Privacy Policy - Legal Basis for Processing

> 5. the categories of personal data concerned;

Categories of personal data are catalogued in section 2.2 of the Privacy Policy - *What Information Do You Collect About Me and Why?*

> 6. the recipients to whom the personal data have been or will be disclosed;

Recipients of personal data are described in section 2.3 of the Privacy Policy - *What Information is Shared With Third Parties and Why?*

> 7. where possible, the envisaged period for which the personal data will be stored, or, if not possible, the criteria used to determine that period;

Data is stored for as long as it remains accessible to users under the access controls defined within the service.

> 8. where the personal data are not collected from the data subject, any available information as to their source;

No personal data is collected from any other source.

> 9. the existence of automated decision-making, including profiling, referred to in Article 22(1) and (4) GDPR and, at least in those cases, meaningful information about the logic involved, as well as the significance and the envisaged consequences of such processing for me.

As per section 2.2 of the Privacy Policy - *What Information Do You Collect About Me and Why?* we perform no automated decision-making or profiling of our users.

> 10. highlight of the specific data that was accessed and/or extracted during the security breach which you disclosed at the following address: https://matrix.org/blog/2019/04/11/we-have-discovered-and-addressed-a-security-breach-updated-2019-04-12

As described in the blog post, there was no evidence of large-scale exfiltration of data. Password hashes
were accessed, but since your account is not native to matrix.org no password data relating to your account
(which resides on kamax.io) was affected. It is not possible to say if the attacker used their access to view
small numbers of messages; if they did then conversations you have had in rooms which are replicated on
matrix.org could have been accessed, and read if they were unencrypted.

> If you are transferring my personal data to a third country or an international organisation, I request to be informed about the appropriate safeguards according to Article 46 GDPR concerning the transfer.

Message data is federated across all participating homeservers, wherever they are located, as described in section 2.3 of the Privacy Policy - What Information is Shared With Third Parties and Why?.

---

The file did not contain any other comment or reply.
