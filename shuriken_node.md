Open ports 22,8080

Port 8080:
  Framework: Express.js
  
  Exploit: NodeJS Deserialization
  
  NodeJS shell generator: https://github.com/ajinabraham/Node.Js-Security-Course/blob/master/nodejsshell.py
  
  How to use the exploit:
  `{"rce": "_$$ND_FUNC$$_function () {[Copy the eval from the shell generator]} ()"}`
  
  ***Use this in the session cookie after base64 encoding the whole thing***.
  
  Now the privesc is just **CVE-2021-3156**, from which I used **exploit_nss.py**
  
  Get this exploit from the *linpeas.sh* scan.
