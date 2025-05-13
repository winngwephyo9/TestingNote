When your organization uses Zscaler (or a similar proxy), it often intercepts and re-encrypts HTTPS traffic. Your browser, by default, trusts a set of well-known Certificate Authorities (CAs). To access sites securely through Zscaler, your browser also needs to trust Zscaler's CA.  Most browsers already trust it or can be easily configured to trust it. Therefore, we can export the Zscaler CA from the browser.


Step-by-Step Browser Instructions

I'll provide detailed instructions for Chrome, Firefox, and Edge, as these are the most common browsers. The general principles are similar across them
Microsoft Edge

Open the Website: Open Edge and navigate to an HTTPS website.
View Certificate Details:
Click the padlock icon in the address bar.
Click "Connection is secure".
Click "Certificate is valid".
Certificate Viewer:
The Certificate Viewer appears.
Go to the "Certification Path" tab.
Select the Zscaler Certificate:
Locate and select the certificate issued by Zscaler.
Export the Certificate:
Click "Details" tab.
Click "Export File...".
The Certificate Export Wizard will open. Click "Next".
Choose "Base-64 encoded X.509 (.CER)" or "DER encoded binary X.509 (.CER)". Base-64 is usually safer. Click "Next".
Enter a filename and path to save the certificate. Click "Next" and then "Finish".

<img width="666" alt="image" src="https://github.com/user-attachments/assets/0b1076b2-3114-49d2-8941-ccb1b55360cb" />
