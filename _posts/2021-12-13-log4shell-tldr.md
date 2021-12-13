---
tags: 
 - blog
 - log4shell
 - security
layout: post
title: Log4Shell What Should I Do ?
excerpt: |
 A TLDR guide to address potential Log4Shell exposure

img_url: /images/2021-12-13-log4shell.png

---
## Context

The context is quite simple, on December 9, 2021 a [CVE rated 10 on 10](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-44228) was released about the Log4j2 library which is a widely used logging library in the Java ecosystem. And as the Java ecosystem is huge, well it's a huge vulnerability that allows to take control of your server with just one request. Plenty of resources are available (look at the end of this post), and here I just want to give a quick remediation guide based on your situation.

## Step 1 : Do I Use Log4j ? And What Version ?

The first thing to know is wether you are using Log4j?

```shell 
$ find . -name "log4j-*.jar"
./lib/log4j-jcl-2.11.1.jar
./lib/log4j-slf4j-impl-2.11.1.jar
./lib/log4j-core-2.13.3.jar
./lib/log4j-api-2.13.3.jar
```

We can see here that we are using log4j version 2.13.3. A piece of advice: if the file is there, then you're affected. Don't fool yourself by thinking your using another logging system or that you're not affected for whatever applicative reason. If the file is there, you need to fix it.


## Step 2 : How Bad Is It ?

A diagram is always better than a long explanation:

![Log4ShellConcerned](/images/2021-12-13-log4shell-diagram.png)

If you're in the danger zone, then you need to escape as quickly as possible by choosing one of the possible solutions, in order:

1. Migrate to `log4j` >= 2.15
2. Launch your application with the Java option: `-Dlog4j2.formatMsgNoLookups=true`
3. Remove the `JndiLookup` class in the `log4j` jar with an empty implementation : `zip -q -d log4j-core-*.jar org/apache/logging/log4j/core/lookup/JndiLookup.class`


THERE IS NO OTHER SOLUTION! RECENT VERSIONS OF THE JVM DO NOT PROTECT YOU!

<div class="admonition warning">
<p class="admonition-title">Note</p>
<p>If your in a version prior to 2.0, then you are not exposed to this CVE. However you are exposed to other remote code execution vulnerabilities and you should plan for an upgrade.</p>
</div>

## Step 3 : What Next ?

If you are in the safe zone then congratulations! If you're still in a warning zone, then plan to migrate to the latest version of `log4j`.


## Bonus : Update My Deployments In Kubernetes

Now if you have some workload running in Kubernetes and you want to globally update the `JAVA_OPTS` environment variable, here is a small dirty script that automates the manipulation for numbers of deployments:

```bash
#!/bin/bash

for DEPLOYMENT in `kubectl get deployment | awk '/myappname/ {print $1}'`;do
    JAVA_OPTS=`oc get deployment/${DEPLOYMENT} -o=json | jq -r '.spec.template.spec.containers[0].env[] | select(.name | test("JAVA_OPTS")).value'`

    # Append only if option was not found
    echo $JAVA_OPTS | grep -q formatMsgNoLookups 
    if [ $? -eq 1 ]; then
        JAVA_OPTS="$JAVA_OPTS -Dlog4j2.formatMsgNoLookups=true"     
        kubectl set env deployment/${DEPLOYMENT} JAVA_OPTS="$JAVA_OPTS"
    fi
done
```

## References
- [CVE-2021-44228](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-44228)
- Longer explanation : https://www.lunasec.io/docs/blog/log4j-zero-day/
- JNDI injections in Java : https://www.veracode.com/blog/research/exploiting-jndi-injections-java
- Java Unmarshaller security: https://github.com/mbechler/marshalsec
- Fix the CVE by exploiting it: https://github.com/Cybereason/Logout4Shell
